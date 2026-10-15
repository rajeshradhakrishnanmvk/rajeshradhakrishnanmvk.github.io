---
layout: post
tags:
  - databricks
  - lakehouse
  - architecture
  - streaming
---

# The Checkpoint Keeps the Count: What Databricks State Repartitioning Changes for an Architect

On 13 October 2026, the Databricks release notes marked on-demand state repartitioning generally available. The sentence is short. A stateful Structured Streaming query, or a Lakeflow pipeline, can change its shuffle-partition count without losing the checkpoint. The page I would take into a design review was updated on 12 October, and the text I read does not say generally available or Public Preview. The engineering post, by Thangam Vaiyapuri, Jay Palaniappan, B. Micheal Okutubo, and Zifei Feng, is 14 September, and it still says Public Preview. The Runtime 18 notes, last updated 11 September, still put Public Preview on the 10 June bullet. A Microsoft Community Hub post from 22 September is circulating with the same preview label. Those pages do not describe the same maturity, and the runtime notes do not describe the same API as the feature page.

The October product notes also say releases are staged, and an account can lag by a week or more. The runtime page says a cluster picks up a dated update when it restarts, and that an account might lag a few days. I would check the cluster. I would not close the review from the calendar.

## The count is a layout

A stateful query keeps intermediate state. A windowed count, a stream-stream join, a drop-duplicates set, a session, a `transformWithState` variable. The stateless query only remembers how far it got in the source. The state is partitioned. The 14 September post is the account of why the count froze. Each key, a user, an account, a window, is hashed to one partition, and that partition is its own RocksDB instance in the checkpoint. The number of partitions is the layout of the store.

Change the hash and the key looks in a different instance. The bytes are still in the old one. Structured Streaming refused that. It wrote the count when the checkpoint was created and ignored a later `spark.sql.shuffle.partitions`. The safe outcome was a query that would not silently lose its memory. The expensive outcome was a count you could not revisit. The historical repair was a new checkpoint location, which drops the state. For a fraud aggregation that has been accumulating for months, or a sessionization job holding days of windows, that repair is a replay, not a configuration change.

The stateful-streaming page, updated 9 October, still prints the lock as an important note. Changing `spark.sql.shuffle.partitions` has no effect once a checkpoint exists, and a new count requires a new checkpoint location. The next paragraph is the carve-out. In Databricks Runtime 18 LTS and above, the count can change without losing checkpoint state. I would read both sentences. An operator who stops at the note will throw away a checkpoint the carve-out was written to keep. An operator who ignores the note on a runtime that does not have the carve-out will restart, watch the old count keep running, and think the job accepted the new one.

The same page has a second sentence that looks like the carve-out and is not. In Databricks Runtime 18.0 and above, stateless queries can change shuffle partitions without a new checkpoint. The 15 January runtime entry says the stateless change does not require a restart. I would keep both of those sentences, and I would not apply either of them to a query that has RocksDB files. A stateless query has no layout to move.

## What the 12 October page runs

The requirements on the AWS page, the Google Cloud page, and the Azure page I read are the same two lines. Databricks Runtime 18 LTS and above. The RocksDB state store provider. On Databricks Runtime 17.3 and above, RocksDB is the default, and a query created there uses it unless someone set another provider. The 14 September post says the same runtime floor and the same provider, and it calls the feature Public Preview.

The runtime lifecycle paragraph, on the page last updated 11 September, is a third label. Runtime 18 is a transitional release, currently in the GA phase, and it becomes LTS when Runtime 19 is generally available. The 10 June entry says Runtime 18 itself is generally available. The feature page says "18 LTS." The blog says "Runtime 18 or above." I would not wait for the lifecycle page to change the word LTS, and I would not pin the cluster to 18.0. That page marks 18.0 and 18.1 end of support. Features that would have shipped as 18.3 or later ship as dated updates to Runtime 18. I used that naming on 12 October. It still applies. The cluster I would name in the design is Runtime 18, restarted onto a build that contains the feature.

