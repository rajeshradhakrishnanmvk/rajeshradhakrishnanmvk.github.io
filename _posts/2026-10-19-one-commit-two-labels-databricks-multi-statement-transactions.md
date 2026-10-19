---
layout: post
tags:
  - databricks
  - lakehouse
  - architecture
  - sql
---

# One Commit, Two Labels: What Databricks Multi-Statement Transactions Change for an Architect

On 16 October 2026, Benjamin Mathew, Prakhar Jain, and Sumeet Varma published multi-statement transactions as generally available on Delta Lake and Apache Iceberg. The product notes do not. On 22 July the AWS, Azure, and Google Cloud notes marked writes to Unity Catalog managed Delta tables generally available, and left writes to managed Iceberg tables in Private Preview. The transactions pages I read on 19 October still show that banner. AWS and Google Cloud were last updated on 30 September. Azure prints it as an Important note. The catalog-commits page, last updated 11 September, repeats it. The October notes, through the 19 October entry on an unchanged job save, have no transactions bullet on AWS or on Azure.

DevFeed dated the post 16 October. brickster.ai and DigitalNeuron are carrying it too. Matei Zaharia's LinkedIn post that still ranks for the feature is 11 March 2026, and it says public preview. I did not find a Hacker News thread for the 16 October post.

## The catalog owns the boundary

A Delta commit has been one table. The catalog-commits page says Unity Catalog can hold one boundary across tables and still keep each table's log. The switch is `delta.feature.catalogManaged = supported`. It is off until someone sets it. Creating a managed table with it takes Databricks Runtime 16.4 or above. Turning it on for an existing managed table takes 18.0 or above. That alter can take several minutes on a heavily written table. Do not cancel the upgrade or downgrade `ALTER` or `DROP`. An interruption can lock the table. `DESCRIBE DETAIL` shows `catalogManaged` in `tableFeatures` when it is on. A streaming table or a materialized view takes the property on its definition, not through `ALTER TABLE`. Runtime 17.3 or above is the floor for those objects, and a single-user cluster cannot access a streaming table that has the property.

I described the Lakebase commit on 9 October. That path is Postgres WAL, safekeepers, and a pageserver. This path is a SQL block. A successful transaction is one Delta log entry, however many statements ran, with the statements kept as JSON on that entry.

## Two syntaxes, two conflict rules

| Mode | Syntax | Compute | Conflict check |
| --- | --- | --- | --- |
| Non-interactive | `BEGIN ATOMIC ... END` | SQL warehouse, serverless, or Runtime 18.0 and above | Row level, when row-level concurrency is already enabled |
| Interactive | `BEGIN TRANSACTION` then `COMMIT` | SQL warehouse only | Table level, except an insert that does not read the target |

```sql
BEGIN ATOMIC
  UPDATE claims
  SET status = 'closed'
  WHERE claim_id = 1700 AND status = 'open';
  INSERT INTO claim_ledger (claim_id, event)
  VALUES (1700, 'closed');
END;
```

Success commits. An error rolls the block back. `SIGNAL` is how a check the job wrote does the same. An atomic block cannot hold another. `CALL` of an atomic procedure is flattened into the outer transaction, and the language page, updated 7 October, says that is not a nested transaction. Privileges are checked as each statement runs.

Interactive mode leaves the decision to the session. A second session sees nothing until `COMMIT`. A failed statement does not roll back by itself. `ROLLBACK` is required while the session is open, including after the 10-minute idle timeout. Both modes end at 48 hours. The modes page, updated 21 September, wants `ROLLBACK` at the start of an interactive session.

The 16 October post says two transactions can write one table when they miss the same row, unlike a warehouse that locks. It prints no number. Conflicts are found at commit. Row-level detection is non-interactive only, and only when the target already has it: Runtime 14.3 LTS or above, no partitions, deletion vectors on. `MERGE` does not get it on AWS GovCloud or on a single-user cluster. Interactive mode stays at the table. JDBC 3.0.5 and ODBC 2.10.0 open that session with autocommit off. JDBC can also send one `BEGIN ATOMIC` string. Inside the interactive session it refuses `CALL`, catalog metadata, and a timezone change. One connection runs one transaction.

## What the block may touch

Delta and Iceberg tables, streaming tables, views, and materialized views can be read with no hint. An external Parquet, Avro, CSV, or JSON table, or a federated JDBC table, needs `allow_nontransactional_read`, and that read is not repeatable. A path read fails with `PATH_BASED_ACCESS`. Time travel and `SHOW TABLES` stay outside.

Writes go only to managed Delta or Iceberg tables that already have `catalogManaged`. Supported DML includes `INSERT`, `UPDATE`, `DELETE`, `MERGE`, and `COPY INTO`, plus `DESCRIBE TABLE`, `SHOW COLUMNS`, and `GET DIAGNOSTICS`. `CREATE`, `ALTER`, and `DROP` stay outside. The 16 October post says a developer can stage rows in a temporary table. I would create that table before `BEGIN`. A second `COPY INTO` that commits first fails the one inside the block.

A streaming table can be a sink. `forEachBatch` may run `BEGIN ATOMIC`, and the Structured Streaming checkpoint does not advance with that commit. I wrote on 15 October about a checkpoint that keeps state. This block does not keep that offset. The caps are 100 tables, reads and writes together, 100 views, and 100 intermediate commits per table. Lineage remains after a rollback. The 16 October post says the block respects row filters and column masks. The transactions page does not. On 13 October I wrote about the external-engine door. I would select the masked column from inside the block before I copied that sentence across. Iceberg writes stay in Private Preview until a note retires the banner.

