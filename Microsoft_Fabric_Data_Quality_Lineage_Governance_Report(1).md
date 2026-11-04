# Microsoft Fabric: Data Quality, Lineage, and Governance Capabilities

**Detailed Fabric-first technical assessment | November 4, 2026**  
**Scope:** Microsoft Fabric is the primary platform. Azure and AWS products are included only as functional benchmarks for specific capabilities.  
**Citation format:** Numbered citations in the text point to complete, official documentation URLs in **References**, at the end of this report.  
**Rendering:** GitHub-flavored Markdown and local SVG diagrams. Keep this file beside the `diagrams/` directory.

## Executive Summary

Microsoft Fabric provides a meaningful **native foundation for trusted analytical data**, but **data quality, lineage, and governance are not implemented by one universal feature**. A practical design combines Fabric transformation-time checks, Delta Lake consistency, materialized lake view (MLV) constraints and quality reports, Fabric lineage and impact analysis, the OneLake catalog, OneLake security, and—where enterprise-wide profiling, scorecards, remediation, or compliance are required—Microsoft Purview capabilities. Additional Purview functionality can introduce separate licensing or usage charges. [1][4][5][11][19]

**Data quality:** Fabric can expose data issues during Power Query authoring, reject incompatible Delta writes, validate data in Spark notebooks, and apply `FAIL`/`DROP` constraints during MLV refresh. MLVs also provide native quality reports and refresh lineage. For cataloged quality rules, scheduled scans, numerical quality scores, scan history, alerts, and failed-record exports, Microsoft Purview Unified Catalog is the more complete Microsoft-governed workflow for supported Fabric Lakehouse assets. Power Query profiling alone should not be mistaken for a recurring production quality SLA. [4][5][6][8][9][11][13]

**Lineage:** Fabric workspace lineage primarily describes **item-to-item relationships**, with a one-step view of upstream dependencies outside the workspace. Impact analysis can identify affected downstream items beyond that immediate view. MLVs have a separate dependency/refresh lineage within their managed graph. A Fabric tenant scan into Purview is useful for estate inventory but does **not** automatically provide complete column-level, cross-workspace, or externally sourced lineage for every non-Power BI item. These boundaries matter when conducting schema-change impact analysis or compliance tracing. [7][20][21][22]

**Governance:** OneLake catalog provides discovery, governance insight, and security surfaces; Fabric supports domains, tags, endorsement, sensitivity labels, workspace/item controls, and detailed data access controls for supported items and engines. Microsoft Purview adds capabilities such as Data Map, data quality management, Data Loss Prevention (DLP), and broader compliance workflows. **Labeling, access control, DLP, and masking solve different problems** and must not be conflated. OneLake catalog Govern is undergoing regional rollout, and the maturity or availability of some APIs and policy actions varies. [1][2][3][24][25][27][29][31][32]

**Recommendation:** Use **Fabric-native controls at the point of ingestion/transformation**, **Purview for managed quality measurement and stewardship where justified**, **Fabric lineage plus documented augmentation for coverage gaps**, and **OneLake/SQL security plus Purview labeling and DLP for governance**. Make lineage coverage, scan permissions, rule thresholds, and failure response part of the production acceptance criteria—not just checkboxes in a catalog. The architecture below is a recommended design, not a statement that every depicted connection is automatically discovered by Fabric. [5][11][13][22][27]

### Capability ownership at a glance

| Requirement | Primary Fabric feature | Complementary Microsoft capability | Important boundary |
| --- | --- | --- | --- |
| Inspect column errors, nulls, distributions while authoring | Dataflow Gen2 / Power Query profiling | Purview profiling | Power Query preview is not an automatically persisted enterprise quality score. [8] |
| Protect writes against incompatible structure | Lakehouse Delta schema enforcement | Schema change review | Type/schema enforcement is not the same as business-rule accuracy. [9] |
| Enforce row conditions during managed transforms | Materialized lake views with quality constraints | Purview quality scoring | `DROP` can remove bad records; `FAIL` blocks a refresh. [5] |
| Validate arbitrary cross-table/business rules | Notebook/Spark or SQL procedures | External testing frameworks as selected | Custom rules, metrics persistence, and recovery need design. [5][10] |
| Measure cataloged quality over time | MLV quality report (MLVs only) | Purview Unified Catalog data quality | Purview requires registration, connection, roles, and usage billing. [6][11][19] |
| See workspace asset dependencies | Fabric lineage view and impact analysis | Purview Data Map | Coverage is not guaranteed at table/column level. [20][22] |
| Track declarative lakehouse refresh dependencies | MLV lineage | Custom run telemetry | Specialized MLV view is not a universal graph for all Fabric code. [7] |
| Discover and curate data | OneLake catalog, domains, tags, endorsements | Purview Unified Catalog data products | Fabric domains and Purview governance domains are related governance concepts but not interchangeable objects. [1][2][25][26] |
| Enforce access controls | Workspace/item access, OneLake security, SQL and semantic security | Microsoft Entra, Purview policies | Workspace edit roles and individual query engines have different effective permissions. [27][28][29][30] |
| Identify and address sensitive data | Sensitivity labels and supported DLP policies | Purview Information Protection / DLP | Item/engine support and label inheritance are scenario-specific. [31][32] |

## 1. How Fabric's quality, lineage, and governance layers fit together

![Fabric trusted-data architecture](diagrams/01_fabric_trusted_data_architecture.svg)

Fabric's components should be read as **complementary control layers**:

1. **Execution/data layer:** Data pipelines, Dataflows, Spark notebooks, MLVs, Warehouses, semantic models, and OneLake hold or transform data. Data quality may be enforced before promotion from bronze to silver or before publishing gold products.
2. **Metadata/lineage layer:** Workspace lineage, MLV dependency lineage, item details, and impact analysis show supported relationships. Coverage depends on item type and relationship creation method; some metadata may be missing unless separately recorded.
3. **Catalog/stewardship layer:** OneLake catalog provides discovery and governance information. Purview Data Map and Unified Catalog can inventory assets, attach business context, run supported quality scans, and provide quality scores and alerts.
4. **Security/compliance layer:** Entra authentication, workspace/item access, OneLake security and compute-engine permissions enforce access. Purview sensitivity labels, DLP, audit, and protection policies support classification and compliance workflows.