The mechanism is a stop, a different configuration, and a start against the same checkpoint.

```python
query.stop()
spark.conf.set("spark.sql.streaming.stateStore.partitions", "400")
query = (
    txns
    .withWatermark("event_time", "10 minutes")
    .groupBy(window("event_time", "5 minutes"), "account_id")
    .count()
    .writeStream
    .format("delta")
    .option("checkpointLocation", checkpoint)
    .outputMode("append")
    .start()
)
```

For a stateful query, `spark.sql.streaming.stateStore.partitions` takes precedence over `spark.sql.shuffle.partitions`. That precedence is why this restart sticks and the old one did not. The query does not resume on the new layout immediately. It finishes the last planned microbatch, if one is still pending. Then it runs one repartition operation that redistributes the state into the new number of instances. Then it resumes. The 12 October page and the 14 September post describe that order in the same way.

Lakeflow does not get a separate feature. The October release notes name the pipeline. The docs set the same configuration with `spark_conf` on `@dp.table` or `@dp.append_flow`. The 14 September post calls that example Spark Declarative Pipelines. The 22 September Community Hub post says a Lakeflow change needs no manual restart outside the pipeline's normal deploy cycle. The Databricks page I read does not say that. I would treat the next pipeline update as the moment the new count is submitted, and I would not describe the stream as resize-free.

After the next microbatch completes, `StreamingQueryProgress` carries the repartition duration. In `durationMs`, the field is `controlBatch.REPARTITION`, in milliseconds. The docs say a larger state might take longer. The blog says the authors expect a few seconds for most workloads, and that a big job should capture the metric rather than assume the expectation. I would put the metric in the runbook. I would not put "a few seconds" in an SLO.

Alexis Chicoine, senior software developer at Coveo, is quoted in the 14 September post. He says the capability cut Coveo's related Amazon S3 API costs by 40 percent, and that before it, every scaling decision was a choice between overprovisioning and a new checkpoint, with storage API costs nearly the same as compute. That is his measurement of their pipelines. The post does not show the method. I would not copy 40 percent onto another account.

## Three knobs, one checkpoint

A production stream already has two other controls that move state around. They do not change this count. I would draw them before anyone calls a restart a resize.

| Knob | What moves | What stays | When I would use it |
| --- | --- | --- | --- |
| `spark.sql.shuffle.partitions` on an existing stateful checkpoint | Nothing | The original count | Nowhere, once the checkpoint exists. The value is ignored. |
| State rebalancing | RocksDB instances onto new executors, loaded from cloud storage | The partition count | The cluster changed size. Lakeflow streaming has this on by default. |
| `spark.sql.streaming.stateStore.partitions` | Keys, rehashed into a new number of instances | The checkpoint location, and the state | The count itself is wrong, and a pause is acceptable. |

State rebalancing is the cluster knob. It is enabled by default for streaming workloads in Lakeflow pipelines. On Databricks Runtime 11.3 LTS or later, a cluster that is not a Lakeflow pipeline turns it on with `spark.sql.streaming.statefulOperator.stateRebalancing.enabled`. A resize loads state from cloud storage onto the executors that now own it, and micro-batches can run slower while that load happens. Stateless work does not benefit. The page also says compute autoscaling is limited when it tries to scale a Structured Streaming cluster down, and it points enhanced autoscaling on Spark Declarative Pipelines at that problem. I would not expect a warehouse-style autoscale policy to retune this query.

The sizing sentence on that same page is still one to two shuffle partitions per core. Rebalancing does not apply it. A cluster that grew from the day-one size, with the day-one count left in the checkpoint, is the gap between the two knobs. Repartitioning is how the count catches the cores. Rebalancing is how the existing instances catch the executors.

A rehash still puts one key in one instance. The 14 September post says some partitions run hot because the hash skewed. More instances spread keys that used to share a partition. They do not split a key. A single account that dominates the state is still one RocksDB instance after the restart. I would look for that key before I doubled the count to cure one slow task.

