---
layout: post
tags:
  - databricks
  - lakehouse
  - architecture
  - governance
---

# The Row Leaves Already Filtered: What Cross-Engine ABAC Changes for an Architect

On 1 October 2026, Databricks marked cross-engine attribute-based access control generally available. The same day, Google published the account of a change the Apache Iceberg community had merged into the REST catalog: a `read-restrictions` object on `loadTable`. A post from 10 September, by Daniel Weeks, Ryan Blue, and Andrei Tserakhau, is the document I would take into a design review, because it is the one that refuses to treat those as the same door. The October product page is the contract for the door Databricks is selling. The OpenAPI schema is the contract for the door Iceberg standardized. They do not describe the same enforcement point.

The June announcement is older than both, and it is still in circulation. Alex Jiang, Alex Reid, and Michelle Leon called the Beta on 2 June. That post tells an operator to enroll in a preview. The October release notes do not. I would not close a design from the June get-started list.

## The policy stops at the engine that defined it

A warehouse has one door. The server knows who is asking, and a mask runs before a row is returned. Iceberg took that door out on purpose. The data is Parquet in object storage. Spark, Trino, DuckDB, Flink, BigQuery, or a job a team runs on its own cluster can open the files once the catalog has said where they are. The catalog could grant or deny the table. It had no vocabulary for "this principal sees the row, with the card number reduced to the last four characters."

Teams filled the gap by copying the policy into every engine, copying the table, or granting the table and accepting the extra columns. Open formats made the files portable. The row filter did not travel with them. That is the problem the 1 October documents are trying to close, and they close it in different places.

Inside Databricks the split is already there. The requirements page, updated 9 October, says an ABAC policy runs on serverless compute, on standard compute at Databricks Runtime 16.4 or above, or on dedicated compute at that runtime with fine-grained access control filtering enabled. The billable-usage page, updated 8 October, names the dedicated-compute serverless usage `FINE_GRAINED_ACCESS_CONTROL`. Cross-engine ABAC is a different line, `EXTERNAL_COMPATIBILITY`. A query on a SQL warehouse is not this feature. I would not add the two lines in one estimate.

## What the October page actually runs

The AWS page and the Google Cloud page were both updated on 1 October. The Azure page on Microsoft Learn carries an article date of 1 October and, in the HTML I read, no Beta banner. All three describe the same mechanism. An external engine queries a table that has ABAC policies, row filters, or column masks. Databricks uses a specialized serverless compute layer to filter and return sanitized data to that engine.

The release notes date the general availability to 1 October and warn that releases are staged, so an account can lag by a week or more. I would check the account the week the job is built. I would not treat the absence of a Beta badge on the page as proof that every workspace has the feature tonight.

Four requirements sit on that page.

External data access is enabled on the Unity Catalog metastore. The querying principal has `EXTERNAL USE SCHEMA`. The table is a managed table with catalog commits. The caller authenticates with OAuth machine-to-machine or a personal access token.

Catalog commits are the table property `delta.feature.catalogManaged = supported`. Creating a managed table with it requires Databricks Runtime 16.4 or above. Turning it on for an existing managed table requires Databricks Runtime 18.0 or above. The catalog-commits page, updated 11 September, says that alter synchronizes table state with the catalog and can take several minutes on a table with a high volume of writes. It also says not to cancel the alter. An interruption can lock the table to later reads and writes. `DESCRIBE DETAIL` shows `catalogManaged` in `tableFeatures` when the property is on. The same page marks safe writes from external engines as Beta. Cross-engine ABAC's own page is about reads. I would not borrow the Beta marker from the write bullet, and I would not ignore it when the service also writes.

The documented clients are Apache Spark. The Delta path wants Delta-Spark 4.1 or above and the Unity Catalog Spark connector 0.4 or above, with `ServerSidePlanning.enabled` set to `true`. The Iceberg path wants Iceberg-Spark 1.11 or above and Spark 4.0 or above, pointed at `<workspace-url>/api/2.1/unity-catalog/iceberg-rest/`, with `cache-enabled` set to `false`. The Iceberg block on that page does not set `ServerSidePlanning.enabled`. The Delta jar list is the cloud's filesystem client: S3A on the AWS page, the Azure Hadoop client on the Microsoft Learn page, and the GCS connector on the Google Cloud page. I would copy the block for the cloud and the client I am configuring.