**Key architecture principle:** A metadata label indicating “Certified” does **not** prove that an ingest passed its latest quality checks, and a successful pipeline run does **not** prove that all downstream tables meet business requirements. Certification, quality results, and freshness must be evaluated independently. [11][15][26]

### 1.1 Distinguish native, integrated, and custom capabilities

| Classification | Meaning | Fabric examples | Engineering consequence |
| --- | --- | --- | --- |
| Native / operational | Runs inside the Fabric item or authoring environment | Delta schema checks; MLV constraints; workspace lineage; MLV quality reports; OneLake security | Closest to the execution path; often the best place to block bad data. |
| Integrated / separately governed | Microsoft service integrated with Fabric, but not identical to Fabric capacity | Purview Unified Catalog quality scans, DLP, data products, cross-system Data Map | Requires separate setup, permissions, and potentially billing. |
| Custom implementation | Logic you write or assemble using Fabric compute | Spark profiling, reconciliation, quarantine tables, contract verification, DQ run/audit tables | Flexible, but you own tests, drift handling, durability, and alerts. |
| Partial/preview | Documented with narrower scope or preview status | Purview Lakehouse sub-item metadata preview; Catalog Search API preview; certain protection actions | Verify actual tenant/region, licensing, item support, and SLA before production adoption. |

## 2. Data Quality in Microsoft Fabric

### 2.1 What “data quality” must actually measure

The six dimensions commonly referenced by Purview are **completeness, consistency, conformity, accuracy, freshness, and uniqueness**. Engineers should add use-case-specific reconciliation and referential integrity checks where needed. The table below contains **illustrative** thresholds; it is a recommended rule design, not a Microsoft default. [11][14]

| Dimension | Example Fabric rule | Typical implementation | Illustrative pass threshold |
| --- | --- | --- | --- |
| Completeness | `customer_id` cannot be NULL | MLV constraint; Spark SQL; Purview empty/blank rule | 100% on critical key |
| Uniqueness | One row per `order_id` | Spark group/count; Purview unique-values rule | No duplicate business keys |
| Conformity | `region` belongs to approved values | MLV Boolean check; custom Purview rule | 99.5% conforming |
| Validity/accuracy proxy | `amount >= 0` for payment records | MLV `CHECK`; warehouse SQL | 100% nonnegative where required |
| Referential integrity | `orders.customer_id` exists in `customers` | Spark join; Purview table lookup (supported scope) | 100% of required relationships |
| Freshness | Latest ingest timestamp not older than SLA | Notebook/SQL metric; Purview freshness rule | Delay < 60 minutes for hourly product |
| Reconciliation | Source vs curated row counts/totals balance | Pipeline/Notebook/SQL audit | 0 unexplained financial variance |
| Distribution/drift | Amount or category frequency changes abnormally | Spark statistical checks; ongoing profiling | Threshold based on history, not one fixed number |

**Quality design rule:** Define each check with an asset name, grain, owner, business rationale, evaluation interval, pass threshold, severity, failure action, and evidence location. A quality score without those fields is useful for observation but weak as an operational contract.

### 2.2 Dataflow Gen2: Power Query profiling and cleaning

In the Power Query authoring experience, data profiling provides **Column quality**, **Column distribution**, and **Column profile**. Column quality distinguishes valid, error, empty, unknown, and unexpected-error values; distribution shows distinct/unique-value patterns; profile exposes more detailed statistics. By default, Power Query profiling is based on **the first 1,000 rows** unless the user explicitly selects entire-dataset profiling. This is a development-time view and can be unrepresentative for a large or skewed dataset. [8]

**Best uses in Fabric:** inspect schema and parsing errors before a Dataflow Gen2 is published; correct type transformations; identify nulls and outlier patterns; and prototype validation logic before converting it into an enforced production check.

**Not sufficient by itself:** do not equate a green authoring preview with a warehouse-wide certification, a scheduled quality history, a failed-row quarantine, or an automated downstream stop. Implement explicit quality gates and persist results when the data is business critical. Dataflow transformations can be followed by a Notebook/MLV validation or an external Purview scan depending on the requirement.

### 2.3 Lakehouse Delta: schema correctness versus business correctness

Fabric Lakehouse commonly stores managed tables as Delta Lake. **Schema enforcement** checks that incoming writes match the table schema unless schema evolution is explicitly enabled. Delta transactions and controlled schema evolution improve structural reliability; they do **not** automatically detect a wrong customer ID, an invalid monetary amount, duplicate business keys, or a late-arriving partition. [9]

Recommended schema-contract practice:

- Treat type changes, renames, and drops as governed changes; test their effect on downstream notebooks, SQL analytics endpoints, semantic models, and reports.
- Prefer additive `mergeSchema` evolution only when the data contract permits it; do not enable unrestricted schema drift as the default.
- Validate dataset-level keys and cross-table relationships separately, ideally before the silver/gold promotion.
- Record input schema version and transformation release with quality-run evidence to make failures reproducible.

### 2.4 Materialized Lake Views: Fabric-native declarative enforcement

**Materialized Lake Views (MLVs)** persist transformations as Delta tables, track dependencies, refresh their materialized results, and support declarative data quality constraints. Supported violation handling includes `FAIL` (abort refresh; default) and `DROP` (discard violating rows and continue). If a view defines both actions, Microsoft's documentation says `FAIL` takes precedence. MLV lineage surfaces dropped-record counts, and an embedded quality report exposes recent violations and trends. [4][5][6][7]

Example: a silver-level order view that excludes null IDs and negative amounts. This is illustrative SQL based on the documented MLV constraint syntax; adapt names and business conditions before deployment.

```sql
CREATE OR REPLACE MATERIALIZED LAKE VIEW silver.valid_orders
(
  CONSTRAINT valid_order CHECK (
    order_id IS NOT NULL AND amount >= 0
  ) ON MISMATCH DROP
)
AS
SELECT order_id, customer_id, amount, order_timestamp
FROM bronze.orders;
```

**Use `DROP`** when the business explicitly permits exclusion and you have a strategy to investigate lost rows. **Use `FAIL`** for high-integrity data where publication of an incomplete dataset would be worse than a delayed refresh. For critical datasets, consider an explicit rejected-row capture path rather than relying solely on aggregate drop counts. Keep rule definitions under version control and test them with bad-row fixtures. [5]

MLVs have important deployment nuances: SQL definition is documented; **PySpark-authored MLVs are marked preview** in the current overview, and the documentation states they currently perform full refreshes. Not every custom UDF behaves identically in a managed lineage refresh; prefer built-in SQL functions for portable and predictable checks. [4][5]

