---
layout: post
tags:
  - databricks
  - lakehouse
  - architecture
  - postgres
---

# One Logical Copy, Two Engines: What Databricks LTAP Changes for an Architect

On 16 June 2026, at Data + AI Summit, Databricks announced Lake Transactional/Analytical Processing, LTAP. The press release still said it was coming soon. The architecture page I would hand to a design review was updated on 6 October. That is the document worth arguing from.

The launch sentence is easy to repeat. One copy of data in the lake. Postgres for transactions, the lakehouse for analytics, open formats, a single governance model. The October page is staffable. LTAP is an architecture, delivered as four Lakebase capabilities that are not at the same level of maturity, and that do not answer the same question. Unity Catalog governs analytical readers. Postgres `GRANT` and `REVOKE` still govern every client that connects to the database.

That split matters if you run services that commit orders to Postgres and an agent that wants both the row that just committed and the history in the lake. The storage idea is the part I would adopt. The sentence that says the pipelines are already gone is the part I would keep redrawing until the picture matches the capability table.

## The bill for two stacks

Transactional work touches a few rows and wants the whole row back quickly. Analytical work scans and aggregates. A row layout serves the first. A column layout serves the second. The bridge between them is what teams actually pay for: change data capture, a streaming job, a replica whose job is to copy. The copy adds lag, it competes with the primary, and lineage breaks on the way across. A GDPR delete then becomes a search across systems that no longer agree about which copy is real.

Agents make that lag harder to live with. Jonathan Katz, in a Databricks post published on 7 September 2026, uses fraud as the concrete case. A decision that waits on a batch copy minutes or hours old misses a card transaction clearing in hundreds of milliseconds. Point the same history scan at the operational primary and a fleet of agents contends with the short writes that primary exists to serve. His economic line is the one I would keep in the review: "Storage is the cheap part of any data system. Compute is the expensive part." HTAP put both workloads in one engine and paid for both with the same compute. LTAP keeps an engine for each workload and unifies the storage under them. Katz describes the separation as inherited from Neon.

## Where a commit actually lands

Lakebase is standard Postgres with the disk pulled out of the machine. The compute node holds shared buffers and a local cache. It owns no durable data. The Lakebase architecture page, updated 27 July 2026, gives the commit path:

1. Postgres changes pages in memory and produces WAL records.
2. Compute streams those records to safekeepers.
3. A quorum of safekeepers acknowledges them, using a Paxos-based protocol, and the client hears success.
4. Pageservers apply the WAL afterward and persist pages to object storage. On AWS that store is Amazon S3. Rebuilding a page stays off the commit path.

LTAP adds a step to that flush. As the storage layer writes object storage, it transcodes row-oriented Postgres data into Parquet, readable as Delta and Iceberg. The transcode runs in the storage layer, isolated from the Postgres process serving the application. Indexes stay in their original representation, so point lookups stay point lookups. Values that do not map cleanly onto Parquet, including `NaN`, `NUMERIC` overflow, and extension types such as vector, array, geography, and JSON, are preserved in an overflow field that holds the canonical Postgres representation. Intermediate row versions are preserved with them.