```sql
CREATE TABLE orders.sensitive.payments (
  payment_id BIGINT,
  region STRING,
  card_number STRING
)
USING delta
TBLPROPERTIES ('delta.feature.catalogManaged' = 'supported');
```

The June post describes the scan differently. The engine sends a scan request through the Iceberg REST scan API, Unity Catalog evaluates entitlements and policies, and Unity Catalog returns a filtered scan plan. The engine then reads the filtered files. A column mask is not a file the catalog can withhold, unless the values already live in separate files. Google's 1 October post says the same thing about the older instruments: credential vending and server-side scan planning can withhold storage or whole files, and neither can express "mask this column" or a row filter whose rows are mixed inside a file. The October Databricks page says the serverless layer returns sanitized data. Its limitations mention mask return types, column projections, and aggregations. A rule about a mask function's return type is what I would expect from a compute layer that evaluates the function. I would prove a mask by selecting the masked column from the external engine. A predicate that drops a file is evidence of a file prune.

The June post also says the Beta includes the full expressiveness of Unity Catalog policies, including tag-based rules, conditional logic, and SQL UDFs, and that Starburst and DuckDB are coming soon. The October page does not repeat the expressiveness sentence, and it does not list those engines. The 10 September post uses DuckDB as an example of an untrusted engine the filtering fleet can sit in front of. That is an architecture sentence. It is not a client matrix. I would not put DuckDB in the runbook.

The limitations I would paste above the query are short.

Only reads are supported from an external engine while fine-grained access controls are enforced. To write, the writing principal must be exempt from the ABAC policy. Dynamic views are not supported. Projecting `VARIANT` columns is not supported. Filtering on `BINARY` columns is not supported. A column-masking function whose return type differs from the original column type is not supported. Large aggregations might degrade. Concurrent writes during query planning can make a self-join or a multi-scan query read the same table at different snapshots, and the page says that can produce incorrect results.

The common-patterns page, updated 9 October, says Databricks casts a mask function's output to the column type when the function returns a castable type. That sentence is about policies running in Databricks. The external page's rule is stricter: a different return type is unsupported. A `VARIANT` mask function, which that patterns page recommends so one function can cover many column types, declares a return type of `VARIANT`. I would not assume the cast still happens on the external path. The patterns page also says a personal access token does not set `request.is_on_behalf_of` to `true`. The external page allows that token. A policy written for on-behalf-of agent traffic does not catch a service that connects with a PAT. I would pick the principal and the token type together.

Customers are charged for the serverless resources. The page I read does not print a rate. It points at a pricing page and shows how to read `system.billing.usage` for `billing_origin_product = 'EXTERNAL_COMPATIBILITY'`, broken down by `sku_name` and `identity_metadata.run_as`. The billing reference, updated 8 October, says usage records are typically available within 12 hours. I would not promise a cost the afternoon the job first runs.

## The spec the engine is asked to obey

Read restrictions are the other 1 October document. Google's post, by Talat Uyarer and Sung Yun, walks through the field. The schema I read is `ReadRestrictions` in `open-api/rest-catalog-open-api.yaml` on the Iceberg main tree. `loadTable` may return it. The issue that opened the work showed an earlier filter sketch with a column name in a `term` field. The schema on main does not. Column references inside the filter must use field ids. I would review the schema, not the issue's example.

Two fields. `required-row-filter` is a predicate. Rows for which it evaluates to false must not appear, and no information derived from them may be included. `required-column-projections` is a list of field ids, each with one action. The reader evaluates the filter against the original column values, then applies the projections to the rows that survive. A policy can filter on region and also mask region. If the mask ran first, the filter would be looking at the mask.