#### The native MLV quality report

From the Lakehouse, open **Manage materialized lake views → Data quality report → Generate report**. Fabric creates a semantic model and report, providing an Overview and an MLV Detail page. The Overview highlights recent violations and drops; the detail view supports narrower investigation. Documentation also describes alerts on report measures through Activator, subject to the supported report setup. The report applies to **MLV runs**, not every table or arbitrary notebook in the tenant. [6]

### 2.5 Notebooks and SQL: an extensible validation gate

For batch reconciliation, joins, duplicate-key detection, and custom statistical validation, a Fabric Notebook provides more flexibility than row-wise MLV checks. A production pattern should separate the **metrics computation** from the **gate decision**, then write both to a durable audit table.

Illustrative PySpark validation (adapt table names and column types). This checks one run of a prepared silver dataset and raises an exception if the example contract fails:

```python
from pyspark.sql import functions as F

source_table = "silver.orders_candidate"
df = spark.table(source_table)

row_count = df.count()
null_ids = df.filter(F.col("order_id").isNull()).count()
duplicate_keys = (
    df.groupBy("order_id").count()
      .filter(F.col("count") > 1).count()
)
negative_amounts = df.filter(F.col("amount") < 0).count()

# Persist these metrics, a UTC run timestamp, run ID, source table,
# rule version, and failure reason to your own audit Delta table.
# Evaluate the rules before publishing into the consumer-facing table.

if null_ids or duplicate_keys or negative_amounts:
    raise ValueError(
        f"DQ gate failed: null_ids={null_ids}, "
        f"duplicate_keys={duplicate_keys}, "
        f"negative_amounts={negative_amounts}"
    )
```

**Important implementation distinction:** `duplicate_keys` above counts **duplicate key groups**, not the number of duplicate rows; choose and document the metric you need. Multiple actions trigger Spark jobs and incur compute; for large tables, aggregate checks efficiently and favor incremental partitions where appropriate. Persisting the audit record and wiring a failed notebook to block a downstream publish are design responsibilities, not implicit results of the snippet.

For Warehouse constraints, **PRIMARY KEY, UNIQUE, and FOREIGN KEY are documented as `NOT ENFORCED`** (with additional syntax requirements), so declaring them does **not** replace operational uniqueness or relationship checks. This is a material gap for workloads migrating from relational databases that enforce these constraints on write. [10]

### 2.6 Microsoft Purview Unified Catalog: enterprise quality assessment for supported Fabric assets

Purview's quality lifecycle is more than an inspection tab. The documented path is:

1. Assign the **data quality steward** role and determine governance-domain/data-product ownership.
2. Register the Fabric tenant as a source, configure a **Data Map scan**, and confirm the Fabric assets are discovered.
3. Associate supported Lakehouse table assets with a **Purview data product**.
4. Establish a data-quality connection to the data source with the appropriate identity and data access.
5. Run profiling to discover column distributions, null patterns, and other statistics.
6. Create built-in or custom rules; set expectations and thresholds.
7. Run scheduled or manual quality scans, examine scores/trends and failed records, assign remediation actions, and configure alerts.

The fabric-specific setup instructions distinguish metadata-scanning authentication from quality-scanning access: Fabric Data Map setup uses service-principal configuration, while the documented quality connection uses **Purview managed identity** and requires access to the Fabric workspace. Do not assume that permission to discover metadata also grants permission to read protected data for a quality scan. For the Fabric Lakehouse setup described in Microsoft's guide, the Purview managed identity is granted workspace Contributor permission—a security consideration to review with your platform team. [11][13][35]

Purview's documented built-in rule choices include **freshness, uniqueness, format, data-type matching, duplicate rows, blank fields, table lookup, and custom expressions**. Supported quality dimensions extend beyond simple null checks. Purview can recommend rules, retain run results, score at multiple governance levels, and alert recipients when scores fall below thresholds or deteriorate. Its configured quality scanning works on documented supported assets and formats; it is **not** a guarantee that every Fabric item type, custom computation, or nested data structure can be assessed. [11][12][14][16][17]

#### Purview quality score semantics

Microsoft documents the quality score for an asset as the **arithmetic average of the scores of its configured rules**; data-product scores average asset scores; governance-domain scores average product scores. This means the aggregate may not reflect the risk of one critical failed rule unless you explicitly retain critical-rule gates. The following example is a *hypothetical illustration*, not observed Fabric data. [15]

| Rule | Example score | Operational criticality |
| --- | ---: | --- |
| Required transaction key | 100% | Critical |
| Amount range | 100% | Critical |
| Email format | 60% | Medium |
| Description completeness | 100% | Low |
| **Unweighted arithmetic average** | **90%** | **A passing overall score can hide a severe rule-specific issue in another scenario.** |

Adopt both an **aggregate-health objective** and **hard stop rules for critical fields**; do not promote a certified data product solely because its mean score clears a generic threshold.

#### Alerts and failed-record investigation

- Purview's alert configurations include **score below a threshold**, **score decrease**, and **failed-scan notifications**, with supported email recipients. Rule/asset threshold settings can also trigger notifications. [16][17]
- Supported quality scans can publish failed-row information to an organization-controlled storage location. Govern who can read those records because they may contain sensitive source values. Microsoft's documented per-rule failed-row export limit is **100,000 records per run**. [18]
- Purview quality scanning supports a documented maximum of **200 active rules per data asset per scan**. The platform may allow more defined rules, but exceeding the active limit causes scans to fail. [14]
- Sources and formats must be verified against the support matrix; Fabric support is specifically described for Lakehouse scenarios, with supported Delta/Parquet/Iceberg patterns and caveats for shortcuts/mirroring. [12][13][41]

### 2.7 Suggested quality-control placement

| Layer | Check type | Failure action | Evidence to retain |
| --- | --- | --- | --- |
| Bronze / ingestion | Source availability, file/schema sanity, row-count baseline | Hold ingest or quarantine bad files | Arrival time, file ID, schema hash, ingest ID |
| Bronze → Silver | Required keys, valid types, permitted ranges, deduplication | `FAIL` or explicit reject path; use `DROP` only with approval | Rule versions, rejects, quality metrics |
| Silver → Gold | Reconciliation, joins, referential checks, business totals | Block consumer-facing publish | Source/target counts, variances, run ID |
| Consumer data product | Freshness, completeness, critical metrics, certification checks | Alert steward; unpublish/deprecate or warn per contract | Product score and critical-rule state |
| Governance review | Classification, ownership, retention, scan coverage | Queue remediation and policy exception | Catalog metadata, policy version, reviewer decision |