<img src="/assets/images/multi-statement-transactions-two-labels.svg" alt="One catalog commit over two labels: the 16 October blog says Delta and Iceberg are generally available, while the July notes and the September docs still mark Iceberg writes as Private Preview. Non-interactive mode can detect row conflicts. Interactive mode detects table conflicts on a SQL warehouse." title="The catalog is the boundary. The blog and the docs do not share a label for Iceberg." style="width:100%; height:auto;">

## Where it fits, and where it does not

I would use `BEGIN ATOMIC` when two managed Delta tables must change together, catalog commits are already on, and the job can retry the whole block. A claim close and its ledger row. The conflict fails the commit. It does not wait. I would use interactive mode when a person has to look and then decide, on a SQL warehouse, with `ROLLBACK` first. I would leave Iceberg writes on the enrollment form. I would leave a partitioned target, and a `MERGE` on GovCloud or a dedicated cluster, out of any plan that assumes a row check. A stream checkpoint, DDL, time travel, and a path read stay outside the promise. Lakebase remains the Postgres commit from 9 October.

## What goes in the design doc

Write both labels. Delta writes, generally available on 22 July, and the account can lag. Iceberg writes, Private Preview on the 30 September page. Pin `catalogManaged` before the syntax. Name the mode next to the conflict check, with the 100-table cap, the 48-hour limit, the idle timeout, and the streaming checkpoint beside it. I would ship the July label.

## References

- Benjamin Mathew, Prakhar Jain, and Sumeet Varma, "Transactions are Generally Available, advancing warehousing on the Lakehouse," Databricks Blog, 16 October 2026. [https://www.databricks.com/blog/transactions-are-generally-available-advancing-warehousing-lakehouse](https://www.databricks.com/blog/transactions-are-generally-available-advancing-warehousing-lakehouse)
- Databricks, Lakehouse category listing for that post. [https://www.databricks.com/blog/category/platform/lakehouse](https://www.databricks.com/blog/category/platform/lakehouse)
- DevFeed, "Transactions are Generally Available, advancing warehousing on the Lakehouse," 16 October 2026. [https://devfeed.tech/articles/transactions-are-generally-available-advancing-warehousing-on-the-lakehouse-88969](https://devfeed.tech/articles/transactions-are-generally-available-advancing-warehousing-on-the-lakehouse-88969)
- brickster.ai, "Transactions are Generally Available, advancing warehousing on the Lakehouse." [https://www.brickster.ai/news/databricks-blog/transactions-are-generally-available-advancing-warehousing-lakehouse](https://www.brickster.ai/news/databricks-blog/transactions-are-generally-available-advancing-warehousing-lakehouse)
- DigitalNeuron, "Databricks Makes Multi-Statement Transactions Generally Available on the Lakehouse." [https://digitalneuron.net/en/news/databricks-multi-statement-transactions-generally-available](https://digitalneuron.net/en/news/databricks-multi-statement-transactions-generally-available)
- Matei Zaharia, LinkedIn post on the public preview, 11 March 2026. [https://www.linkedin.com/posts/mateizaharia_transactions-on-the-open-lakehouse-unlock-activity-7437308445776850944-kokF](https://www.linkedin.com/posts/mateizaharia_transactions-on-the-open-lakehouse-unlock-activity-7437308445776850944-kokF)
- Databricks, "July 2026" release notes, transactions generally available for managed Delta tables on 22 July 2026. [https://docs.databricks.com/aws/en/release-notes/product/2026/july](https://docs.databricks.com/aws/en/release-notes/product/2026/july)
- Databricks, "July 2026" release notes (Google Cloud). [https://docs.databricks.com/gcp/en/release-notes/product/2026/july](https://docs.databricks.com/gcp/en/release-notes/product/2026/july)
- Microsoft, "July 2026" release notes (Azure Databricks). [https://learn.microsoft.com/en-us/azure/databricks/release-notes/product/2026/july](https://learn.microsoft.com/en-us/azure/databricks/release-notes/product/2026/july)
- Databricks, "October 2026" release notes (AWS). [https://docs.databricks.com/aws/en/release-notes/product/2026/october](https://docs.databricks.com/aws/en/release-notes/product/2026/october)
- Microsoft, "October 2026" release notes (Azure Databricks). [https://learn.microsoft.com/en-us/azure/databricks/release-notes/product/2026/october](https://learn.microsoft.com/en-us/azure/databricks/release-notes/product/2026/october)
- Databricks, "Transactions" (AWS), last updated 30 September 2026. [https://docs.databricks.com/aws/en/transactions/](https://docs.databricks.com/aws/en/transactions/)
- Databricks, "Transactions" (Google Cloud), last updated 30 September 2026. [https://docs.databricks.com/gcp/en/transactions/](https://docs.databricks.com/gcp/en/transactions/)
- Microsoft, "Transactions" (Azure Databricks). [https://learn.microsoft.com/en-us/azure/databricks/transactions/](https://learn.microsoft.com/en-us/azure/databricks/transactions/)
- Databricks, "Transaction modes" (AWS), last updated 21 September 2026. [https://docs.databricks.com/aws/en/transactions/transaction-modes](https://docs.databricks.com/aws/en/transactions/transaction-modes)
- Databricks, "ATOMIC compound statement," last updated 7 October 2026. [https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-txn-begin-atomic](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-txn-begin-atomic)
- Databricks, "Catalog commits," last updated 11 September 2026. [https://docs.databricks.com/aws/en/tables/features/catalog-commits](https://docs.databricks.com/aws/en/tables/features/catalog-commits)
- Databricks, "Row-level concurrency." [https://docs.databricks.com/aws/en/optimizations/isolation/row-level-concurrency](https://docs.databricks.com/aws/en/optimizations/isolation/row-level-concurrency)