The action set is closed. The schema names nine: `mask-alphanum`, `show-first-4`, `show-last-4`, `replace-with-null`, `mask-to-fixed-value`, `truncate-to-year`, `truncate-to-month`, `sha-256-global`, and `sha-256-query-local`. Each action must return the same type as the input, so the schema the engine planned against does not change. `mask-alphanum` turns digits into `n`, keeps a short list of punctuation, and turns every other character into `x`. The schema's example is `iceberg16112018@apache.org` becoming `xxxxxxxnnnnnnnn@xxxxxx.xxx`. `show-last-4` keeps four code points and masks the rest, so `4111-1111-1111-4444` becomes `nnnn-nnnn-nnnn-4444`. `mask-to-fixed-value` is the action that also replaces NULL. The others preserve NULL. `replace-with-null` is illegal on a required field, and a reader that receives one must fail the query. `sha-256-global` is deterministic. Google's post says the cost of that determinism is that a hashed value can be tested against a precomputed guess. It is pseudonymization. `sha-256-query-local` adds a fresh salt of at least 16 bytes per query, and the schema leaves the definition of a query to the implementation.

The fail-closed sentence in the schema is the one I would quote:

> If a reader that supports read-restrictions cannot apply any returned restriction (a filter expression or an action), it must fail the query and must not silently return raw, partial, or empty results.

Unrecognized actions, a filter the reader cannot evaluate, a duplicate field id, a projection on a map key, and a projection on both a nested type and a field inside it all fail the query. Projections are bound to field ids, so renaming `ssn` to `national_id` does not detach the mask. The response applies only to the authenticated principal of that call, including later `planTableScan` and `fetchScanTasks` calls made with it. The schema says the response should not be cached outside that authentication scope. The Databricks Iceberg example sets `cache-enabled` to `false`. I would not treat that client flag as proof that Databricks is returning `read-restrictions`. I would treat a cache of either object, a user-specific scan plan or a restricted `loadTable` response, as something that must not be shared across principals.

Trust is outside the protocol. The 10 September post says a securely configured Trino deployment is an example of a trusted engine, because it can enforce row filters and column masks and the user cannot step around it. The same post's footnote says Spark is untrusted when users control the runtime, because user-submitted code runs without restrictions. Google's post is as direct: the mechanism does not protect data from the engine. It lets the catalog direct a trusted engine to protect data from that engine's users. A client that ignores the restrictions and holds the storage credential can read the raw files. How a catalog decides the caller is that trusted engine is deliberately not in the spec. The discussions have considered mTLS and OAuth. A claim from the client is not enough.

The vocabulary is the other boundary. The 10 September post says policies that depend on subqueries, lookup tables, or custom UDFs cannot be represented, and a catalog that cannot reduce a policy to the nine actions and the predicate has lost the policy rather than translated it. The common-patterns page documents a row filter whose function is an `EXISTS` against an `access_rules` table, compared with `session_user()`. Nothing in `ReadRestrictions` carries that subquery. Google notes that a broader Iceberg expressions proposal exists and that, as of that post, no engine implements general expression evaluation, which is why the first version is an enum. An Iceberg commit dated 16 August 2026, `badca2d`, adds a Spark 4.1 rewrite that turns a restricted relation into a projection of masks over a filter, and it fails closed on an unrecognized action. Google's post on 1 October still says the specification change does not itself ship enforcement. I would not pin a runtime to that commit until a release note names it.

<img src="/assets/images/cross-engine-abac-three-doors.svg" alt="Three doors for a Unity Catalog policy: a serverless filter in front of an untrusted engine, Iceberg read restrictions enforced by a trusted engine, and catalog labels mapped by another catalog" title="Cross-engine ABAC returns sanitized rows. Read restrictions return obligations. Labels return metadata." style="width:100%; height:auto;">

## Labels move context, not a decision

Catalog labels are the third path in the 10 September post, and they are a field on the same `LoadTableResult` I read. `object-labels` is a flat key-value map for the table. `fields` carries the same kind of map per field id. The schema says labels are optional, a client may ignore them, and a client may cache them following the response ETag. The spec does not define how a catalog produces them, and it does not define what a key means.

The 10 September post is about the case the engine path does not cover. Two catalogs, each with its own users and its own policy language, federate. Calling the source catalog once per user so the destination can render a catalog explorer does not scale. Labels let the source hand over lightweight metadata, a column marked `pii=ssn` or a table tied to a domain, and the destination maps those pairs onto its own tags and enforces with its own engine. Enforcement stays local. Audit stays local. The meaning does not travel. The post says lineage for labels does not extend beyond the source catalog, and the destination receives the resolved key and value without being told whether the label was meant for access control, cost, or discovery. A label does not mask a column. A policy on the destination, written by someone who agreed what the key means, masks the column.