![Fabric data quality gates](diagrams/02_fabric_data_quality_gates.svg)

## 3. Data Lineage in Microsoft Fabric

### 3.1 Workspace lineage: the built-in item relationship view

Every Fabric workspace has a lineage view that displays the items within that workspace and their supported relationships. External **upstream** dependencies appear one step away; the canvas does not simply show every downstream item in unrelated workspaces. Viewers can see the workspace lineage, but the Viewer role does not expose data-source cards. Relationships may depend on the way an item was connected—for example, certain manually created Power Query/Mashup connections are not guaranteed to appear as expected. [20]

Lineage is especially valuable for:

- Identifying which pipelines, dataflows, lakehouses, semantic models, and reports depend on an item.
- Evaluating proposed schema changes and whether a downstream report or model might break.
- Finding the upstream item responsible for a stale analytical result.
- Supporting documentation and operational change review.

**Do not infer column-level transformation provenance from a workspace item edge.** An edge such as `Notebook → Lakehouse` does not by itself explain which source columns populated a particular output field or what transformation expression was applied.

### 3.2 Impact analysis: the downstream question

Fabric **impact analysis** is the companion to lineage view. For supported Fabric items, it can show direct children or all affected downstream items/workspaces, and offers a way to notify contacts about an intended change. For external data sources, the documented view is limited to their direct children unless you continue analysis from a downstream item. It should be consulted before deleting, renaming, changing schemas, or modifying refresh contracts. [21]

Practical change process:

1. Identify the source item and the proposed schema/permission change.
2. Open lineage and run impact analysis to find affected consumers.
3. Review code-level dependencies that the visual graph may not capture, especially custom Spark SQL, APIs, shortcuts, or manually maintained datasets.
4. Test representative downstream computations and reports in a nonproduction workspace.
5. Record approval, dependency owners, and the release identifier in your change ticket.

### 3.3 Materialized Lake View lineage: a distinct, more specialized graph

MLV lineage is a **managed dependency/refresh graph**: it represents upstream tables/views and the ordering needed to refresh dependent MLVs. It can include dependencies across Lakehouses for supported MLV scenarios, and the interface exposes run and quality signals such as violated constraints and dropped rows. For file-backed MLVs, source folder/file nodes can appear, with per-run files processed and rows added; a projected `__filepath__` column can help retain row-to-source-file traceability. Shortcuts are treated as source entities in the documented MLV lineage view. [4][7]

This graph is stronger for its specific workload than an informal notebook diagram, but it does **not** imply that arbitrary pipelines, UDFs, SQL scripts, and exports have full row/column lineage automatically. Treat it as a specialized component of the larger governance architecture.

### 3.4 Microsoft Purview lineage: estate inventory, not universal column lineage

After registration and scanning, Microsoft Purview can inventory many Fabric items and their supported item-level relationships. Microsoft's Fabric-to-Purview documentation states that **non-Power BI Fabric artifacts generally expose item-level metadata and lineage**, while **Lakehouse table/file sub-item metadata scanning is preview and sub-item lineage is not supported**. The same documentation flags **unsupported cross-workspace lineage for non-Power BI assets**, non-Power BI external upstream sources, and notebook-to-pipeline lineage gaps. These are documented limits of that Purview capture path, not a claim that all Fabric interfaces lack every cross-workspace view. [22]

![Lineage surfaces and coverage](diagrams/03_lineage_coverage.svg)

### 3.5 Coverage and mitigation matrix

| Question | Fabric workspace lineage | MLV lineage | Purview Fabric scan | Recommended supplemental evidence |
| --- | --- | --- | --- | --- |
| Which items are linked in my workspace? | Yes, supported item relationships | Only MLV DAG subset | Scanned item metadata/lineage | Versioned inventory |
| What is directly upstream outside my workspace? | One step shown | Supported MLV sources / shortcut nodes | Limited for non-Power BI items | Explicit source registry and contracts |
| What is affected downstream across workspaces? | Use impact analysis | MLV graph can show supported dependencies | Non-Power BI cross-workspace lineage limited | Change approval plus dependency ownership |
| Which column produced `gross_margin`? | Not a full column transformation trace | Depends on MLV definition; not universal column governance lineage | Not covered by non-Power BI sub-item lineage | SQL definitions, code review, column mapping |
| Why did 400 rows disappear? | No general row-level explanation | DROP/violation counts for MLVs | Quality error records for supported scans | Rejected-row table, run ID, file ID |
| Which user edited/exposed an item? | Not the primary audit interface | Not the primary audit interface | Other Purview/audit features | Fabric/Purview audit logs and ticket records |

### 3.6 Recommended enterprise lineage record

For critical transformations not fully covered automatically, maintain a **versioned lineage register** with: `source_asset`, `source_field` (where meaningful), `target_asset`, `target_field`, `transformation_id`, `code_commit`, `workspace`, `run_id`, `owner`, `effective_from`, and `last_verified_at`. Prefer generated metadata from deployed SQL/notebook definitions and runtime logs when reliable; flag mappings as `manual`, `inferred`, or `verified`. This is a recommended engineering compensating control, not an out-of-the-box Fabric feature.

## 4. Governance Capabilities in Microsoft Fabric

### 4.1 OneLake catalog: discovery, Govern, and Secure

The **OneLake catalog** is Fabric's central place to discover and understand items and governance status. Microsoft documents distinct **Explore, Govern, and Secure** experiences. Govern can surface tenant/domain/workspace inventory, label coverage, DLP evaluations, endorsement and description completeness, and recommendations; governance insights are refreshed from supporting administrative metadata rather than reflecting every data change instantaneously. The Govern experience is **rolling out by region**; the Admin portal remains an alternative where rollout is incomplete. [2][3]

A useful design distinguishes these four tasks:

| Governance task | Fabric capability | Practical standard |
| --- | --- | --- |
| Ownership boundary | Domains/subdomains and workspace roles | Each production item belongs to a named business domain and owning team |
| Findability | Catalog metadata, descriptions, tags | Required business description, owner, classification, update frequency |
| Trust | Promotion, certification, master-data endorsement | Certification is reviewed and revocable; do not automatically equate it to passing a scan |
| Security posture | Secure experience, OneLake and item permissions | Least-privilege groups, approved exceptions, periodic access reviews |