<img src="/assets/images/state-repartition-three-knobs.svg" alt="Three knobs on one stateful checkpoint: an ignored shuffle.partitions change, state rebalancing that keeps the count, and stateStore.partitions that rehashes RocksDB state in one control batch" title="The count is the layout. Rebalancing moves instances. Repartitioning builds a new layout in the same checkpoint." style="width:100%; height:auto;">

The blog's own examples pull in two directions, and I would keep the stricter one. It describes scaling an ad-bidding stream up for the day and down overnight, and scaling a backfill up and then down to the steady-state count. The same section says each change is a stop, a restart, and a one-time pause, and that it belongs in a window where that pause is acceptable. I would use it for a count that has been wrong for weeks. I would not put it on a cron beside the traffic curve.

## The batch that has to finish first

The repartition is a batch in the checkpoint, and the error class `STATE_REPARTITION_INVALID_CHECKPOINT` is the contract for when that batch is refused. The page was last updated on 11 September. SQLSTATE is 55019. The class name sounds like a corrupt checkpoint. Several of the subclasses are not.

`SHUFFLE_PARTITIONS_ALREADY_MATCH` means the last committed batch already has the count you asked for. The message calls it a no-op. I would not treat that condition as damage.

`LAST_BATCH_FAILED` means the previous batch did not finish. The page says to let the query finish successfully before repartitioning. If the trigger is processing time, it says you can use an AvailableNow trigger so the query terminates successfully by itself. It also names a parameter, `enforceExactlyOnceSink`, that skips the check. The same sentence says the skip can emit duplicate output from the failed batch when the sink is exactly-once. The page does not show a configuration key for that parameter. I would not go looking for one in order to force a resize through a failed batch.

`LAST_BATCH_ABANDONED_REPARTITION` means the last batch was itself a repartition, at a count you are no longer requesting, and it did not finish. The repair is to retry that same count. A different count is legal only after that attempt commits. I would write the attempted count down before the restart, because the next person will otherwise try the count they wanted rather than the count the failed batch used.

`NO_BATCH_FOUND` and `NO_COMMITTED_BATCH` mean the checkpoint has not successfully completed a microbatch. There is nothing to redistribute. `OFFSET_SEQ_NOT_FOUND` points at `spark.sql.streaming.minBatchesToRetain` set very low, or at files deleted from the checkpoint. `MISSING_OFFSET_SEQ_METADATA`, `UNSUPPORTED_OFFSET_SEQ_VERSION`, and `UNSUPPORTED_COMMIT_METADATA_VERSION` all tell you to use a checkpoint from Spark 4.0 or above, or from Databricks Runtime 14.3 or above. A stream that has been alive since an older runtime can be refused even after the cluster is on Runtime 18. I would check the checkpoint generation before I promised the resize.

`UNSUPPORTED_PROVIDER` is the provider lock. `UNSUPPORTED_STATEFUL_OPERATOR` names an operator and does not publish the list. `UNSUPPORTED_TRANSFORM_WITH_STATE_VARIABLE_TYPE` names a variable type, whether TTL is enabled, and a column family. The 14 September post says the feature applies to aggregations, stream-stream joins, deduplication, sessionization, and `transformWithState`. I would treat that as the authors' scope. I would not treat it as a closed allow-list, and I would not treat it as a guarantee while those two subclasses exist. The 9 October page, in a different section, says chained stateful operators support only append mode, and that `flatMapGroupsWithState` and `applyInPandasWithState` are not part of that chaining support. I would not copy that limitation onto the repartition error. The error page does not.

## The store under the count