| Door | Who enforces | What crosses the wire | When I would use it |
| --- | --- | --- | --- |
| Cross-engine ABAC | Databricks serverless compute | Sanitized rows | An engine the user controls. Spark, as documented on 1 October. |
| Read restrictions | A trusted engine | Nine actions and a predicate, for this principal | Policies that reduce to that vocabulary, on an engine operators control. |
| Catalog labels | The destination catalog | Key-value metadata | Federation into a catalog that already has its own policies. |

## Where a derived table drops the policy

A policy matches tags. A new table does not inherit them. The tag-propagation page, updated 12 October, says so, and it marks the repair as Beta. A workspace admin turns the preview on. Propagation is then enabled per governed tag, it is off by default, and it follows lineage for `CREATE TABLE AS SELECT`, `CREATE VIEW`, `ALTER VIEW`, `CREATE OR REPLACE VIEW`, `INSERT INTO ... SELECT`, and `MERGE`. `CLONE` propagates table-level tags only. The operation has to run on serverless compute. Classic compute is not supported. Streaming tables and materialized views from Lakeflow Declarative Pipelines are not supported. Volumes are not supported.

The page's example is the one I would keep. A source column tagged `pii=ssn` is selected into a new table, and propagation applies `pii=ssn` to the matching column. An ABAC policy that matches the tag then applies to the derived column without anyone tagging it by hand. Propagation is asynchronous. Tags usually appear within minutes, and until the tag appears the policy is not enforced on the derived object. A conflict, two sources carrying the same key with different values, resolves to a default propagation value chosen when propagation was enabled. An object holds at most 50 tags, propagated tags included, and propagation stops at that limit. `ASSIGN` on the tag is not required for the user who runs the query, because propagation is a system operation. `MANAGE` on the tag is what turns it on.

I would not describe that Beta as part of cross-engine ABAC. It is how a tag, and therefore a policy that matches the tag, survives a copy made inside Databricks. A Spark job outside Databricks that writes a new table of its own is not on the supported-operation list. The mask on the source does not follow the files.

## Where it fits

I would use cross-engine ABAC when the table is already a Unity Catalog managed table, the reader is Apache Spark outside the workspace, and the requirement is that the raw column never arrives in that Spark process. A nightly extraction into a service the team runs. A notebook on a cluster the analyst controls. A job that must keep using Spark SQL and must not be handed a storage credential that can see the unmasked file. The policy stays the Unity Catalog policy: a governed tag, a mask or a row filter, an `EXCEPT` list for the principal that loads and transforms.

The property I want from that choice is narrow. The external process is untrusted, so the mask runs in the serverless layer, and the design says so.

I would reach for read restrictions only when three sentences are true at once. The policies in force reduce to the nine actions and a field-id predicate. The engine is one the operators control, and users do not hold the storage credential. Someone can name the Iceberg release that enforces the object and fails closed. Until that release is named, the spec is a contract to design toward, not a feature to turn on.

I would use labels when the destination is another catalog that already governs its own users. The work is a mapping of keys, reviewed like a schema, not a hope that the source policy will fire.

## Where it does not fit

I would leave cross-engine ABAC out of a streaming table or a materialized view. The catalog-commits page says catalog commits are not compatible with external data access on those objects. The cross-engine page requires both. I would also leave it out of a pipeline refresh that is supposed to store raw history. The requirements page says a pipeline evaluates ABAC as the owner or the run-as identity, and if that identity is subject to the policy the materialized view or streaming table permanently contains the masked or filtered data. The exempt principal belongs in `EXCEPT`. The page says to exempt only identities such as an ETL service principal, because an exemption sees the unmasked table.

I would leave it out of a service that reads and writes through one principal. The external page allows the read. The write requires an exemption, and an exemption is the unmasked table. Two principals, or the write moves somewhere the policy is not the thing being bypassed by accident.