### 4.2 Domains, tags, and endorsement

**Domains** group related Fabric content by organizational subject area and can allow delegated governance settings. **Tags** are flexible catalog metadata: administrators can define tenant- or domain-level tags, and authorized users apply tags to workspaces/items. Microsoft documents a **maximum of 10 tags per item or workspace**, and tags can surface in search, item details, and lineage views. **Endorsements** are separate trust signals: *Promoted* is owner-driven; *Certified* requires an authorized reviewer; *Master data* marks a core authoritative item. [1][25][26]

Recommended minimal metadata contract for every production Lakehouse/Warehouse/semantic model:

- Business domain and owning group; named operational contact.
- Short description, data product purpose, grain, retention class, and refresh expectation.
- Sensitivity classification and access model.
- Data quality rule set/critical threshold and evidence dashboard.
- Upstream contract, most important downstream consumers, and change-notification process.
- Certification state, approver, review date, and deprecation policy.

**Fabric domains versus Purview governance domains:** these serve related grouping and delegated ownership purposes but are **not the same automatically synchronized entity**. Purview's quality workflow also introduces *data products* and *data assets*. Plan the mapping between your Fabric workspace/domain structure and Purview product ownership rather than assuming identical hierarchies. [1][11][13]

### 4.3 Security architecture: authentication, control plane, data plane, and engines

Fabric security is layered:

1. **Entra identity/authentication** authenticates users and service principals.
2. **Control-plane permissions** determine what people can create, administer, share, or modify via Fabric workspace roles and item permissions.
3. **OneLake data-plane security** scopes access to supported items, folders, tables, schemas, rows, and columns.
4. **Compute-engine policies** can add SQL endpoint/Warehouse permissions and semantic-model rules; those policies apply to their respective query engines rather than automatically to every other read path. [27][28][29][30]

**Security gotchas:**

- **Workspace Admin, Member, and Contributor permissions are privileged** and can bypass restrictions designed for lower-privilege data readers. Don't assume that assigning a broad workspace role while adding an RLS filter yields a least-privilege outcome. [27][29]
- OneLake security uses **grant semantics**: access granted through another role/path may still be effective; design group membership carefully. Row- and column-level security work for supported table formats and authorized engines, while unsupported engines can be blocked rather than given a filtered view. [28][29]
- OneLake **RLS/CLS behavior can differ at the presentation layer**: for instance, an unauthorized-column `SELECT *` may behave differently in Spark and the SQL analytics endpoint even while protected values remain unavailable. Test every actual consumption path. [29]
- Warehouse SQL security, semantic-model RLS, OneLake security, and DLP policies are not substitutable. Test independently from Spark, SQL endpoint, Direct Lake/semantic model, and any external engine in scope. [27][30][34]

![Governance access and policy layers](diagrams/04_governance_security_layers.svg)

### 4.4 Sensitivity labels and DLP: classification is not access enforcement by itself

Microsoft Purview Information Protection sensitivity labels can be applied to Fabric items, with default, programmatic, and some automatic inheritance support. However, inheritance and mandatory-labeling behavior are **not uniform across Fabric item types and relationships**. For example, Microsoft documents that inheritance **from labeled data sources** is currently supported for Power BI semantic models, not all Fabric ingest items, and label propagation between Power BI and non-Power BI items has directional constraints. Do not assume that every exported file inherits a Fabric item label. [31]

**Purview DLP** policies examine **supported** Fabric items and can notify users, generate alerts, and—under a documented **preview** action—restrict access when policy conditions are met. Supported types include semantic models, lakehouses, warehouses, KQL databases, mirrored databases, and SQL databases. DLP requires appropriate Purview licensing. A DLP policy does not guarantee a scan of unsupported data paths or replace row-level access design. [1][32]

**Protection policies** are a further capability that can enforce item-level access based on configured sensitivity labels for supported Fabric items. Documented limits include a policy-count cap and a warning that Git integration and deployment pipelines use workspace permissions rather than protection-policy item permissions. Preview rollout and exceptions require testing. [33]

**Dynamic data masking (DDM)** in Warehouse/SQL endpoint can obscure query results for nonprivileged users, but Microsoft explicitly warns that DDM is **not sufficient protection against deliberate inference** through queries. Treat it as an exposure-reduction aid, not as a replacement for RLS/CLS, least-privilege grants, or secure curated views. [34]

### 4.5 Metadata collection, APIs, and audits

Fabric administrative **metadata scanning** supports organization-wide discovery through authorized administrative APIs. Enabling enhanced admin API responses can include detailed Power BI semantic model tables, columns, and measures, as well as DAX/mashup expressions under the appropriate tenant settings. That metadata access may be broad; enabling the service principal for read-only admin APIs should be a deliberate security decision. [23][35]

The **OneLake Catalog Search REST API** enables programmatic discovery of accessible catalog entries, but is marked **Preview** in the documentation. Search is permission-filtered; discovering metadata is not equivalent to being authorized to read the underlying records. Plan for API compatibility checks rather than using preview response formats as an untested enterprise contract. [24]

For auditability, capture the appropriate Fabric/Purview activity logs and access reviews separately from run-quality metrics. A quality scan tells you whether records meet a rule; lineage tells you a subset of their dependencies; an audit event is what helps answer **who did what and when**. These records should have access controls and retention that align to your organization's policy. [1][27][32]

### 4.6 Governance responsibility matrix

| Function | Suggested accountable role | Minimum operational obligation |
| --- | --- | --- |
| Tenant-wide controls and capacity policy | Fabric administrator / platform owner | Tenant settings, deployment guardrails, emergency permissions |
| Business definitions and data product suitability | Domain owner / data product owner | Ownership, business terms, criticality, consumer expectations |
| Data validation rules and triage | Data quality steward + engineering owner | Rule approvals, thresholds, failed-run and false-positive review |
| Catalog metadata and certification | Steward / authorized certifier | Descriptions, ownership, endorsement reviews |
| Technical lineage and change review | Data engineer / solution architect | Source mapping, code references, impact analysis, schema versioning |
| Classification and DLP | Information security / compliance | Sensitivity taxonomy, policy exceptions, audit and retention |
| Consumer read permissions | Workspace/item owner with security approval | Approved groups, RLS/CLS validation, quarterly access checks |

