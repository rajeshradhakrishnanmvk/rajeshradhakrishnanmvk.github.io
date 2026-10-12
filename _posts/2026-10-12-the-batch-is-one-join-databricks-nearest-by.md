---
layout: post
tags:
  - databricks
  - lakehouse
  - architecture
  - sql
---

# The Batch Is One Join: What Databricks NEAREST BY Changes for an Architect

On 5 October 2026, Databricks published the engineering account of `NEAREST BY`. The SQL clause is older than that post. It appears in the Databricks SQL 2026.20 Preview notes dated 29 June 2026, and the Apache Spark 4.2.0 release notes list [SPARK-56395](https://issues.apache.org/jira/browse/SPARK-56395). The October post is the document I would take into a design review. It is the first write-up I found that puts a Photon fused operator and a Delta IVF index on the same page as the join. The language reference, updated 11 September, is the contract for the SQL. Those documents do not describe the same system.

## A batch is not a million lookups

Vector search grew up as a serving problem. One embedding arrives, and the system is built to return a handful of neighbors in tens of milliseconds. The October post says a good share of the vector work on Databricks is the other shape. A payments company matches 100 million or more daily transactions against 140 million merchant embeddings. A data firm enriches tens of millions of historical records each night. A quantitative fund runs million-query batches against a 50 million vector corpus. Entity resolution, deduplication, semantic tagging, classification, record enrichment, batch recommendations. The success unit is whether the job finishes inside its window, at a cost someone will sign.

Their first SQL path for this work did not run the search. It dispatched it. The post describes the `VECTOR_SEARCH` function as a Generate node that streamed one query row at a time. Every row paid for a network request to a real-time Vector Search endpoint, a response to deserialize, and retries when the endpoint asked for them. Throughput stopped at the size of that endpoint. The cluster running the job was a client.

The function page I read, updated 7 October 2026, still documents `vector_search` as a query against an AI Search index. The create page, updated 9 October, says Databricks AI Search was formerly known as Databricks Vector Search. That is the endpoint family the October post describes the earlier function calling. The feature is in Public Preview. It is unavailable on classic SQL warehouses. AI Search with `num_results` greater than 100 is unsupported. A batch on that page is a `LATERAL` subquery, one index call per driving row. That page and the October engineering post share a verb. I would draw both, and I would refuse a design that says "vector search" without naming which one.

A batch nearest-neighbor job is a relational operator. For each row on the left, keep the k best rows on the right under a scoring expression. That is a top-k ranking join. `NEAREST BY` is that operator, written as SQL, executed by the engine that already holds the tables.

## Two words that are the design

The language page, which applies the clause to Databricks SQL and Databricks Runtime 18 LTS and above, gives the shape:

```text
{ INNER | LEFT [ OUTER ] } JOIN target_table_reference
  { APPROX | EXACT } NEAREST [ num_results ]
  BY { DISTANCE | SIMILARITY } ranking_expression
```

The same page says Databricks Runtime 18 is newer than Databricks Runtime 18.0, 18.1, and 18.2, because features that would have shipped as a later numbered release now ship as dated updates to Runtime 18. I would read every "18.x and above" pin against that note before treating 18.1 as a higher bar than 18 LTS.

Four choices on that page belong in the design doc.

The left side drives. The right side is searched. The clause is not commutative. One hundred rows from `users` joined to one thousand rows from `products` with `NEAREST 5` returns up to 500 rows. Swap the sides and the same keyword returns up to 5,000 rows, because each product is now a query. Swapping the tables asks a different question. I would write the driving set on the left of the diagram, in the query, where a later editor cannot miss it.

`EXACT` returns the exact top-k under the ranking expression. `APPROX` returns a top-k that approximates that ranking, and the optimizer may use a faster strategy instead of evaluating every candidate. The October post states the consequence I would keep: creating or dropping an index cannot silently change results, because only a query that says `APPROX` has consented to approximation. The Spark proposal makes the same choice, and it calls out systems where the mere presence of an index changes behavior.

`BY DISTANCE` ranks the smallest value first. `BY SIMILARITY` ranks the largest value first. The expression is any orderable scalar that can see both sides. The documented examples are `vector_cosine_similarity`, `vector_l2_distance`, `vector_inner_product`, and a Manhattan distance built as `vector_norm(zip_with(a.col, b.col, (x, y) -> x - y), 1.0f)`. A composite expression is in bounds. A type that cannot be ordered, such as `MAP`, raises `DATATYPE_MISMATCH.INVALID_ORDERING_TYPE`.

`INNER` is the default, and it drops a query row that has no candidate. `LEFT OUTER` returns every query row. Target columns are `NULL` when no candidate exists, including when the target is empty or every candidate score is `NULL`. The language page's example is the one I would paste above the query. A zero-magnitude embedding makes `vector_cosine_similarity` return `NULL` for every comparison, and `LEFT OUTER` keeps that row with `NULL` products. The cosine function page, updated the same day as the clause page, says the function also returns `NULL` for an empty vector, a `NULL` input, or an input that contains `NULL`. Only `ARRAY<FLOAT>` is accepted. The arrays must share a dimension, or the function raises `VECTOR_DIMENSION_MISMATCH`. A pipeline that writes `ARRAY<DOUBLE>` does not get a quiet cast.

`num_results` is optional, defaults to 1, and must be an integer literal from 1 to 100,000. Values outside that range raise `NEAREST_BY_JOIN.NUM_RESULTS_OUT_OF_RANGE`. `RIGHT`, `FULL`, `SEMI`, `ANTI`, `CROSS`, and `NATURAL` raise `NEAREST_BY_JOIN.UNSUPPORTED_JOIN_TYPE`. Streaming DataFrames and Datasets raise `NEAREST_BY_JOIN.STREAMING_NOT_SUPPORTED`.

The documented way to restrict the target is a subquery. Filter the merchants, or the articles, before the nearest-neighbor clause sees them. The Spark proposal lists a cross-table predicate, such as a price on the target compared with a budget on the query row, as planned future work, outside that proposal. I would wait for a language page that shows the predicate.

```sql
SELECT q.txn_id, t.merchant_id, t.merchant_name
FROM txns_today AS q
INNER JOIN (
  SELECT merchant_id, merchant_name, embedding
  FROM merchants
  WHERE country = 'IN' AND status = 'active'
) AS t
  APPROX NEAREST 5 BY SIMILARITY vector_cosine_similarity(q.embedding, t.embedding);
```

The same shape with `EXACT` and `BY DISTANCE vector_l2_distance(q.embedding, t.embedding)` is the query I would run when the target is a labeled reference set small enough to score in full, or when a missed true neighbor is a defect in the decision that follows. Embeddings are produced elsewhere. The language page points at `ai_query` with a `system.ai` model service, and it gives `system.ai.gte-large-en` as the example. The join knows the values are `ARRAY<FLOAT>` of one width. A Malayalam ticket corpus and an English article corpus are the same join when the model that embedded both sides covers both scripts. That choice sits in the embedding job.

## What the optimizer is allowed to do

The October post describes the rewrite. Tag each query row with a generated id, score pairs, keep the k best per id with a grouped top-k, and project the survivors. Semantically that is a cross join, a scalar expression, and an aggregate. Those operators already distribute, spill, and retry, so a job that runs for hours inherits the runtime's fault tolerance. The client-side retry loop around an endpoint is the piece this path is trying to retire.

Spark 4.2 stops at that rewrite. [SPARK-56395](https://issues.apache.org/jira/browse/SPARK-56395) says the open-source scope is the syntax and a brute-force path, lowered onto existing operators, including `max_by` and `min_by` with a k argument. The same Databricks SQL 2026.10 Preview notes that introduced the vector functions, dated 26 March 2026, gave `max_by` and `min_by` that third argument, capped at 100,000. The 2026.10 Current rollout is dated 23 April 2026. The proposal is explicit about what it leaves out: vector index DDL and indexed approximate search, a new fused physical operator, cross-table filter predicates, and threshold retrieval that returns every row inside a distance rather than a fixed count. The Spark 4.2.0 release notes list the join as catalyst support and a DataFrame API. The 16 July Databricks introduction of Spark 4.2 describes vector functions and `NEAREST BY` as SQL primitives for retrieval, recommendations, entity resolution, and candidate generation. The headline says that release is available in Databricks Runtime 19. The closing line says Apache Spark 4.2 will also be available there. Neither sentence describes a Photon kernel or an IVF index.

The 5 October post is the Databricks Runtime path beyond that rewrite. Photon runs the vector functions as native SIMD kernels. Each kernel is compiled into several ISA-specific clones, and the runtime picks the one the CPU supports, AVX-512 on a modern Intel core and SVE2 on Graviton. The fast path is not one binary and not one cloud. The kernels drop strict IEEE ordering so the compiler can emit fused multiply-adds and split the reduction. I would not write a test that demands bitwise equality with a strict scalar cosine. The implementation said it relaxed the ordering, so the test needs a tolerance.

The roofline argument is the part I would keep, and the part I would keep out of a capacity spreadsheet. The FLOPs are the same either way: `2 × d × n_q × n_b`. A pairwise plan loads every operand fresh and uses it once, moves `2 × 4 × d × n_q × n_b` bytes, and sits at an arithmetic intensity of 1/4. The post places that plan on the memory roof at 0.8 percent of peak. A fused operator buffers one side, streams the other, and scores tiles with a blocked matrix-multiply kernel. Bytes moved for the streamed side become `4 × d × n_b`, and intensity grows with the side that stays resident, `n_q / 2` in their formula. On the illustrative m6i.2xlarge they use, that crosses the ridge at 64 query rows. With top-k fused, the operator emits at most `n_q × k` candidates rather than every pair. Peak memory per task is the buffered side, one in-flight batch, and selection state on the order of `n_q × k`. When k is large enough that this state becomes memory pressure, the operator runs the matrix multiply alone and hands scored tiles to `max_by` or `min_by`. Intensity on that fallback is `d / 2`. A cluster of 64 m6i.2xlarge executors, four physical cores each, is given as peaking at about 52.5 TFLOP/s of fp32 fused multiply-add, from `64 × 4 × 204.8` GFLOP/s. The post marks the machine constants as illustrative. I would mark them the same way.

Distribution follows ordinary joins. Broadcast the smaller side when it fits. Otherwise partition both sides and run a block cartesian product. The index changes that picture, and only for `APPROX`.

## The index is a Delta table, and the runbook is thin

Exhaustive search remains proportional to `n_q × n_b`. The post's own line: a million queries against a billion rows is 10^15 dot products, and no register tile removes an exponent. `APPROX`, plus an index, is how they step off it.

The index is an inverted file. Indexing trains k-means over a sample of the corpus. Each base row is assigned to its nearest centroid. At query time each query scores the vectors in its `nprobe` nearest clusters. They chose this over a graph index because independent cluster scans parallelize across executors and the layout is columnar, while graph traversal is a chain of serial lookups. At billions of base rows, the post says a query probes 0.1 percent of the corpus or less. That is a design point in the post. It is not a recall number I would copy into an SLA.

Physically, the index is an ordinary Delta table. Assignment rows carry the vector next to a centroid id, and the table is liquid-clustered by that id. A cluster's candidates sit together on object storage. Files whose clusters no query probed are pruned before they are read. Refresh is transactional and incremental. An `APPROX` query probes the centroids with the same top-k join, equi-joins on centroid id, scores inside the probed clusters, and merges the partial top-k lists. Files added since the last refresh are brute-forced in a compensation branch and unioned into that merge. The sentence I would quote, and then narrow, is theirs: "a stale index prunes less, but never degrades search quality." The mechanism in the sentence before it is files added since the refresh. I did not find a statement that an update or a delete inside an already indexed file is compensated the same way.

The large-against-large strategy is the one I would draw. Probed queries are shuffled by centroid id. The index side is not fully shuffled. Liquid clustering lets each partition scan the files for the centroids it owns. Inside a partition the candidates arrive as a contiguous block, and the same kernel runs once per cluster. The exact path runs it once for the target.

The SQL reference documents `APPROX` as permission for a faster strategy. It does not document index DDL, a refresh command, or a setting named `nprobe`. The AI Search create page documents a different index. You build it in Catalog Explorer, with the Python SDK, or through `POST /api/2.0/vector-search/indexes`. The page names two types, a Delta Sync Index that tracks a source table, and a Direct Vector Access Index that the caller maintains. Either one is served from an endpoint. That is the index `vector_search` queries. It is not the liquid-clustered Delta table in the October post. I would not invent a `CREATE VECTOR INDEX` to close that gap, and I would not accept a runbook that does. Until the build and refresh steps sit next to the clause, an `APPROX` plan gets an `EXPLAIN` before anyone promises pruning.

<img src="/assets/images/nearest-by-two-contracts.svg" alt="NEAREST BY splits into an EXACT exhaustive score and an APPROX path over a liquid-clustered Delta IVF index, with vector_search against an AI Search index drawn as a separate serving call" title="EXACT scores the target. APPROX may probe a Delta index and brute-force files added since refresh. vector_search is a different product." style="width:100%; height:auto;">

The October post also gives the join shape for the indexed path, which is worth copying beside the picture.

| Shape | Strategy the post describes |
| --- | --- |
| Small index, small query | Broadcast nested loop. Both sides in memory. |
| Small index, large query | Broadcast the index. Queries stay partitioned and stream. |
| Large index, small query | Broadcast the probed queries. Index partitions stream. |
| Large index, large query | Shuffle probed queries by centroid id. Scan the index files in place. |

## What the numbers are

The October post reports an evaluation on workloads they describe as canonical, with base tables from 100,000 to 5 billion vectors and query batches of up to 10 million, using indexed approximate search aimed at 96 percent recall@K or better. A self-join of a 10-million-record table, for semantic deduplication, completed in minutes. Ten million queries against 100,000 reference vectors, for classification and tagging, took less than a minute. One million queries against 10 million catalog vectors, for recommendation refresh, completed in minutes. One million queries against 1 billion reference vectors, for entity resolution and enrichment, completed in minutes.

[Data Phoenix](https://dataphoenix.info/news/databricks-nearest-by-join-vector-search), the same day, wrote that those efficiency claims have not been independently benchmarked. The write-up omits the cluster shape, the embedding dimension, the value of k, the datasets, the precise elapsed times, and the cost. "In minutes" is not a capacity number. I would read the table as evidence that Databricks has run the indexed path at those cardinalities under a recall target they chose. I would retime the job on the warehouse the design intends to pay for.

## Where it fits

I would use `NEAREST BY` when both sides already live in tables the runtime can scan, the work is a scheduled batch, and the output is rows a downstream job can join. Candidate generation for entity resolution. A nightly deduplication before a merge. Tagging a corpus against a labeled reference set. Refreshing recommendations into a table another service reads later. `EXACT` when the target is that reference set and a missed true neighbor is a defect. `APPROX` when the base side is large enough that 10^15 dot products is the plan being refused, and only after the index path is visible in the plan.

I would keep k in the range where the fused top-k path still holds, because that is the path that avoids writing the full score matrix. The October post says the fallback, handing tiles to `max_by` or `min_by`, is what runs when k is large enough to pressure memory. I would pre-filter in a subquery whenever a relational predicate can shrink the right side. I would put the driving cardinality on the left, and I would write the asymmetry numbers next to the query. The language page's 100-by-1,000 example is enough when the real cardinalities are not known yet.

## Where it does not fit

I would leave this clause when the requirement is one lookup in tens of milliseconds for an interactive client or an agent tool call. That is the serving shape the October post starts from, and it is the shape `vector_search` still implements. The limits on the 7 October page travel with it: Public Preview, classic SQL warehouses excluded, `num_results` above 100 unsupported, `DIRECT_ACCESS` indexes unsupported, and no `filters_json` or `columns` arguments. The function also cannot be used with model serving endpoints that use Foundation Model APIs provisioned throughput. A `LATERAL` over `vector_search` is still one index call per row. `NEAREST BY` does not retire that endpoint.

I would leave it out of a streaming query. The error code is already named. I would leave `APPROX` out of a decision that requires the true top-k. I would leave `EXACT` out of a design whose base table is approaching the billion-row example until the team has a measured runtime and a bill. I would leave the clause when the score depends on a predicate across the two sides that a subquery cannot express. A graph index, a hybrid keyword-plus-vector query, and a threshold-by-distance query are different products or future work. Hybrid search is a `query_type` on `vector_search`. Threshold retrieval is listed as future work in the Spark proposal. The IVF choice is an argument about parallelism in the October post.

I would also keep Apache Spark 4.2 and the Databricks Photon path as two lines. Spark 4.2.0 has the join. The kernels and the Delta IVF layout are described for the Databricks Runtime on 5 October. A job that must stay on the open-source engine gets the rewrite onto ordinary operators. It does not get the fused kernel, and it does not get the index the proposal put out of scope.

## What goes in the design doc

**Name the product.** `NEAREST BY` over tables in the runtime, or `vector_search` against an AI Search index. The October post and the 7 October function page are both current. They answer different latency questions.

**Write `EXACT` or `APPROX` in the query, and say why.** An index is allowed to affect only the queries that opted in.

**Pin the runtime, and read the numbering note.** The clause page says Databricks SQL and Databricks Runtime 18 LTS and above, and it warns that Runtime 18 is newer than 18.0, 18.1, and 18.2. The cosine function page says Databricks Runtime 18.1 and above. The Spark 4.2 introduction points at Databricks Runtime 19. The warehouse channel page, updated 9 October 2026, lists Current and Preview both at Databricks SQL 2026.39, and it says a release is staged, so an account can lag by a week or more. The clause entered the notes as a Preview-channel feature of SQL 2026.20 on 29 June 2026. The clause page itself carries no Preview badge and no GA badge. Data Phoenix flagged that gap on 5 October. I would check the warehouse channel the week the job is built.

**Record the array contract.** `ARRAY<FLOAT>`, equal dimensions, `NULL` cosine on a zero vector, `LEFT OUTER` when every driving row must survive that `NULL`.

**Put the index runbook next to `APPROX`, or do not promise pruning.** The physical layout in the October post is a liquid-clustered Delta table keyed by centroid id, with a compensation branch for files added since refresh. The SQL reference does not yet say how an operator builds it. `EXPLAIN` is the check. Updates and deletes need their own sentence, because the compensation sentence covers files added.

**Retime the job.** The published runs are Databricks measurements, expressed in minutes, at a recall target of at least 96 percent recall@K, without the cluster, the dimension, or the cost. The roofline constants are labeled illustrative. They explain why a naive pair score stays memory-bound. They are not the SLO.

**Keep serving off this join.** Interactive lookup stays on the AI Search index and its endpoint. The batch job writes a table. A service reads that table, or it calls `vector_search`. It does not scan a billion embeddings on the request path.

`NEAREST BY` is the Databricks architecture that treats a batch nearest-neighbor search as a join the engine can see. The storage consequence is the one I would adopt: the embeddings stay in the table the platform already governs, and a sync into a second vector store is no longer a requirement of the batch. The serving system remains. The approximate index is a Delta layout the operator has to be able to build, refresh, and explain. I would bet a nightly resolution job on the clause. I would keep a user-facing lookup on the index endpoint, and I would wait to promise recall on `APPROX` until the plan shows the index the October post describes.

## References

- Zero Qu, Alexis Schlomer, Akash Nayar, Yingyi Bu, and Sergei Tsarev, "NEAREST BY Join: Scaling Vector Search in Databricks Runtime," Databricks Blog, 5 October 2026. [https://www.databricks.com/blog/nearest-join-scaling-vector-search-databricks-runtime](https://www.databricks.com/blog/nearest-join-scaling-vector-search-databricks-runtime)
- Databricks, "NEAREST BY clause," Databricks on AWS, last updated 11 September 2026. [https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-qry-select-nearest-by](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-qry-select-nearest-by)
- Databricks, "vector_cosine_similarity function," Databricks on AWS, last updated 11 September 2026. [https://docs.databricks.com/aws/en/sql/language-manual/functions/vector_cosine_similarity](https://docs.databricks.com/aws/en/sql/language-manual/functions/vector_cosine_similarity)
- Databricks, "vector_search function," Databricks on AWS, last updated 7 October 2026. [https://docs.databricks.com/aws/en/sql/language-manual/functions/vector_search](https://docs.databricks.com/aws/en/sql/language-manual/functions/vector_search)
- Databricks, "Create AI Search endpoints and indexes," Databricks on AWS, last updated 9 October 2026. [https://docs.databricks.com/aws/en/ai-search/create-ai-search](https://docs.databricks.com/aws/en/ai-search/create-ai-search)
- Databricks, "Databricks SQL release notes 2026," last updated 6 October 2026. [https://docs.databricks.com/aws/en/sql/release-notes/2026](https://docs.databricks.com/aws/en/sql/release-notes/2026)
- Databricks, "Databricks SQL release notes," last updated 9 October 2026. [https://docs.databricks.com/aws/en/sql/release-notes/](https://docs.databricks.com/aws/en/sql/release-notes/)
- Apache Spark, "Spark Release 4.2.0." [https://spark.apache.org/releases/spark-release-4-2-0.html](https://spark.apache.org/releases/spark-release-4-2-0.html)
- Zhidong Qu, "SPARK-56395: SPIP: NEAREST BY Top-K Ranking Join," Apache Jira. [https://issues.apache.org/jira/browse/SPARK-56395](https://issues.apache.org/jira/browse/SPARK-56395)
- Wenchen Fan and others, "Introducing Apache Spark 4.2," Databricks Blog, 16 July 2026. [https://www.databricks.com/blog/introducing-apache-spark-42](https://www.databricks.com/blog/introducing-apache-spark-42)
- Data Phoenix, "Databricks launches NEAREST BY Join for batch vector search," 5 October 2026. [https://dataphoenix.info/news/databricks-nearest-by-join-vector-search](https://dataphoenix.info/news/databricks-nearest-by-join-vector-search)