June coverage in [The New Stack](https://thenewstack.io/databricks-is-rebuilding-the-data-stack-for-ai-agents/) described Mooncake Labs, which it reported Databricks acquired in 2025, as mirroring Postgres changes into a second columnar copy. I would draw both physical layouts, because the October architecture page is explicit: analytical engines read columnar Parquet, and Lakebase keeps Postgres pages for OLTP. Databricks' phrase for the pair is one logical copy. There is no CDC job you deploy to produce that columnar layout. Other copies still exist in this architecture. They show up the moment you follow a different capability.

<img src="/assets/images/ltap-one-logical-copy.svg" alt="LTAP write path from Postgres through safekeepers and pageservers into columnar object storage, with Lakehouse RT, Change Data Feed, and synced tables as separate readers and writers" title="One logical copy in object storage. Postgres pages serve transactions. Analytical engines read the columnar layout. Synced tables are a separate, lakehouse-owned path back into Postgres." style="width:100%; height:auto;">

A Lakehouse//RT query against live Lakebase data, as that architecture page describes it, reads the bulk of the data from the columnar copy, asks Postgres for the current log sequence number so the result is transactionally consistent, and merges the small tail of changes the pageserver has not yet flushed. The scan is not served by Postgres. That capability is Beta on the AWS page updated 6 October. The Google Cloud page from the same day marks it not available. That page also says Lakebase is in Beta on GCP as of 15 June. It does not name the year.

## Four capabilities, one writer

The roster on the AWS architecture page updated 6 October 2026 is short.

| Capability | Status on that AWS page |
| --- | --- |
| Register a Lakebase database in Unity Catalog | GA |
| Synced tables, with LTAP Direct Writes for bulk loads | GA, Direct Writes in Beta |
| Lakehouse//RT querying Lakebase | Beta |
| Lakebase Change Data Feed | Public Preview |

The Google Cloud page from the same day lists registration and synced tables as GA, and marks Lakehouse//RT and Change Data Feed as not available. Its synced-tables row does not mention Direct Writes. I could not find a matching Azure architecture page, so the table for the cloud you run belongs in the design, ahead of the AWS column.

Each table has a single writer, either Lakebase or the lakehouse. The writer picks the capability.

When the application owns the write, analytics reads that data in place. Lakehouse//RT answers for current state. Change Data Feed answers for the change stream: row images in a Unity Catalog Delta table, for pipelines and audit. The architecture page is careful, and the care is worth keeping. Both read Lakebase data. Databricks describes both as operating on the single copy, and describes both as distinct from the external CDC stack LTAP retires.

When the lakehouse owns the write, synced tables serve it into Postgres for low-latency lookups. The synced-tables page, also updated 6 October, calls this reverse ETL. A managed Lakeflow pipeline maintains the Postgres table. Snapshot reloads the data and is the documented fit when a cycle changes more than 10 percent of the source rows. Triggered and Continuous apply row-level changes, so the source needs a change feed. Continuous is the lowest lag and the highest cost, with a minimum interval of 15 seconds. Direct Writes, supported on Postgres 16, 17, and 18, puts the bulk load into the storage layer behind the branch instead of through the live compute endpoint. You select it when the synced table is created. An existing synced table does not gain it later. Delete the table and create it again.

One service often needs both directions. A price or a feature computed in the lakehouse is served into Postgres. An order committed in Postgres is visible to analytics. Two datasets, two writers, two capabilities.

Registration creates a read-only Unity Catalog catalog: one database per catalog. Queries against it run on a Serverless SQL warehouse. A Pro or Classic warehouse returns `PERMISSION_DENIED`.

```sql
SELECT o.order_id, o.status, f.risk_band
FROM orders_db.public.orders AS o
JOIN main.gold.order_features AS f
  ON o.order_id = f.order_id
WHERE o.placed_at >= current_date - INTERVAL 1 DAY;
```

## What an architect still has to draw

**Two permission lists.** After registration, Unity Catalog permissions, lineage, and audit apply to the external compute reading the catalog. They do not apply to individual Postgres tables. The connection string used by a .NET service, an agent, or `psql` is a Postgres role. A Unity Catalog `SELECT` grant leaves that role untouched. I would publish both lists in the design, and I would review the Postgres role on its own.

**Change Data Feed is a product you operate.** Its own page was last updated on 11 June 2026 and still marks the feature Public Preview. The 6 October overview agrees. The `wal2delta` extension runs inside Lakebase compute and captures the WAL through logical decoding. Tables need `REPLICA IDENTITY FULL`, so updates and deletes carry the full before-and-after image. Rows land in `lb_<table_name>_history` about every 15 seconds, with `_pg_change_type` set to `insert`, `delete`, `update_preimage`, or `update_postimage`. The feed is scoped to a schema, and one feed covers one database. Adding, dropping, or retyping a column re-snapshots that table. Partitioned tables are unsupported. A table with no rows does not appear until it has one. A row filter or column mask on the destination stops the feed. Enabling Delta change data feed on that destination breaks the re-snapshot. Catalogs that use default storage are unsupported, and so is managed storage reachable only through a private endpoint. Disabling CDF stops the feed for every schema in the project. Types without a Delta equivalent are stored as strings. `NUMERIC` NaN becomes NULL, and precision above 38 falls back to string. That fidelity story is different from the overflow field on the storage transcode. Name the path before you promise a vector value or a NaN will survive.

**Lakehouse//RT is a Beta warehouse with a short allow-list.** The product page, updated 6 October, says performance characteristics and the supported feature set will change before general availability. You cannot upgrade an existing SQL warehouse to Lakehouse//RT, or downgrade one back. Summit coverage in SiliconANGLE reported Databricks describing it as a drop-in replacement for current warehouse deployments. The adoption constraint I would write down is the October one: a new warehouse, `SELECT` only, Statement Execution API only. A driver on the legacy Thrift protocol receives a `501`. Attribute-based access control, row filters, and column masks are unsupported. Genie, Genie Agents, and Jobs tasks are unsupported. Compliance security profiles, outbound Private Link, and serverless egress control are unsupported. `GEOGRAPHY` and `GEOMETRY` are unsupported. The sub-second figure on that page is for selective reads against Unity Catalog Delta or Iceberg tables. The documented preparation is to run the query on a serverless SQL warehouse, confirm it already finishes in a few seconds, then filter early and project fewer columns. A custom agent can call the Statement Execution API. A Genie Agent on this warehouse is outside what that page supports. Enabling it is a workspace preview, and the same page says to ask the Databricks account team to turn the feature on for the account.

**Synced tables are a managed copy, with ceilings.** Each sync can use up to 16 connections. The same page documents up to 1,000 concurrent connections on Lakebase Postgres. Primary-key columns on the synced table are not nullable, and source rows with a null key are left out. Each source table can back at most 20 synced tables, and tables pending deletion still count. Databricks recommends read queries against the Postgres side. I would encode that in the role.

**A branch is a catalog you already have.** Branching is a copy-on-write metadata operation against shared storage, which is why an experiment or an agent can attach to a branch and discard it. The registration page, updated 9 September 2026, says you cannot register that branch as its own Unity Catalog catalog. The branch inherits the parent's registration metadata, and a new catalog registration fails.

Adopting these capabilities requires no migration, and no change to how applications connect, when the application is already on Lakebase. Extensions, indexes, and queries stay. The real adoption decision is whether the system of record moves to Lakebase. LTAP is built on that storage. It is a poor fit for a primary you intend to leave on another engine.

## Where it fits

I would use it when the system of record can live in Lakebase and a reader needs committed rows without placing a scan on the primary. A dashboard on live orders. An agent that has to see a payment committed moments ago, on an analytical engine, while checkout keeps the Postgres pool. A service that already speaks Postgres and also needs a price, a segment, or a feature a lakehouse job produced. That last path is synced tables, and it is the capability I would ship first, because registration and synced tables are the rows marked GA.

I would use Change Data Feed when the consumer is a medallion pipeline or an audit history, and a flush on the order of 15 seconds matches the requirement. Current state and the change log are different products. The design should say which question it is answering.

## Where it does not fit

I would leave the pattern alone when the operational database is staying where it is. I would leave Lakehouse//RT and Change Data Feed out of a Google Cloud design until that capability table changes. I would leave it when the security review requires a single authorization system, or requires column masks on the analytical path in front of me. Lakehouse//RT's current page does not support those masks. Change Data Feed stops if you add them to the history table. I would leave it when one table must accept writes from the application and from a lakehouse job. I would leave Lakehouse//RT when the caller is a Genie Agent, a Jobs task, or a workspace that requires a compliance security profile, and when the work is a wide transform. Those transforms belong in Spark or Lakeflow. Change Data Feed can feed them.

Summit coverage framed the announcement as the end of pipelines. I would remove a pipeline from a design when I can name the mechanism that replaced it: the storage transcode, a synced table, or a change-history table. Each one has its own freshness, its own failure mode, and its own operator.

## What goes in the design doc

**Name the writer of every table.** Application or lakehouse. Freshness and the capability both fall out of that choice.

**Publish the two permission lists.** Unity Catalog grants for warehouse readers. Postgres roles for every connection string, agents included.

**Match the read to the question.** Current and consistent means Lakehouse//RT, where the cloud offers it, with the Beta constraints in the risk log. Change history means Change Data Feed, with `REPLICA IDENTITY FULL`, the re-snapshot behavior, and the private-endpoint limit in the runbook. A key lookup of lakehouse data inside the application means synced tables, with the sync mode taken from the documented lag and cost, and Direct Writes only on a new table running Postgres 16, 17, or 18.

**Treat June performance claims as launch claims.** The October Lakehouse//RT page says the characteristics will change before general availability. Prove the query on a serverless SQL warehouse before you promise a latency number to a product owner.

**Keep historical scans off the primary.** Short reads and writes stay on Postgres, through a narrow role. The scan goes to the analytical engine. A herd of agents, each issuing something that looks cheap on its own, is how an operational pool falls over. Katz's fraud example is that failure mode. The Postgres role still needs a human review, because Unity Catalog does not see that connection.

**Read the capability table again the week you build.** The 16 June press release is the announcement. The 6 October architecture page is the contract, and the AWS and Google Cloud copies of it already disagree.

LTAP is the Databricks architecture that takes the row-versus-column argument seriously and keeps a purpose-built engine on each side. The storage flush, which makes a committed Postgres row readable by an analytical engine without a connector farm, is the piece I would bet a design on. Pipelines, physical copies, and a second permission list are still in the picture. Some of them moved into the platform. They remain the architect's to draw.

## References

- Databricks, "Databricks Launches LTAP: The First Lake Transactional/Analytical Processing Architecture," 16 June 2026. [https://www.databricks.com/company/newsroom/press-releases/databricks-launches-ltap-first-lake-transactionalanalytical](https://www.databricks.com/company/newsroom/press-releases/databricks-launches-ltap-first-lake-transactionalanalytical)
- Databricks, "LTAP architecture" (AWS), last updated 6 October 2026. [https://docs.databricks.com/aws/en/oltp/projects/ltap-overview](https://docs.databricks.com/aws/en/oltp/projects/ltap-overview)
- Databricks, "LTAP architecture" (Google Cloud), last updated 6 October 2026. [https://docs.databricks.com/gcp/en/oltp/projects/ltap-overview](https://docs.databricks.com/gcp/en/oltp/projects/ltap-overview)
- Databricks, "Lakebase architecture" (AWS), last updated 27 July 2026. [https://docs.databricks.com/aws/en/oltp/projects/architecture](https://docs.databricks.com/aws/en/oltp/projects/architecture)
- Databricks, "Register a Lakebase database in Unity Catalog," last updated 9 September 2026. [https://docs.databricks.com/aws/en/oltp/projects/register-uc](https://docs.databricks.com/aws/en/oltp/projects/register-uc)
- Databricks, "Serve lakehouse data with synced tables," last updated 6 October 2026. [https://docs.databricks.com/aws/en/oltp/projects/sync-tables](https://docs.databricks.com/aws/en/oltp/projects/sync-tables)
- Databricks, "Lakebase Change Data Feed," last updated 11 June 2026. [https://docs.databricks.com/aws/en/oltp/projects/lakebase-cdf](https://docs.databricks.com/aws/en/oltp/projects/lakebase-cdf)
- Databricks, "Lakehouse Real-Time," last updated 6 October 2026. [https://docs.databricks.com/aws/en/compute/sql-warehouse/real-time](https://docs.databricks.com/aws/en/compute/sql-warehouse/real-time)
- Jonathan Katz, interviewed in "The 40-year-old database rule agents just broke," Databricks Blog, 7 September 2026. [https://www.databricks.com/blog/40-year-old-database-rule-agents-just-broke-how-ltap-unifies-oltp-and-olap-workloads](https://www.databricks.com/blog/40-year-old-database-rule-agents-just-broke-how-ltap-unifies-oltp-and-olap-workloads)
- Frederic Lardinois, "Databricks wants to merge the two databases every company runs," The New Stack, 16 June 2026. [https://thenewstack.io/databricks-is-rebuilding-the-data-stack-for-ai-agents/](https://thenewstack.io/databricks-is-rebuilding-the-data-stack-for-ai-agents/)
- Paul Gillin, "Databricks declares the end of pipelines with a unified platform for operational and analytical data," SiliconANGLE, 16 June 2026. [https://siliconangle.com/2026/06/16/databricks-declares-end-pipelines-unified-platform-operational-analytical-data/](https://siliconangle.com/2026/06/16/databricks-declares-end-pipelines-unified-platform-operational-analytical-data/)