The roles in this matrix represent an **operating model recommendation**, not a claim that each is a literal Fabric RBAC role.

## 5. What Microsoft Fabric Does Not Automatically Solve

The following is a **gap analysis**, not a claim that Fabric lacks all mechanisms to address each concern.

| Requirement | Native capability | Main gap or qualification | Practical mitigation |
| --- | --- | --- | --- |
| A single data quality score for all Fabric items | MLV-specific reports; Purview for supported scanned assets | No universal, free, automatically generated score for every notebook, table, and Warehouse object | Federate custom metrics with Purview/MLV evidence |
| Hard business constraints on Warehouse keys | Key metadata declarations | PK, FK, UNIQUE are NOT ENFORCED | Enforce in ETL/SQL test gate and block publish |
| Column-level provenance for every workload | Some specialized metadata / MLV code | Purview non-Power BI sub-item lineage is not generally supported | Maintain tested column mappings and source IDs |
| Complete estate lineage across all workspaces and external systems | Workspace/impact views, Purview scans | Coverage varies, particularly non-Power BI cross-workspace/external | Source registry, manual edge reconciliation, audit jobs |
| Instant view of governance health | Catalog Govern insights | Administrative insights may be refreshed periodically and regionally rolled out | Operational checks + governance reporting |
| Consistent label behavior through every engine/export | Sensitivity labels | Scenario-specific propagation/export limits | Classification tests for actual routes |
| DLP uniformly scans every stored object | DLP-supported Fabric item types | Limited item types and action licensing/preview status | Scope inventory, independent storage policies |
| Effective RLS for privileged workspace contributors | OneLake RLS/CLS for eligible readers | Broad workspace roles grant data access | Separate development/admin roles from consumer roles |
| Auditable exception management | Quality records and policy metadata | Approval workflow not automatic for every rule and transformation | Ticketing, evidence retention, remediations |

## 6. Focused Comparisons: Fabric vs. Azure and AWS

Azure and AWS are included **only to benchmark specific Fabric decisions**. Their architectures are not introduced independently, and no single service is treated as a one-to-one substitute for the entire Fabric + Purview governance stack.

| Capability | Microsoft Fabric position | Relevant Azure benchmark | Relevant AWS benchmark | Assessment for Fabric users |
| --- | --- | --- | --- | --- |
| Visual exploratory profiling | Power Query profiling in Dataflow Gen2; Purview scans for managed measures | Azure Databricks Unity Catalog profiling and monitoring | AWS Glue Data Quality profiling | Native Fabric authoring is convenient; Purview is the broader governed scan layer. [8][11][36][40] |
| Declarative pipeline quality | MLV row constraints `FAIL`/`DROP`, report; custom Notebook gates | Databricks Unity Catalog + managed quality capabilities | AWS Glue Data Quality rules within ETL | Choose MLV for Fabric-centric declarative medallion paths; choose custom rules for unusual checks. [5][36][40] |
| Catalog quality scoring | Purview Unified Catalog DQ scores, alerts, steward workflows | Azure Databricks quality monitoring; Purview also usable in Azure | AWS Glue DQ scores and EventBridge integration | Purview is integrated but has separate authorization/billing. [11][15][16][36] |
| Data lineage granularity | Fabric workspace items + MLV graph; Purview scan limits for non-Power BI | Unity Catalog auto lineage can reach column level across attached workspaces (with exceptions) | Amazon DataZone lineage is OpenLineage-compatible and supports API ingestion | For pervasive field-level tracing, Fabric's default lineage may need supplementary metadata. [20][22][37][39] |
| Data catalog and discovery | OneLake catalog, domains, tags, endorsements; Purview inventory | Unity Catalog assets and governed tags | Glue Data Catalog / Amazon DataZone | Fabric is convenient for its own workspaces; verify multi-platform discovery need. [2][25][26][38][40] |
| Fine-grained lake access | OneLake security for supported items and engines | Unity Catalog row filters, column masks, grants | Lake Formation table, column, row, and cell policies | Test enforcement against each Fabric query engine and avoid broad workspace write roles. [27][29][38][40] |
| Sensitive information handling | Purview sensitivity labels, DLP and protection policies | Purview policies can participate in Microsoft estate | Lake Formation/IAM plus other security services | Fabric protection requires item-specific tests and Purview licensing where applicable. [31][32][38] |
| Governance cost profile | Fabric capacity for execution; Purview DGPU and eligible compliance licenses | Azure compute/service/Databricks usage + governance costs | Glue DQ / Lake Formation / DataZone service meters | Do not compare only Fabric CU price with a multi-service competitor total. [19][36] |

**Interpretation:** For organizations already using Fabric Lakehouses, Dataflow Gen2, Warehouses, and Power BI, the primary opportunity is to **formalize the Fabric quality and governance operating model** before adding external orchestration or catalog complexity. A different service is justified when a specific gap—such as deep cross-platform/column-level lineage, independent catalog governance, or specialized quality automation—has a measurable requirement, not merely because another platform offers more individual features.

## 7. Production Reference Design and Implementation Playbook

### 7.1 Data-product acceptance criteria

Define the following as an explicit release requirement for every critical gold table or semantic model:

- **Contract:** dataset grain, key fields, allowed schema evolution, owner, consumers, SLA, and classification are recorded.
- **Validation:** a published set of critical, warning, and informational rules is reproducible; failed checks block publication where required.
- **Evidence:** each production run writes a durable record of evaluated rows, failure counts, threshold results, data version, and pipeline/notebook/MLV run identifier.
- **Lineage:** upstream owners and downstream dependents have been checked with Fabric impact analysis, with any manual mappings flagged.
- **Access:** least-privilege read groups and OneLake/SQL/semantic security have been tested under actual user roles.
- **Governance:** correct description, tags, classification, endorsement state, and Purview stewardship are present when applicable.
- **Operations:** alert recipient, escalation, reprocessing procedure, and expired-data response are known.

### 7.2 A realistic Fabric-first implementation path

**Phase A — Inventory and scope.** Identify 2–3 critical Lakehouse/Warehouse data products, choose owners, inspect lineage, and document current dataflow/notebook transformations. Create a minimum catalog metadata standard and identify sensitive source attributes.