RocksDB is the provider the feature will repartition. The state-store page, updated 11 September, says it is the default on Databricks Runtime 17.3 and above. Below that, the page sets `spark.sql.streaming.stateStore.providerClass` to `com.databricks.sql.streaming.state.RocksDBStateStoreProvider`. The `transformWithState` page sets the same configuration to `org.apache.spark.sql.execution.streaming.state.RocksDBStateStoreProvider`. I would not paste either string over a running checkpoint. The stateful-streaming page says the state management scheme cannot change across a restart. A query that started on the other provider needs a new checkpoint to move, and a new checkpoint is the path this feature was built to avoid. Changelog checkpointing is the setting that can change in place. The RocksDB page says you can enable it on an existing stream and keep the state. It is the recommendation for stateful queries, and it is on by default from Runtime 17.3. Enabling the changelog is not a repartition. I would not confuse the two restarts.

More partitions are more RocksDB instances. From Runtime 17.3, Databricks caps RocksDB memory per node, across the instances on that node. A higher count does not raise that cap. On earlier runtimes the cap is manual, through `boundedMemoryUsage` and `maxMemoryUsageMB`, or the instances grow until the node runs out. I would read the node cap before I multiplied the count on a cluster that is already close to it.

The metrics I would use are not one number. The RocksDB page's sample puts `numRowsTotal` on the state operator. From Runtime 16.4, the `customMetrics` entries are labeled with the partition id and the store name, so a resize changes which keys appear. The 22 September post describes `numRowsTotal` as a per-partition figure. The sample I read does not. I would use the partition-labeled metrics for skew, and the operator total for the size of the pause I am about to take.

Runtime 18 has a neighboring issue I would put on the same page, because a long-running stream is the workload that hits it. The known-issues note says Apache Avro 1.12.1 turns the fast reader on by default, the per-schema cache is not evicted, and a Structured Streaming or Auto Loader job with a large `maxFilesPerTrigger` can accumulate readers until the executor fails with `OutOfMemoryError: GC overhead limit exceeded`. Runtime 17.3 and earlier are not affected. The workaround is `-Dorg.apache.avro.fastread=false` on `spark.driver.extraJavaOptions` and `spark.executor.extraJavaOptions`, appended to whatever is already there, and a cluster restart. Moving a stream onto Runtime 18 to gain this feature is the move that inherits that cache.

## What Spark shipped, and what it did not