I would leave it out when the mask is a `VARIANT` function, a function whose return type differs from the column, a filter on a `BINARY` column, or a projection of a `VARIANT` column. Those are the unsupported lines. I would leave a wide aggregation on this path until the job has been timed, because the page already says large aggregations might degrade, and the degradation is on compute the external cluster does not show in its own metrics. I would leave a self-join of a table that is being written during planning. The snapshot warning is a correctness bug waiting for a busy table, not a performance note.

I would leave read restrictions out of any design whose policy is the lookup-table function, a custom redaction, or a mask that keeps the last four characters by calling `right` rather than the spec's `show-last-4`. Those are different masks. I would leave them out of a laptop Spark process. The 10 September footnote is the reason.

I would leave labels out of a design that needs the source catalog to guarantee a mask. A client may ignore a label. The schema says so.

Two neighboring limits belong in the same risk log, because they look like this feature and are not. The requirements page says ABAC policies on a source table do not apply to an AI Search index built from that table. The index syncs the rows and does not enforce the filter or the mask when it serves. Deep and shallow clones of a table with ABAC policies are not supported unless the cloning principal is exempt. OpenSharing does not carry the source policy to the recipient. The share owner has to be exempt, and the recipient applies their own policies. ABAC on views is a separate Beta on that page, requiring the preview and Databricks Runtime 19 or above, and metric views are excluded. The cross-engine page says dynamic views are not supported. I would not collapse those two sentences into one.

## What goes in the design doc

**Name the door.** Cross-engine ABAC, read restrictions, or labels. The June post, the 10 September post, and the 1 October pages are all current enough to be quoted in a review. They answer different questions.

**Write the principal twice.** The reader, who has `EXTERNAL USE SCHEMA` and is subject to the policy. The writer, who is in `EXCEPT` and therefore sees raw rows. A PAT and an OAuth machine-to-machine token are not interchangeable once a policy inspects `request.is_on_behalf_of`.

**Pin the table shape.** Managed table, `catalogManaged` visible in `DESCRIBE DETAIL`, external data access on for the metastore. Not a streaming table or a materialized view. The alter that enables catalog commits on a busy table is an operation with a duration and a do-not-cancel warning.

**Pin the client from the page, not from the June list.** Spark, Delta-Spark 4.1 with the Unity Catalog connector 0.4 and `ServerSidePlanning.enabled`, or Iceberg-Spark 1.11 on Spark 4.0 against the Iceberg REST URI with caching off. Starburst and DuckDB stay out until a client page names them.

**Record the mask contract that will actually run.** On the external path, the function return type matches the column. `VARIANT` projections and `BINARY` filters stay out. A mask proved by a query from the external engine, not by a file that disappeared from a scan plan. Inside Databricks, the cast rules on the 9 October patterns page are a different contract.

**Put the bill on `EXTERNAL_COMPATIBILITY`.** Not on `FINE_GRAINED_ACCESS_CONTROL`. No rate is printed on the page I read. Records typically land within 12 hours. Large aggregations are a cost risk the page already flags.

**Keep the snapshot warning next to any self-join.** Concurrent writes during planning can make one statement see two snapshots.

**Separate the spec from the product.** Read restrictions fail closed, use field ids, and cannot carry a subquery. Trust is an operational fact about the engine, not a field in the JSON. Labels are metadata a destination catalog may map, and a client may ignore. Tag propagation, Beta as of 12 October, is how a governed tag survives a serverless `CREATE TABLE AS SELECT` inside Databricks. It is not how a mask survives an external copy.

Cross-engine ABAC is the Databricks architecture that puts the mask back in front of an engine that was never going to enforce it. The storage stays open. The raw row is not supposed to. The Iceberg spec, published into public view the same day, is the other honest design: a trusted engine enforces a small vocabulary, or it returns nothing. Labels are how a second catalog learns the classification without learning the policy. I would bet an external Spark read on the serverless door, with the exemption list and the usage line written down. I would not bet a custom UDF on the spec, and I would not bet a mask on a label.

## References