**Phase B — Enforce correctness before promotion.** Apply Delta schema contracts, MLV `FAIL`/`DROP` checks where suited, or Notebook/SQL validations for keys, foreign-key relationships, reconciliation, and freshness. Persist quality-run outcomes. Introduce a durable reject path for invalid records where the business requires complete accounting.

**Phase C — Add scorecards and stewardship.** Pilot Purview quality scans on **supported Lakehouse tables**: register/scan source, create quality connection, map assets to products, profile, create rules, and configure thresholds and alerts. Track critical-rule results separately from the mean quality score. Estimate Purview DGPU consumption from observed jobs before scaling. [11][13][15][19]

**Phase D — Strengthen governance.** Establish domains, tags, endorsement review, OneLake/SQL permissions, sensitivity labels, and appropriate DLP policy scope. Test the actual data read paths and external exports. Resolve unexpected broad workspace roles and inherited permissions. [25][27][29][31][32]

**Phase E — Improve lineage and automate assurance.** Use impact analysis in change reviews, augment missing lineage with code-derived mappings, add Git/deployment gates for schema and critical tests, and schedule periodic checks of audit logs, catalog metadata completeness, and exception closure.

### 7.3 Failure lifecycle example

A new order extract arrives with an unexpected negative amount and duplicate `order_id`.

1. **Detect:** ingestion records file/version; the MLV or Notebook gate identifies violations. If the MLV uses `FAIL`, the silver refresh stops; if `DROP` is chosen, bad rows are excluded with visible drop metrics. [5]
2. **Contain:** consumer-facing gold is not published until the critical contract passes; bad rows are written to a restricted quarantine table where configured.
3. **Trace:** use workspace lineage plus MLV lineage and impact analysis to find affected consumers; examine the source file ID and relevant transformation release. [7][20][21]
4. **Notify:** operational failure and a Purview quality-alert path (if enabled) reach the proper owner. Alerts should identify the asset, rule, run ID, severity, and remediation link. [6][16]
5. **Repair and replay:** upstream data is corrected, the transformation is rerun with an idempotent policy, quality checks are reevaluated, and closure evidence is retained.
6. **Govern:** if a classification or contract change contributed, update metadata and approve a corrected rule/version; do not silently lower the threshold simply to make the dashboard green.

### 7.4 Quality metrics and ownership dashboard

| Metric | Definition | Operational use |
| --- | --- | --- |
| Critical-rule pass rate | Passed critical evaluations / total critical evaluations | Release gate effectiveness |
| Data quality coverage | Production assets with current rule set / in-scope production assets | Governance adoption, not quality itself |
| Freshness compliance | Runs within agreed SLA / eligible runs | Consumer timeliness |
| Rejected-record rate | Rejected records / evaluated input records | Ingest health and upstream trends |
| Duplicate key group count | Number of business keys appearing more than once | Integrity, distinct from duplicate-row count |
| Lineage completeness (managed) | Confirmed required upstream/downstream edges / expected edges | Visibility gap measurement; expected edges require an inventory |
| Sensitive-label coverage | Correctly classified assets / in-scope assets | Classification completeness |
| Remediation lead time | Resolution timestamp minus detection timestamp | Incident response effectiveness |

Store denominators and scope alongside metrics to avoid misleading aggregate trends. These are **recommended KPIs**, not built-in Fabric dashboard fields or Microsoft-provided targets.

## 8. Security, Licensing, and Operational Considerations

**Fabric capacity:** SQL, MLV, Dataflow, and Spark execution use Fabric workload capacity and can compete for shared compute. Designing a large quality scan directly in custom Fabric notebooks has a different cost/operational profile than using Purview's independently billed quality service. [4][19]

**Purview billing:** Microsoft Purview Unified Catalog data quality is billed through a **Data Governance Processing Unit (DGPU)** consumption model. Data health management/quality functions and some protection/compliance products may involve additional charges or licensing. Obtain current region-specific rates from your Azure commercial agreement or pricing page, and size a pilot from measured scan frequency, assets, data volume, and data complexity rather than inserting a universal estimate. [1][19]

**Purview identity permissions:** Metadata discovery, quality computation, and end-user consumption are separate trust boundaries. Microsoft's Fabric-Purview setup documentation specifically warns that broad metadata extraction through read-only admin APIs may expose catalog metadata to the Purview account's own permission model after extraction. Document the administrative access approval and review scanned sensitive metadata. [13][23][35]

**Deployment and rollout:** Track Fabric region availability, preview flags, Spark/MLV authoring choices, table formats, DLP-supported item types, and source access modes. Revalidate them before a production launch; documentation may change independently of tenant enablement.

**Retention:** Choose retention for audit, scan history, quality metrics, failed rows, and lineage evidence as a compliance decision. Do not assume an automatically generated quality report or catalog scan is a permanent, immutable legal audit ledger.

## 9. Prioritized Recommendations

1. **Enforce quality nearest to transformation.** Prefer MLV constraints for compatible declarative silver/gold processing, backed by Notebook/SQL checks for multi-table/business reconciliation. Protect against the `NOT ENFORCED` Warehouse key constraint gap. [5][10]
2. **Separate observation from certification.** Use Power Query profiling for development insight; use durable quality gates and, when justified, Purview rule sets and quality scans for recurring measurement. [8][11]
3. **Promote complete quality evidence, not just a green mean score.** Critical fields should be explicit pass/fail conditions even when the Purview arithmetic mean exceeds a threshold. [15]
4. **Accept lineage only at its documented granularity.** Combine Fabric lineage, impact analysis, and specialized MLV lineage; record cross-workspace, column, and external-source mappings where the automatic graph is incomplete. [7][20][21][22]
5. **Make OneLake catalog the Fabric entry point for ownership and discovery.** Use domains, metadata tags, and endorsement processes, while acknowledging the Govern rollout and differences between Fabric and Purview business-domain structures. [2][3][25][26]
6. **Design security per access path.** Apply least privilege across workspace, OneLake, Warehouse SQL, and semantic-model layers; independently verify sensitivity-label and DLP behavior for each relevant item/export route. [27][29][30][31][32]
7. **Pilot Purview cost and permissions deliberately.** Start on supported high-value Lakehouse datasets, measure DGPU usage, review the managed identity permissions, and avoid uncontrolled expansion to every available table. [11][13][19]

### Bottom line

