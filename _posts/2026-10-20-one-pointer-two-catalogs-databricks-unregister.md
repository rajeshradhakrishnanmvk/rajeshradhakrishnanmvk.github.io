---
layout: post
tags:
  - databricks
  - lakehouse
  - architecture
  - iceberg
---

# One Pointer, Two Catalogs: What Databricks UNREGISTER Changes for an Architect

On 19 October 2026, Elizabeth Bowman published a test for an open catalog. The files sit in storage you own. The maintenance you want runs there. A table can leave without a copy. That last part is `REGISTER` and `UNREGISTER` on the Iceberg REST catalog. The post says both are in Private Preview, and that the way in is the account team. The October product notes, through the 19 October entry on an unchanged job save, have no bullet for either call. The managed-tables page I read, updated 8 October, does not name them.

Ryan Blue and Marco Kroll described the calls on 5 October. Data Phoenix, the same day, already called the Unity Catalog support a private preview, and it noted that the announcement names no privilege, no downtime, and no recovery if the second call fails. Lakenaut carries the same label. I did not find a Hacker News thread for the 19 October post.

## The pointer is the handoff

An engine asks the catalog to load the table. The catalog returns the current metadata and where the files sit. The engine reads the Parquet from that bucket. The catalog is what names the current metadata, so two writers do not silently pass each other.

`DROP` is the wrong exit. On a Unity Catalog managed table it removes the name and marks the files for deletion. `UNDROP` can bring the name back inside a recovery window. On an external table, `DROP` removes the name and leaves the files. Neither call hands a pointer to another catalog.

`UNREGISTER` is an empty POST:

```text
POST /v1/{prefix}/namespaces/{namespace}/tables/{table}/unregister
```

The OpenAPI file on the Apache Iceberg main branch says the name is gone from that catalog and the data and metadata files stay. The response must include `metadata-location` and `metadata`, and that metadata must include every commit from before the call. A later commit through the same catalog must fail. An `Idempotency-Key` header is optional. Ryan Blue's pull request 16400 added the path and merged on 26 May 2026 after 16 +1 votes. The REST catalog overview I opened on 20 October does not name the endpoint. It links out to the YAML.

`REGISTER` is a POST to `/v1/{prefix}/namespaces/{namespace}/register` with a name and that metadata location. `overwrite` defaults to false, so an existing name is not replaced. Register while the old catalog still has the table and the catalogs do not talk. Both accept commits. The first write forks the history. The 5 October post calls that split-brain. The order in the spec is unregister, take the pointer, then register. That post also says a production move still has to stop writers and repoint jobs, and that those steps would come later. The 19 October post is not that follow-up.

## What "managed" is allowed to mean

Bowman writes that managed describes who maintains the table, not who owns the bucket. The page on managed and external assets, updated 11 September, says the files stay in your cloud account and Unity Catalog chooses the path. Its summary table says a drop permanently deletes those files after an 8-day retention period.

The managed-tables page, updated 8 October, uses another clock. `UNDROP` works for 7 days by default. A catalog or a schema can set 0 hours, or anywhere from 7 to 30 days, and that configuration is in Public Preview. The schema wins when both are set. It applies only to tables dropped after the change. Bowman's sentence puts earlier versions and a dropped-table restore together and says up to 30 days. Thirty days is the top of that preview range. It is not the default.

The storage guide, updated 9 October, still has a second place a catalog can live. Databricks-managed storage sits in a Databricks-owned account and is optional for Unity Catalog catalogs. Customer-managed storage sits in yours. That guide also says not to hand out storage-level access to the buckets. Bowman writes that 99 percent of managed tables are already in customer-owned storage, and shows no count. I would leave 99 out of the design doc. Predictive optimization, the maintenance that post wants to keep, runs `OPTIMIZE`, `VACUUM`, and `ANALYZE` on managed Delta and Iceberg tables, on by default for accounts created on or after 11 November 2024. The Zepto quote and the Snowflake comparisons stay in the blog. I did not open the Snowflake pages.

## SET MANAGED copies. The handoff does not.

The same 19 October page says two things about conversion. The roadmap says a future `SET MANAGED IN PLACE` would turn a foreign or external table into a managed table on the same path, with no data movement. The FAQ speaks in the present: convert a foreign table with `SET MANAGED`.

The convert page, updated 11 September, is a third sentence, and it is about Delta Lake. An external table is copied into the managed location, then switched. A foreign table must say `MOVE` or `COPY`, and `COPY` makes a second copy. The foreign sources there are Hive metastore and Glue, the runtime floor is 17.3, and a Hive table must already be external. Iceberg is not in that list. `UNSET MANAGED` rolls a converted external table back within 14 days, onto its old path inside Unity Catalog. It does not hand a pointer to another catalog.

Foreign Iceberg stays read-only until the owner of the commits changes. The Iceberg client page lists managed Iceberg as readable and writable, and foreign Iceberg as readable only. The 9 October guide says a foreign catalog is read-only. Bowman's FAQ agrees, and it says `REGISTER` is how Unity Catalog takes those commits. That call is the private preview.