Databricks Runtime 18 is powered by Apache Spark 4.1.0. That line is on the runtime page. The Spark 4.1.0 release notes list [SPARK-53942](https://issues.apache.org/jira/browse/SPARK-53942), support for changing shuffle partitions in stateless streaming workloads. They do not list a stateful equivalent. The stateless knob on the 9 October page has an upstream note. The stateful knob does not.

The upstream work I found is later, and it is not a configuration I would hand to an open-source job. [SPARK-54346](https://issues.apache.org/jira/browse/SPARK-54346) is resolved, with fix version 4.2.0. The description says the task introduces the repartition API and the runner core, and that Spark Connect and PySpark APIs come in later pull requests. The pull request that landed it, authored by B. Micheal Okutubo, says the API is not exposed because it is still in development, and it answers "no" to a user-facing change. Later pull requests I read, for column-family readers and writers and for `transformWithState` tests, give the same answer. The Spark 4.2.0 release notes page I read does not list SPARK-54346. Okutubo is also an author of the 14 September Databricks post. I would not collapse the pull request and the product into one surface. The contract I would operate is the Databricks configuration. The Spark 4.2 runner is an offline implementation the introducing pull request told me not to call.

The runtime notes point at a third surface, and the feature page does not contain it. The 15 January entry says a state repartition API for `transformWithState` is available in Scala, so a restart can change the key partitioning. The 2 April entry says that API is also in PySpark. Both bullets link to the on-demand state repartitioning page. The page I read on 15 October shows the configuration and the Lakeflow decorator. It does not show a method. I would not invent the call.

## Where it fits

I would use it when the query is already stateful, the checkpoint is already RocksDB, at least one microbatch has committed, and the count is the thing that is wrong. A fraud stream that launched in one region on the default of 200 and now covers every market. A backfill that needed a wide count and is now a steady stream. A cluster whose core count moved, after rebalancing has already put the old instances on the new executors and the tasks are still too big, or still too numerous. The pause fits in a maintenance window. The metric is on the dashboard before the restart. The attempted count is written down, so an abandoned repartition can be retried at that count.

I would ship the first resize on a copy of the checkpoint in a non-production run, long enough to read `controlBatch.REPARTITION`, and only then on the stream whose late state I cannot replay. The blog's expectation of a few seconds is not a prediction of that copy.

## Where it does not fit

I would leave it when the provider is not RocksDB. Switching providers is a new checkpoint, which is the outcome this feature is supposed to retire. I would leave it when the checkpoint's offset or commit metadata is older than Spark 4.0 or Databricks Runtime 14.3, until someone has proved the error page's version check accepts it. I would leave it when the last batch failed, until that batch commits. I would leave the `enforceExactlyOnceSink` skip unused.

I would leave it when the symptom is a cluster resize rather than a wrong count. That is rebalancing. I would leave it when one key is the hot partition. A new count does not split that key. I would leave it off a schedule that follows the hour. The pause is the feature. I would leave a stateless stream on the stateless knob, and I would leave an open-source Spark 4.1 or 4.2 job on whatever that release actually exposes. The Databricks configuration is not that exposure.

I would leave the September preview label, and the 22 September post, out of the status line. The 13 October release notes are the status. I would also leave the June runtime bullet's Public Preview out of a claim that the feature is still a preview, and I would leave the 13 October note out of a claim that every cluster has it tonight. The account lag and the cluster restart are both still in the notes.

## What goes in the design doc

**Name the knob.** An ignored `shuffle.partitions`, a rebalance, or `stateStore.partitions`. The stateless dynamic shuffle is a fourth sentence, and it does not belong on this checkpoint.

**Pin the runtime from the cluster, not from the word LTS.** Runtime 18, restarted, on a build that contains the feature. Not 18.0. The 13 October note says generally available, and it says the account can lag a week or more. The 11 September runtime page still tags the 10 June bullet as Public Preview.

**Prove the provider and the checkpoint generation.** RocksDB already, not a provider change in the same restart. A committed microbatch. Offset and commit metadata new enough for the 11 September error page. `minBatchesToRetain` high enough that the offset sequence is still there.

**Write the restart order.** Stop. Set `spark.sql.streaming.stateStore.partitions`. Start the same checkpoint location. The last planned microbatch finishes. Then one repartition batch. Then ordinary batches. On Lakeflow, the same key on `spark_conf` for the table or the flow. The next update is the submission. Record the count you submitted.

**Put the failure next to the count.** Same count is a no-op under `SHUFFLE_PARTITIONS_ALREADY_MATCH`. A failed batch has to commit before a new count. An abandoned repartition has to be retried at its own count. An unsupported operator or an unsupported `transformWithState` variable fails the restart. I would not flatten the September list into an allow-list.

**Time the pause, and read the right metric.** `controlBatch.REPARTITION` in `durationMs`. Partition-labeled `customMetrics` for skew. `numRowsTotal` as the operator total the RocksDB sample shows. The node memory cap still covers every instance. The Coveo figure stays in the blog.

**Keep the Avro flag on the Runtime 18 stream.** `-Dorg.apache.avro.fastread=false` on the driver and the executors, if the job reads many Avro schemas. The known issue is on the runtime this feature requires.

State repartitioning is the Databricks architecture that admits the partition count was a layout, and then moves the layout without discarding the checkpoint. The hash changes only inside a batch the checkpoint records. Rebalancing, the stateless shuffle change, and a new checkpoint location are still the other three answers, and they answer different questions. I would bet a tired RocksDB count on the 12 October page, with the 13 October status written beside the September preview label, and with the abandoned-batch rule in the runbook. I would not bet a hot key on it, and I would not bet an open-source Spark upgrade on the Databricks configuration.

## References

- Databricks, "October 2026" release notes, on-demand state repartitioning generally available on 13 October 2026. [https://docs.databricks.com/aws/en/release-notes/product/2026/october](https://docs.databricks.com/aws/en/release-notes/product/2026/october)
- Databricks, "On-demand state repartitioning for stateful streaming queries" (AWS), last updated 12 October 2026. [https://docs.databricks.com/aws/en/structured-streaming/state-repartitioning](https://docs.databricks.com/aws/en/structured-streaming/state-repartitioning)
- Databricks, "On-demand state repartitioning for stateful streaming queries" (Google Cloud). [https://docs.databricks.com/gcp/en/structured-streaming/state-repartitioning](https://docs.databricks.com/gcp/en/structured-streaming/state-repartitioning)
- Microsoft, "On-demand state repartitioning for stateful streaming queries" (Azure Databricks). [https://learn.microsoft.com/en-us/azure/databricks/structured-streaming/state-repartitioning](https://learn.microsoft.com/en-us/azure/databricks/structured-streaming/state-repartitioning)
- Thangam Vaiyapuri, Jay Palaniappan, B. Micheal Okutubo, and Zifei Feng, "Announcing On-Demand State Repartitioning for Apache Spark Structured Streaming on Databricks," Databricks Blog, 14 September 2026. [https://www.databricks.com/blog/announcing-demand-state-repartitioning-apache-sparktm-structured-streaming-databricks](https://www.databricks.com/blog/announcing-demand-state-repartitioning-apache-sparktm-structured-streaming-databricks)
- Databricks, "What is stateful streaming?", last updated 9 October 2026. [https://docs.databricks.com/aws/en/structured-streaming/stateful-streaming](https://docs.databricks.com/aws/en/structured-streaming/stateful-streaming)
- Databricks, "Configure RocksDB state store on Databricks," last updated 11 September 2026. [https://docs.databricks.com/aws/en/structured-streaming/rocksdb-state-store](https://docs.databricks.com/aws/en/structured-streaming/rocksdb-state-store)
- Databricks, "`STATE_REPARTITION_INVALID_CHECKPOINT` error condition," last updated 11 September 2026. [https://docs.databricks.com/aws/en/error-messages/state-repartition-invalid-checkpoint-error-class](https://docs.databricks.com/aws/en/error-messages/state-repartition-invalid-checkpoint-error-class)
- Databricks, "Databricks Runtime 18 LTS," last updated 11 September 2026. [https://docs.databricks.com/aws/en/release-notes/runtime/18](https://docs.databricks.com/aws/en/release-notes/runtime/18)
- Databricks, "Build a custom stateful application with `transformWithState`." [https://docs.databricks.com/aws/en/stateful-applications/](https://docs.databricks.com/aws/en/stateful-applications/)
- Wiliam_Rosa, "The Partition Count You Picked on Day 1 Doesn't Have to Follow You Forever," Microsoft Community Hub, 22 September 2026. [https://techcommunity.microsoft.com/discussions/azure/the-partition-count-you-picked-on-day-1-doesnt-have-to-follow-you-forever/4559106](https://techcommunity.microsoft.com/discussions/azure/the-partition-count-you-picked-on-day-1-doesnt-have-to-follow-you-forever/4559106)
- Apache Spark, "Spark Release 4.1.0," including SPARK-53942. [https://spark.apache.org/releases/spark-release-4.1.0.html](https://spark.apache.org/releases/spark-release-4.1.0.html)
- Apache Spark, "Spark Release 4.2.0." [https://spark.apache.org/releases/spark-release-4-2-0.html](https://spark.apache.org/releases/spark-release-4-2-0.html)
- Apache Spark, Jira SPARK-54346, "Introduce repartition API and repartition runner," fix version 4.2.0. [https://issues.apache.org/jira/browse/SPARK-54346](https://issues.apache.org/jira/browse/SPARK-54346)
- B. Micheal Okutubo, "[SPARK-54346][SS] Introduce state repartition API and repartition runner," pull request 53056. [https://github.com/apache/spark/pull/53056](https://github.com/apache/spark/pull/53056)