Microsoft Fabric has credible native building blocks for data quality, lineage, and governance, **especially around Delta Lake, materialized lake views, OneLake catalog, and OneLake security**. The strongest production pattern is neither “Fabric handles everything automatically” nor “Fabric needs a completely separate governance stack.” It is **Fabric-first enforcement and discovery**, extended with **Purview quality and compliance capabilities where their scope, authorization, and extra cost are justified**, plus targeted manual/automated metadata for the documented lineage gaps.

---

## References — Official Documentation URLs

All source URLs are consolidated here. Bracketed numbers throughout the report refer to these entries. Feature support and availability were evaluated against documentation accessible on **November 4, 2026**; recheck preview and rollout status before deployment.

### Microsoft Fabric: quality, lineage, and governance

[1] Microsoft — Governance overview and guidance: https://learn.microsoft.com/en-us/fabric/governance/governance-compliance-overview

[2] Microsoft — OneLake catalog overview: https://learn.microsoft.com/en-us/fabric/governance/onelake-catalog-overview

[3] Microsoft — Govern and manage data with OneLake catalog: https://learn.microsoft.com/en-us/fabric/governance/onelake-catalog-govern

[4] Microsoft — Materialized lake views overview: https://learn.microsoft.com/en-us/fabric/data-engineering/materialized-lake-views/overview-materialized-lake-view

[5] Microsoft — Data quality in materialized lake views: https://learn.microsoft.com/en-us/fabric/data-engineering/materialized-lake-views/data-quality

[6] Microsoft — Materialized lake view data quality report: https://learn.microsoft.com/en-us/fabric/data-engineering/materialized-lake-views/data-quality-reports

[7] Microsoft — Manage materialized lake views lineage: https://learn.microsoft.com/en-us/fabric/data-engineering/materialized-lake-views/view-lineage

[8] Microsoft — Power Query data profiling tools: https://learn.microsoft.com/en-us/power-query/data-profiling-tools

[9] Microsoft — Delta table schema evolution and enforcement: https://learn.microsoft.com/en-us/fabric/data-engineering/delta-lake-schema-evolution

[10] Microsoft — Primary, foreign, and unique keys in Fabric Warehouse: https://learn.microsoft.com/en-us/fabric/data-warehouse/table-constraints

[20] Microsoft — Lineage in Fabric: https://learn.microsoft.com/en-us/fabric/governance/lineage

[21] Microsoft — Impact analysis: https://learn.microsoft.com/en-us/fabric/governance/impact-analysis

[23] Microsoft — Set up metadata scanning in your organization: https://learn.microsoft.com/en-us/fabric/admin/metadata-scanning-setup

[24] Microsoft — Fabric Catalog Search REST API (preview): https://learn.microsoft.com/en-us/rest/api/fabric/core/catalog/search

[25] Microsoft — Tags in Microsoft Fabric: https://learn.microsoft.com/en-us/fabric/governance/tags-overview

[26] Microsoft — Endorsement overview: https://learn.microsoft.com/en-us/fabric/governance/endorsement-overview

[27] Microsoft — Data security in OneLake: https://learn.microsoft.com/en-us/fabric/onelake/security/get-started-security

[28] Microsoft — Create and manage OneLake security roles: https://learn.microsoft.com/en-us/fabric/onelake/security/create-manage-roles

[29] Microsoft — OneLake table, column, and row-level security: https://learn.microsoft.com/en-us/fabric/onelake/security/row-level-security

[30] Microsoft — Fabric permission model: https://learn.microsoft.com/en-us/fabric/security/permission-model

[31] Microsoft — Information protection in Fabric: https://learn.microsoft.com/en-us/fabric/governance/information-protection

[33] Microsoft — Protection policies in Fabric: https://learn.microsoft.com/en-us/fabric/governance/protection-policies-overview

[34] Microsoft — Dynamic data masking in Fabric Warehouse: https://learn.microsoft.com/en-us/fabric/data-warehouse/dynamic-data-masking

### Microsoft Purview: quality management and estate lineage

[11] Microsoft — Data quality overview in Purview Unified Catalog: https://learn.microsoft.com/en-us/purview/unified-catalog-data-quality

[12] Microsoft — Supported sources/formats for Unified Catalog quality: https://learn.microsoft.com/en-us/purview/unified-catalog-data-quality-supported-sources-file-formats

[13] Microsoft — Set up quality for Fabric Lakehouse data: https://learn.microsoft.com/en-us/purview/unified-catalog-data-quality-fabric-lakehouse

[14] Microsoft — Create data quality rules: https://learn.microsoft.com/en-us/purview/concepts-data-quality-rules

[15] Microsoft — Data quality scores: https://learn.microsoft.com/en-us/purview/unified-catalog-data-quality-scores

[16] Microsoft — Data quality alerts: https://learn.microsoft.com/en-us/purview/unified-catalog-data-quality-alerts

[17] Microsoft — Data quality thresholds: https://learn.microsoft.com/en-us/purview/unified-catalog-data-quality-threshold

[18] Microsoft — Data quality error records: https://learn.microsoft.com/en-us/purview/unified-catalog-data-quality-error-records

[19] Microsoft — Billing in Purview Data Governance: https://learn.microsoft.com/en-us/purview/unified-catalog-billing-consent

[22] Microsoft — Fabric item metadata and lineage in Purview: https://learn.microsoft.com/en-us/purview/data-map-lineage-fabric

[32] Microsoft — Get started with Purview DLP for Fabric and Power BI: https://learn.microsoft.com/en-us/purview/dlp-powerbi-get-started

[35] Microsoft — Register and scan a Fabric tenant in Purview: https://learn.microsoft.com/en-us/purview/register-scan-fabric-tenant

[41] Microsoft — Purview data quality for Fabric mirrored databases: https://learn.microsoft.com/en-us/purview/unified-catalog-data-quality-fabric-mirrored-databases

### Targeted Azure/AWS feature benchmarks

[36] AWS — Glue Data Quality: https://docs.aws.amazon.com/glue/latest/dg/glue-data-quality.html

[37] AWS — Amazon DataZone lineage: https://docs.aws.amazon.com/datazone/latest/userguide/datazone-data-lineage.html

[38] AWS — AWS Lake Formation: https://docs.aws.amazon.com/lake-formation/latest/dg/

[39] Microsoft — Lineage in Azure Databricks Unity Catalog: https://learn.microsoft.com/en-us/azure/databricks/data-governance/unity-catalog/data-lineage

[40] Microsoft — Azure Databricks Unity Catalog governance: https://learn.microsoft.com/en-us/azure/databricks/security/data-governance