<img src="/assets/images/unregister-one-pointer-two-catalogs.svg" alt="UNREGISTER on catalog A returns a metadata location. REGISTER on catalog B attaches that pointer. The files stay in the bucket. DROP is a separate door that marks managed files for deletion." title="One pointer, two catalogs. The files do not move. DROP is not the handoff." style="width:100%; height:auto;">

## Where it fits, and where it does not

I would use the pair for an Iceberg table whose files the next catalog can already read, with writers stopped and jobs pointed at the new name before the next commit. I would ask the account team first. A missing October bullet is not a denial, and the blog is not a general-availability date.

I would not use `DROP`, and I would not point a Delta table at an Iceberg REST path. I would not register while the old catalog still has the table. I would not run the pair on the only copy of a production table. After `UNREGISTER` the name is gone and the files are still in the bucket. The 5 October post gives no compensating call if `REGISTER` then fails, and Data Phoenix notes that gap. Neither post names the bucket grant the next catalog needs. The 9 October guide says not to hand storage access out in the meantime.

## What goes in the design doc

Write Private Preview, and write that the account team is the gate. Draw unregister, then the metadata location, then register. Name the bucket, including whether it is the optional Databricks-owned storage. Put recovery on its own line: 7 days by default, 30 only where `RETAIN DROPPED` is set, and that clause is Public Preview. The 8-day cell is the 11 September table. Leave 99 percent and the Zepto quote in the blog. The follow-up promised on 5 October was still not this post.

## References

- Elizabeth Bowman, "What makes a catalog open? Unity Catalog Advances the Open Lakehouse," Databricks Blog, 19 October 2026. [https://www.databricks.com/blog/what-makes-catalog-open-unity-catalog-advances-open-lakehouse](https://www.databricks.com/blog/what-makes-catalog-open-unity-catalog-advances-open-lakehouse)
- Ryan Blue and Marco Kroll, "Unlocking Data Portability: Preventing Catalog Lock-in with REGISTER and UNREGISTER APIs," Databricks Blog, 5 October 2026. [https://www.databricks.com/blog/unlocking-data-portability-preventing-catalog-lock-register-and-unregister-apis](https://www.databricks.com/blog/unlocking-data-portability-preventing-catalog-lock-register-and-unregister-apis)
- Data Phoenix, "Databricks adds Iceberg catalog handoff APIs in private preview," 5 October 2026. [https://dataphoenix.info/news/databricks-iceberg-catalog-portability-apis](https://dataphoenix.info/news/databricks-iceberg-catalog-portability-apis)
- Lakenaut, "Iceberg REST catalogs gain UNREGISTER; Unity Catalog support is in private preview." [https://lakenaut.dev/news/iceberg-rest-catalogs-gain-unregister-unity-catalog-support-is-in-private-previe-e2838a/](https://lakenaut.dev/news/iceberg-rest-catalogs-gain-unregister-unity-catalog-support-is-in-private-previe-e2838a/)
- Ryan Blue, "REST Spec: Add unregister table endpoint," apache/iceberg pull request 16400, merged 26 May 2026. [https://github.com/apache/iceberg/pull/16400](https://github.com/apache/iceberg/pull/16400)
- Apache Iceberg, REST catalog OpenAPI, `unregisterTable` and `registerTable` on the main branch. [https://github.com/apache/iceberg/blob/main/open-api/rest-catalog-open-api.yaml](https://github.com/apache/iceberg/blob/main/open-api/rest-catalog-open-api.yaml)
- Apache Iceberg, "REST Catalog Spec" overview. [https://iceberg.apache.org/rest-catalog-spec/](https://iceberg.apache.org/rest-catalog-spec/)
- Databricks, "October 2026" release notes (AWS). [https://docs.databricks.com/aws/en/release-notes/product/2026/october](https://docs.databricks.com/aws/en/release-notes/product/2026/october)
- Databricks, "Unity Catalog managed tables for Delta Lake and Apache Iceberg," last updated 8 October 2026. [https://docs.databricks.com/aws/en/tables/managed](https://docs.databricks.com/aws/en/tables/managed)
- Databricks, "Managed versus external assets in Unity Catalog," last updated 11 September 2026. [https://docs.databricks.com/aws/en/data-governance/unity-catalog/managed-versus-external](https://docs.databricks.com/aws/en/data-governance/unity-catalog/managed-versus-external)
- Databricks, "Phase 5: Design storage architecture," last updated 9 October 2026. [https://docs.databricks.com/aws/en/lakehouse-architecture/deployment-guide/storage](https://docs.databricks.com/aws/en/lakehouse-architecture/deployment-guide/storage)
- Databricks, "Predictive optimization for Unity Catalog managed tables." [https://docs.databricks.com/aws/en/optimizations/predictive-optimization](https://docs.databricks.com/aws/en/optimizations/predictive-optimization)
- Databricks, "Convert external or foreign Delta Lake tables to Unity Catalog managed tables," last updated 11 September 2026. [https://docs.databricks.com/aws/en/tables/convert-to-managed](https://docs.databricks.com/aws/en/tables/convert-to-managed)
- Databricks, "ALTER TABLE," `SET MANAGED` and `UNSET MANAGED`. [https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-alter-table](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-alter-table)
- Databricks, "Access Databricks tables from Apache Iceberg clients." [https://docs.databricks.com/aws/en/external-access/iceberg](https://docs.databricks.com/aws/en/external-access/iceberg)