- Databricks, "October 2026" release notes, cross-engine ABAC generally available on 1 October 2026. [https://docs.databricks.com/aws/en/release-notes/product/2026/october](https://docs.databricks.com/aws/en/release-notes/product/2026/october)
- Databricks, "Cross-engine attribute-based access controls (ABAC)" (AWS), last updated 1 October 2026. [https://docs.databricks.com/aws/en/external-access/cross-engine-abac](https://docs.databricks.com/aws/en/external-access/cross-engine-abac)
- Databricks, "Cross-engine attribute-based access controls (ABAC)" (Google Cloud), last updated 1 October 2026. [https://docs.databricks.com/gcp/en/external-access/cross-engine-abac](https://docs.databricks.com/gcp/en/external-access/cross-engine-abac)
- Microsoft, "Cross-engine attribute-based access controls (ABAC)" (Azure Databricks), article date 1 October 2026. [https://learn.microsoft.com/en-us/azure/databricks/external-access/cross-engine-abac](https://learn.microsoft.com/en-us/azure/databricks/external-access/cross-engine-abac)
- Alex Jiang, Alex Reid, and Michelle Leon, "Introducing Cross-Engine ABAC," Databricks Blog, 2 June 2026. [https://www.databricks.com/blog/introducing-cross-engine-abac](https://www.databricks.com/blog/introducing-cross-engine-abac)
- Daniel Weeks, Ryan Blue, and Andrei Tserakhau, "Read Restrictions and Catalog Labels: Unifying governance across engines and catalogs," Databricks Blog, 10 September 2026. [https://www.databricks.com/blog/unifying-governance-across-engines-and-catalogs-open-lakehouse](https://www.databricks.com/blog/unifying-governance-across-engines-and-catalogs-open-lakehouse)
- Talat Uyarer and Sung Yun, "Standardizing fine-grained access control in Apache Iceberg's REST catalog," Google Open Source Blog, 1 October 2026. [https://opensource.googleblog.com/2026/10/standardizing-fine-grained-access-control-in-apache-icebergs-rest-catalog.html](https://opensource.googleblog.com/2026/10/standardizing-fine-grained-access-control-in-apache-icebergs-rest-catalog.html)
- Apache Iceberg, REST Catalog OpenAPI, `ReadRestrictions` and `Labels` on `LoadTableResult`. [https://github.com/apache/iceberg/blob/main/open-api/rest-catalog-open-api.yaml](https://github.com/apache/iceberg/blob/main/open-api/rest-catalog-open-api.yaml)
- Apache Iceberg, "REST Catalog Spec." [https://iceberg.apache.org/rest-catalog-spec/](https://iceberg.apache.org/rest-catalog-spec/)
- Apache Iceberg, "Spec: Add finer grained read restrictions as part of loadTable," pull request 13879. [https://github.com/apache/iceberg/pull/13879](https://github.com/apache/iceberg/pull/13879)
- Apache Iceberg, commit `badca2d`, "Spark: Enforce ReadRestrictions returned by loadTable," 16 August 2026. [https://github.com/apache/iceberg/commit/badca2d4553cd41108df65d56b02b17e56fc7e20](https://github.com/apache/iceberg/commit/badca2d4553cd41108df65d56b02b17e56fc7e20)
- Databricks, "Catalog commits" (AWS), last updated 11 September 2026. [https://docs.databricks.com/aws/en/tables/features/catalog-commits](https://docs.databricks.com/aws/en/tables/features/catalog-commits)
- Databricks, "Requirements, quotas, and limitations for row filter and column mask policies," last updated 9 October 2026. [https://docs.databricks.com/aws/en/data-governance/unity-catalog/abac/requirements](https://docs.databricks.com/aws/en/data-governance/unity-catalog/abac/requirements)
- Databricks, "Common patterns for row filtering and column masking," last updated 9 October 2026. [https://docs.databricks.com/aws/en/data-governance/unity-catalog/abac/common-patterns](https://docs.databricks.com/aws/en/data-governance/unity-catalog/abac/common-patterns)
- Databricks, "Tag propagation," last updated 12 October 2026. [https://docs.databricks.com/aws/en/admin/governed-tags/tag-propagation](https://docs.databricks.com/aws/en/admin/governed-tags/tag-propagation)
- Databricks, "Billable usage system table reference," last updated 8 October 2026. [https://docs.databricks.com/aws/en/admin/system-tables/billing](https://docs.databricks.com/aws/en/admin/system-tables/billing)
