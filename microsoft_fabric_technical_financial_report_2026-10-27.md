# Microsoft Fabric: Architecture, Competitive Alternatives, CI/CD, and F4 Cost Evaluation

**Research date:** 27 October 2026  
**Scope:** Microsoft Fabric workloads, comparable Azure and non-Azure services, practical deployment automation, economics at an existing **F4 / 4-CU capacity**, and the difference between a Fabric Lakehouse and Azure Storage (including Blob and ADLS Gen2).  
**Method:** Vendor technical documentation and published prices are cited throughout. Financial scenarios are transparent *illustrations*, not claims about the organization's measured usage or performance. All amounts are USD, before taxes, enterprise discounts, egress, and ancillary services.

## Executive summary

1. **Fabric is an integrated analytics SaaS platform, not merely a no-code ETL tool.** It unifies ingestion, orchestration, Spark-based engineering, Delta lakehouses, SQL warehousing, event and KQL analytics, machine learning, transactional SQL, semantic modeling, BI, and governance around OneLake and a common capacity system. It reduces integration overhead but does not eliminate specialized products or their feature advantages. [S1][S2][S3]
2. **F4 is a shared compute budget, not four dedicated cores per service.** Published East US list-price reporting dated September 2026 gives approximately **$0.72/hour = $525.60/month** for a continuously running F4; a one-year reservation corresponds to **$312.67/month** in committed capacity charges. OneLake storage and certain user licenses remain separate. Region, contract, reservations, newer on-demand compute options, and enabled features can change the bill. [S4][S5][S6][S7]
3. **F4 does not unconditionally save money.** In the two specified *hypothetical* workload mixes below, the fully itemized Fabric F4 estimate is **$688.60/month** PAYG or **$475.67/month** with reservation, both including 1,000 GB OneLake hot storage and ten Power BI Pro users. Against other representative stacks, Fabric is **more expensive for a light-use scenario** and **less expensive in a busier scenario** if the workload really fits within F4. These are billing illustrations, **not throughput-equivalent benchmarks**. [S4][S5][S6][S7][S8][S9][S10][S11]
4. **Fabric CI/CD is genuine DevOps applied to workspace *definitions*.** Git integration, PR validation, Fabric deployment pipelines, variable libraries, REST APIs, fabric-cicd, CLI, and Terraform support Dev → Test → Prod. **Deployment does not copy lakehouse or warehouse data**; it does not automatically transplant credentials, schedules, or permissions. [S12][S13][S14][S15][S16]
5. **OneLake / Lakehouse is not simply another Blob container.** Azure Blob is a general-purpose storage service; ADLS Gen2 adds hierarchical filesystem semantics atop Blob; OneLake is Fabric's managed, tenant-wide logical lake built on ADLS Gen2; a Lakehouse is a managed *analytics item* over OneLake with Delta tables, Spark, and an automatically provisioned read-only SQL analytics endpoint. Blob and ADLS data can be exposed through shortcuts without wholesale copying. [S17][S18][S19][S20][S21]

## 1. What Fabric does and how the pieces fit

Fabric provides SaaS experiences for the whole path from source data to analytical decisions, centered on **OneLake**, a tenant-wide logical storage layer. Data Factory orchestrates and ingests; Data Engineering transforms through Spark; Lakehouse or Warehouse stores queryable Delta tables; Data Science trains and tracks models; Real-Time Intelligence ingests and analyzes events; and Power BI serves models and reports. Fabric also offers operational SQL databases, conversational Data Agents, catalog/governance, and integrated deployment tooling. Some features have independent prerequisites, additional charges, or service limitations. [S1][S2][S3][S22]

**A canonical batch architecture** is:

`Azure SQL / files / SaaS API → Fabric Copy job or Data Factory Pipeline → OneLake Lakehouse (Bronze) → Dataflow Gen2 / Spark Notebook (Silver) → Lakehouse Delta table or Fabric Warehouse (Gold) → Power BI semantic model → Reports`.

**An event-driven variant** is:

`Event Hubs / application event source → Eventstream → Eventhouse (KQL) → Real-Time Dashboard / Activator → alert or automation`.

These are patterns, not a claim that every data set must move through every stage. Shortcuts can expose data already held in Azure Blob, ADLS Gen2, S3, and other compatible sources without first replicating it; supported sources, credentials, transaction fees, and latency still matter. [S1][S3][S23][S20]

### 1.1 Components, functions, and closest alternatives

| Fabric capability | What it actually provides | Separate Azure alternative | Other comparable products | Fabric advantage / where the alternative can win |
|---|---|---|---|---|
| **Data Factory: Copy job, pipeline** | Connect, copy, schedule, branch, and orchestrate ETL/ELT; pipelines invoke notebooks, dataflows and other jobs. | Azure Data Factory, Logic Apps for specific workflows | AWS Glue/Step Functions; Airflow; Fivetran | Native workspace/OneLake access and lower integration effort; ADF has longstanding specialized integration-runtime, SSIS and networking options, and activity support differs. [S24][S25] |
| **Dataflow Gen2** | Low-code Power Query-style data preparation and transforms | ADF mapping data flows, Power Query, Azure Databricks | Alteryx, dbt, Glue Studio | Great for analysts and common cleanup; code-oriented dbt/Spark is more transparent and flexible for complex reusable software engineering patterns. [S24][S25][S26] |
| **Data Engineering / Spark notebooks / Spark job definitions** | Python, Scala, Spark SQL, R; batch transformations over Delta | Azure Databricks, Synapse Spark | Databricks; AWS Glue / EMR; Google Managed Spark | Less setup and integrated BI; Databricks offers a broader specialist data-engineering / ML ecosystem and independent cluster sizing. [S1][S27] |
| **OneLake + Lakehouse** | Managed Files + Delta Tables on OneLake; Spark writes and read-only SQL endpoint | ADLS Gen2 + Delta / Databricks / Synapse serverless | S3 + Delta/Iceberg; Databricks Lakehouse; GCS | Fewer services to assemble, shortcuts and shared catalog; independently managed storage gives more storage/account/network design autonomy. [S17][S18][S19] |
| **Warehouse / SQL analytics** | Managed T-SQL analytics, curated relational models, Delta-based storage | Synapse dedicated SQL pool, Azure SQL analytical patterns | Snowflake, BigQuery, Redshift, Databricks SQL | Integrated storage and Power BI, shared CU pool; standalone engines can better fit independent scaling, SQL dialect, isolation, or workload-specific performance. [S3][S28][S29] |
| **SQL database in Fabric** | Operational OLTP database based on Azure SQL, with analytic replication into OneLake | Azure SQL Database | Amazon RDS / Aurora; Cloud SQL | Simplifies operational-to-analytics path; external OLTP services remain appropriate for independent production application lifecycles and sizing. [S22] |
| **Real-Time Intelligence: Eventstream, Eventhouse, Activator** | Streaming ingestion, KQL event/log storage, real-time dashboards and rules-triggered actions | Event Hubs + Stream Analytics / Azure Data Explorer + Logic Apps | Kinesis, Kafka, Flink, Confluent, ClickHouse | End-to-end event analytics with no-code rules; specialist streaming systems offer finer control over streaming semantics and infrastructure. [S23][S30] |
| **Data Science / ML** | Integrated notebooks, experiments/models and analytics preparation | Azure Machine Learning | SageMaker, Vertex AI, Databricks ML | Convenient near governed analytical data; specialized ML platforms may provide richer, independently scaled training/serving operations. [S1][S27][S31] |
| **Power BI** | Semantic models, DAX, dashboards and organizational reporting | Power BI as stand-alone SaaS (same underlying product) | Tableau, Looker, QuickSight | Direct Lake and close native integration; other BI tools can win on an existing BI estate, visualization semantics or a different operating model. Pro user licensing applies on F4. [S1][S7][S10] |
| **Governance / OneLake Catalog** | Shared catalog, discovery, workspace controls, built-in Purview integration | Microsoft Purview + separate Azure service governance | Unity Catalog, AWS Glue Catalog/Lake Formation, Google Knowledge Catalog | More cohesive discovery for Fabric items; multi-cloud control planes or specialist catalogs may be preferable for a heterogeneous organization. [S1][S27][S31] |
| **Fabric Data Agents / Copilot** | Natural-language conversation with governed Fabric data; assistants for authoring | Microsoft Foundry + AI Search + Azure OpenAI (custom app composition) | Snowflake Cortex Analyst, Databricks Genie, custom text-to-SQL | Faster conversational analytics over supported Fabric items; custom RAG, complex application integration, model selection and bespoke control remain external projects. Data Agent has prerequisites. [S1][S32] |
| **Git, deployment pipelines, variable library, APIs** | Version and promote item definitions / configuration across workspaces | Azure DevOps/GitHub + ARM/Bicep/Terraform per Azure service | GitHub Actions + dbt Cloud, Databricks Bundles, Airflow CI | Unified Fabric-item promotion; cross-service DevOps may expose more granular infrastructure controls and test tooling. [S12][S13][S14][S15][S16] |

**Crucial differences in the UI:** A **Fabric task flow** is a visualization of the project's logical steps and item associations, not an ETL execution scheduler. **A Data Factory pipeline** actually coordinates executable activities, can call a notebook or Dataflow, and can be scheduled or triggered. There is no need to re-create every pipeline-internal activity on the task-flow canvas. [S33][S24]

### 1.2 Strengths and drawbacks by architectural concern

**Advantages supported by product architecture:** Common OneLake layer; reduced copy/movement when using Shortcuts; a unified workspace/catalog/security experience; integration among Power Query, Spark, SQL and Power BI; SaaS managed runtimes; one shared purchased CU pool; and native Git/deployment capabilities. This generally reduces glue-code, duplicated identity/resource setup, and the number of billing meters the team manages. Those operational benefits are plausible design consequences, **not an independent quantified savings study**. [S1][S4][S12][S17]

**Disadvantages and genuine trade-offs:** (a) pipelines, warehouse queries, Spark, BI, and events may compete for the same F4 pool; (b) the service coverage and network/security details are not identical to ADF or specialist products; (c) F4 has only one starter-pool node by default/maximum in current Spark settings, with an 8-vCore SKU allocation; (d) background CU usage is smoothed, which can accumulate debt and lead to throttling; (e) independently scalable serverless/PaaS products can be cheaper when lightly used; (f) reservations trade flexibility for cost; (g) portability must be evaluated at the query, orchestration, security, and semantic model levels—not only file format. [S25][S34][S35][S4][S8]

**Capacity sizing point:** The 4 CUs are a **billing/compute capacity abstraction, not 4 vCPUs**. Fabric can burst above its nominal CU allocation and smooth operation consumption (interactive operations over minutes, background operations over 24 hours); sustained use can still throttle. On a small SKU, run a representative load test of ingestion, Spark, SQL and report concurrency before assuming it is a replacement for several larger independently billed systems. [S35][S34][S36]

## 2. How to implement Fabric CI/CD

### 2.1 Recommended architecture

```text
Developers / feature branches
         ↓ pull request + automated checks
GitHub or Azure DevOps repository (main branch)
         ↓ Fabric Git sync
Fabric DEV workspace  ──[deployment pipeline / REST / fabric-cicd]──> TEST workspace
                                                                   ↓ tests, approvals
                                                               PROD workspace

Environment-specific Variable Library value sets / deployment rules
                + Secrets and connection management per environment
```

Create distinct **Development**, **Test**, and **Production** workspaces. For smaller cost-sensitive teams they can sit on a single F4 capacity, but that creates performance contention and weakens performance isolation; independent capacities add costs. Connect the Dev workspace to GitHub or Azure DevOps from workspace settings. Git integration versions item definitions such as pipelines, notebooks, lakehouses, warehouses and semantic assets **where supported**; consult the current supported-items list for preview flags and item-specific limits. [S12][S13][S15]

### 2.2 A workable sequence

1. **Model each environment:** Create `fabric-project-dev`, `fabric-project-test`, `fabric-project-prod`, equivalent lakehouse/warehouse items, access controls and appropriate Fabric capacities. Use the Fabric Terraform provider for supported resource provisioning. Keep production credentials and sensitive configuration out of Git. [S12][S16]
2. **Develop:** Use the UI to construct a no-code Dataflow or Pipeline, or notebooks/SQL for code-based transforms. Associate the Dev workspace with a repo/branch; commit workspace changes to Git. No-code **does not mean unversionable**: supported items serialize definitions to source-controlled files. [S12][S15]
3. **Validate pull requests:** Review diffs and run appropriate checks: parse notebook/SQL definitions, Python unit tests, SQL lint and schema assertions, approved pipeline definitions, source access policy checks, and optionally integration tests in a nonproduction workspace. These tests are recommended engineering controls, **not default automatic guarantees** of Fabric's UI. [S12][S16]
4. **Configure target-specific references:** Make separate variable-library value sets for `dev`, `test`, `prod` (e.g., source URLs, schema names, workspace and lakehouse references, pipeline parameters). Use native auto-binding and deployment rules for supported references. Verify credentials, gateways, permissions and data source connections independently. [S14][S15][S37]
5. **Promote definitions:** Create a Fabric deployment pipeline, associate workspaces with its stages, compare items, and selectively deploy Dev → Test. After testing, promote Test → Prod. For greater automation, invoke deployment REST APIs via Azure DevOps/GitHub Actions, or use `fabric-cicd` with item-definition parameterization; the exact orchestration approach depends on whether Git or the Fabric workspace is the source of truth. [S12][S13][S16]
6. **Provision/load test data separately:** Fabric deployment pipelines **do not copy data—only metadata**. Deploy tables/definitions and independently ingest, seed, or migrate Test/Prod data. Recheck source mappings, security, model refresh schedule, semantic model roles, credentials, subscriptions, triggers, and permissions, because not every item property transfers. [S37]
7. **Smoke-test and monitor:** Execute a small pipeline job, validate row counts/schemas/data freshness, exercise SQL endpoints and report visuals, monitor capacity CU seconds and throttling. Keep production activation behind approval gates. [S36][S12]
8. **Rollback:** Re-deploy known-good definitions from Git/release history where supported. Treat a destructive SQL migration and data changes as separate, explicitly reversible tasks; a Git revert by itself will not restore lakehouse rows. [S15][S37]

**Other supported CI/CD modes:** Microsoft documents per-stage Git branches, main-branch item-API deployment (`fabric-cicd`), and Fabric-native deployment pipelines. For a mostly low-code team, **Git in Dev + Fabric deployment pipelines + stage variables** is simplest. For a team requiring fully scripted promotions, **GitHub Actions/Azure Pipelines + Fabric REST/CLI/fabric-cicd + Terraform** is better controlled. [S13][S12][S16]

**Common failure modes:** Assuming an item exists in Git even when Git support is preview/unavailable; exporting the wrong production connection; deploying definitions without credentials or schedules; forgetting that data is not deployed; overloading the shared F4 while Test runs; skipping actual pipeline run tests. Microsoft publishes supported-item lists and item-specific restrictions—these need a fresh check for every release. [S12][S15][S37]

## 3. Costs: Fabric F4 vs separate Azure vs mixed-vendor services

### 3.1 Rates and financial assumptions

The following are **reported reference list rates**, not a signed quote. Microsoft's region-filtered Fabric price table does not expose numeric prices in the crawler's text view; the F4 rate used here is a **September 12, 2026 East US Azure retail-list quotation reproduced by a third-party licensing analyst**. Confirm the applicable region and Azure contract in the live calculator before a spending decision. Microsoft's own pricing page confirms the shared CU and reservation model and says reserved capacity can save roughly 41%. [S4][S5][S6]

| Billing item | Rate used here | Evidence / qualification |
|---|---:|---|
| F4 = 4 CU PAYG | **$0.72 per capacity-hour** | East US quoted 12 Sep 2026; varies by region/contract. [S4][S5] |
| F4 1-year reservation | **$3,752/year = $312.67/month equivalent** | Annual commitment; quoted East US list. [S5] |
| OneLake hot data storage | **$0.023 per GB-month** | Microsoft Learn example rate; region, storage tier, cache, transactions and BCDR can differ. [S6][S38] |
| Power BI Pro | **$14 per user-month** | Microsoft public list price, billed yearly; some Microsoft 365 licenses may already include Pro. F4 does not exempt viewers from paid-license requirements. [S7][S10] |
| Azure Data Factory orchestration | **$1 per 1,000 activity runs** | Azure-hosted Integration Runtime reference rate. [S8] |
| Azure Data Factory copy/movement | **$0.25 per DIU-hour** | Azure-hosted Integration Runtime reference rate. [S8] |
| Azure Databricks **example selected DS12 v2 configuration** | **$489.10/730 h ≈ $0.67/active hour** | Taken from Microsoft's Databricks pricing table's monthly-equivalent PAYG total for that specific VM/DBU configuration. Not a universal Spark price or equivalence to F4 Spark. [S9] |
| Synapse serverless SQL | **$5 per processed TB (reference rate)** | Microsoft example/technical guidance, billed on processed data; live rate is region-dependent. [S11][S39] |
| AWS Glue Spark jobs | **$0.44 per DPU-hour** | AWS published example rate. [S40] |
| Amazon Athena SQL | **$5 per TB scanned** | AWS published example rate; pruning and formats can change scanned volume. [S41] |
| Separate storage in alternative stacks | **$23 per 1,000 GB-month** | **Model assumption** for a common storage-equivalent comparison, *not* an Azure Blob/S3 procurement quote. Storage API operations, replication, tiers and region may change actual costs. OneLake reference used as a neutral placeholder. [S6][S42] |

**Common scenario inputs:** A continuously available F4 for **730 hours per month**, **1,000 GB** billable hot data, **ten $14 Power BI Pro users**, 2,000 ADF activity runs in the Azure case, and a shared single analytical reporting requirement. This is an illustrative small-team model; real Fabric capacities may serve many workspaces rather than one. Other products' different compute unit systems are *not interchangeable*. [S4][S7][S8][S35]

### 3.2 F4 baseline including separate charges

- **PAYG compute:** $0.72 × 730 = **$525.60/month**.
- **Reserved compute:** $3,752 ÷ 12 = **$312.67/month equivalent** on a one-year commitment.
- **OneLake hot storage:** 1,000 GB × $0.023 = **$23.00/month** (not compute CUs).
- **Power BI:** 10 × $14 = **$140.00/month** (assuming none of those ten licenses is already included in another agreement).
- **All-in model:** **$688.60/month PAYG** or **$475.67/month reserved**. The latter is an allocated monthly equivalent of a year-long financial commitment, not permission to buy only one reserved month. [S4][S5][S6][S7][S10]

**Excluded from both totals:** backup/BCDR storage, OneLake cache, read/write/retrieval transaction-related CU use, external network egress, separate services, optional on-demand/autoscale Spark billing, overage, premium gateways or security, special AI charges, capacity for additional environments, engineer labor, and taxes. Pricing and performance must be recalculated if those apply. [S4][S6][S38][S35]

### 3.3 Two transparent workload mixes

**Scenario L (light analytical use):** Azure: 100 DIU-hours ADF movement, 2,000 orchestration activities, 120 Databricks DS12-v2 hours, 20 TB Synapse query processing. Mixed-vendor: 4 AWS Glue DPUs × 120 hours, 20 TB Athena scanned, external Power BI; 1,000 GB storage in each architecture. **Scenario B (busier use):** Azure: 300 DIU-hours, 2,000 activities, 240 Databricks hours, 80 TB Synapse data processing. Mixed-vendor: 4 Glue DPUs × 240 hours, 80 TB Athena scanned. Shared storage and BI counts unchanged. All service-meter quantities are **hypothetical inputs**, not actual measured workloads or performance-equivalent runs. Sources of rate units: [S8][S9][S10][S11][S40][S41].

| Expense per month | Fabric F4 PAYG | Fabric F4 1-year reserved | Azure services, L | Mixed AWS + Power BI, L | Azure services, B | Mixed AWS + Power BI, B |
|---|---:|---:|---:|---:|---:|---:|
| Compute capacity or job compute | $525.60 | $312.67 | Databricks $80.40 | Glue $211.20 | Databricks $160.80 | Glue $422.40 |
| Data-movement / orchestration | Included in modeled F4 capacity* | Included* | $27.00 | Not separately modeled | $77.00 | Not separately modeled |
| Analytics SQL usage | Included in modeled F4 capacity* | Included* | $100.00 | $100.00 | $400.00 | $400.00 |
| Data storage placeholder | $23.00 | $23.00 | $23.00 | $23.00 | $23.00 | $23.00 |
| Ten Power BI Pro licenses | $140.00 | $140.00 | $140.00 | $140.00 | $140.00 | $140.00 |
| **Illustrative monthly total** | **$688.60** | **$475.67** | **$370.40** | **$474.20** | **$800.80** | **$985.40** |

*Fabric compute coverage assumes all the modeled Fabric activity can actually execute within F4 without additional billed capacity/services or unacceptable performance. That assumption **has not been demonstrated**; OneLake transactions and supplemental charges could apply. Mixed AWS orchestration and data-movement charges are not separately modeled, so its scenario totals are **lower-bound illustrations**, not fully burdened production costs. [S4][S6][S38]

**Interpreting the examples:**

- In **L**, Fabric F4 PAYG costs **$318.20 more** than the Azure mix and **$214.40 more** than the mixed-vendor mix per modeled month. The reserved version is still **$105.27 more** than Azure and **$1.47 more** than mixed AWS; the reservation also locks in capacity spending. These are scenarios, not observed results.
- In **B**, assuming the F4 can do the work, Fabric PAYG is **$112.20 (14.0%) less** than the Azure mix and **$296.80 (30.1%) less** than mixed AWS. Reserved F4 is **$325.13 (40.6%) less** than Azure and **$509.73 (51.7%) less** than mixed AWS. **This does not establish an F4 performance benchmark.**
- The cost pattern is understandable: F4 incurs steady capacity charges while bursty independent services meter query bytes, job hours, or activities. As utilization rises, shared capacity may amortize better **until concurrency, CU consumption, or features force scaling**; for low duty cycles, elastic pay-per-use may be cheaper. [S4][S8][S11][S35][S40][S41]

### 3.4 What a *real* savings claim requires

The only defensible **production** cost-savings percentage is:

`(Current matched-workload total monthly cost − Proposed Fabric matched-workload total monthly cost) / Current matched-workload total monthly cost`.

Measure current data source bytes, storage growth/tiers, movement DIU/runtime, query bytes/warehouse-hours, Spark job hours/configuration, peaks, BI viewers, orchestration, network, and maintenance. In Fabric, use the **Capacity Metrics app** to collect per-item **CU-seconds**, 14-day usage, high-watermark utilization and throttling and 30-day storage usage. Replay the same input data, concurrency, freshness objectives, SLA, and query quality through an F4 pilot. At 730h continuously, 4 CU supply **10,512,000 nominal CU-seconds/month**, which should be interpreted in conjunction with smoothing, bursting, and throttling—not as a promise every job receives dedicated hardware. [S35][S36]

**Special F4 caveats:** 1) viewing Power BI reports on F4 requires per-user Pro/PPU (or eligible trial); the *free viewer* exception normally starts at **F64**; 2) 1-year reservation is a financial commitment; 3) an F4 starter Spark pool's default and maximum nodes are both **one** in the cited table; 4) Fabric now has additional on-demand/autoscale billing options that require explicit settings and separate assessment; 5) mirrored-database replica storage has a distinct F4 allowance **up to 4 TB**, which is **not** 4 TB of free general lakehouse storage. [S7][S34][S4]

## 4. Fabric Lakehouse vs Azure Storage, Azure Blob, and ADLS Gen2

### 4.1 Four layers, not interchangeable products

| Concept | Correct definition | What you provision / manage | Typical best fit |
|---|---|---|---|
| **Azure Blob Storage** | General Azure object/blob storage, storing bytes (raw files, images, logs, backups, or Delta files) | Azure storage account, containers, access policies, networking, redundancy, tier/lifecycle, transactions | Source/landing/archive/application object files; independent cloud workloads. [S18][S19] |
| **Azure Data Lake Storage Gen2 (ADLS Gen2)** | Analytics-oriented **capabilities built on Azure Blob Storage**, especially hierarchical namespace and filesystem security/semantics | An Azure Blob storage account with hierarchical namespace enabled; directory/ACL/access patterns and external compute services | Self-managed enterprise data lake accessed by many compute engines. [S19] |
| **Fabric OneLake** | Fabric's automatically provided **tenant-wide logical storage layer built on ADLS Gen2**, governed as Fabric SaaS | Fabric capacities and workspaces/items; OneLake policies and billable storage, without separately engineering an ADLS storage account for Fabric-native data | Shared storage fabric across Fabric workloads; cross-item discovery and zero-copy patterns. [S1][S17] |
| **Fabric Lakehouse** | A **Fabric item over OneLake**, organizing **Files** plus managed **Delta Tables**, Spark interfaces, and read-only T-SQL analytics endpoint | Lakehouse item, tables, shortcuts, permissions, Spark/SQL processes and data quality | Medallion data engineering + SQL BI on shared Delta data. [S3][S18] |

A simple analogy is **Blob/ADLS = storage substrate**, **OneLake = managed enterprise lake abstraction**, and **Lakehouse = analytics workspace/data model over that lake**. Creating a Lakehouse is **not** equivalent to creating an empty Azure Blob container, and creating an ADLS container **does not** automatically furnish Spark/SQL engines or Power BI semantic models. [S1][S3][S19]

### 4.2 Specific technical differences

- **File formats and queryability:** Blob/ADLS can hold arbitrary files; Fabric Lakehouse Files can also hold arbitrary files, but its managed Tables use Delta/Parquet with Delta transaction logs. The automatically provisioned Lakehouse **SQL analytics endpoint reads Delta tables**; it is **read-only for data changes** and does not query raw CSV/Parquet in Files as registered lakehouse tables. Use Spark/engineering to convert files, or separate query options. [S18][S21]
- **Storage ownership and governance:** Azure Blob and ADLS storage accounts offer user-controlled networking, redundancy, tier and access control. Fabric provides a SaaS OneLake workspace layer and catalog/access control with less storage infrastructure setup. This improves convenience but can reduce the degree of control over low-level storage design. [S1][S17][S19]
- **Cost:** OneLake general hot storage is **not included free in F4**; it is billed separately per GB, and transactions can consume capacity. OneLake now also documents cool/cold storage tiers with different storage and transaction economics. ADLS/Blob storage is likewise billed for stored data/requests/replication and optionally other services. Compare same region, same hot/cool, replication and access pattern; do not assume either is always cheaper. [S6][S38]
- **Connecting to existing storage:** OneLake **shortcuts** let a lakehouse expose paths in ADLS Gen2 or **Azure Blob Storage**, with distinct connector and security settings. They are not a bulk migration. Data stays at the source for a shortcut, but source billing, access checks, cross-region/network costs, caching and formats can still affect total cost and latency. [S20][S43][S44]
- **Choice:** Keep Azure Blob for raw operational/application objects and cold archives; keep ADLS Gen2 if several engines/apps need a separately administered central data lake; prefer Fabric Lakehouse for a Fabric-led Delta/Spark/Power BI analytical domain. Hybrid is sensible: Blob/ADLS as source of record with a Fabric shortcut and selective curated Delta output to OneLake. [S18][S19][S20]

## 5. Recommendation for the existing F4 implementation

**Recommended initial direction: retain F4 for a measured proof of value, not a forced all-service replacement.** Put representative ingestion, a low-code Dataflow, one Spark notebook, Lakehouse Delta table, optional warehouse SQL, and a Power BI semantic model on the same capacity. Record CU consumption, runtimes, query response time, refresh success, peak BI concurrency, and any throttling in Capacity Metrics. F4 is particularly useful to test whether unified developer workflow and lower operational overhead matter more than raw compute price. [S1][S34][S35][S36]

**Workload boundaries:** Keep existing Azure Storage if it already serves other consumers; shortcut it before paying to duplicate raw data. Keep Azure Data Factory when a connector, SSIS migration, managed integration runtime, or networking requirement is not covered appropriately by Fabric. Retain Azure Databricks or Azure ML where separately scaled high-intensity Spark/ML is required. Treat Fabric's Data Agent as a governed **data Q&A** interface—not a substitute for arbitrary enterprise RAG pipelines, custom vector search, or model-serving application architecture. [S20][S24][S25][S27][S32]

**CI/CD minimum viable pattern:** Git-backed Dev workspace → required PR approvals/checks → Fabric native Deployment Pipeline → Test execution and data checks → Production promotion; manage environment configuration through Variable Library and deployment rules, then separately validate operational data, scheduling, gateway connections, refresh credentials and roles. Set up separate production capacity if isolation is a hard SLA. [S12][S13][S14][S15][S16][S37]

**Finance decision gate:** Use the report's two modeled scenarios as bounds, not quotes. Export actual Azure/AWS/vendor bills and Fabric Capacity Metrics, build a line-item cost table at identical data volume and service quality, then recalculate with existing enterprise Power BI licenses, real storage retention, egress and required 24×7 availability. Only accept a cost-savings claim when **F4 throughput and reliability are demonstrated under the exact benchmark**. [S4][S7][S35][S36]

## 6. Sources and verification links

[S1] Microsoft Learn — [What is Microsoft Fabric?](https://learn.microsoft.com/en-us/fabric/fundamentals/microsoft-fabric-overview)  
[S2] Microsoft Learn — [Microsoft Fabric workloads, Well-Architected Framework](https://learn.microsoft.com/en-us/azure/well-architected/microsoft-fabric/overview)  
[S3] Microsoft Learn — [Data storage options in Microsoft Fabric](https://learn.microsoft.com/en-us/fabric/fundamentals/store-data)  
[S4] Microsoft Azure — [Microsoft Fabric pricing](https://azure.microsoft.com/en-us/pricing/details/microsoft-fabric/)  
[S5] Microsoft Made Easy — [2026 Fabric F-SKU price sheet commentary, citing 12 Sep 2026 Azure retail rates](https://microsoftmadeeasy.com/microsoft-fabric-licensing-cost) (secondary price source: verify against a live Azure quote)  
[S6] Microsoft Learn — [OneLake storage tiers (example $/GB rates)](https://learn.microsoft.com/en-us/fabric/onelake/onelake-storage-tiers)  
[S7] Microsoft Learn — [Fabric license and capacity matrix](https://learn.microsoft.com/en-us/fabric/enterprise/licenses)  
[S8] Microsoft Learn — [Azure Data Factory pricing, per-run and DIU-hour](https://learn.microsoft.com/en-au/azure/data-factory/pricing)  
[S9] Microsoft Azure — [Azure Databricks pricing](https://azure.microsoft.com/en-us/pricing/details/databricks/)  
[S10] Microsoft — [Power BI Pro and PPU list prices](https://www.microsoft.com/en-us/power-platform/products/power-bi/pricing)  
[S11] Microsoft Tech Community — [Synapse serverless SQL $5/TB processed worked example](https://techcommunity.microsoft.com/blog/azuresynapseanalyticsblog/metadata-based-ingestion-in-synapse-with-delta-lake/3844900)  
[S12] Microsoft Learn — [Introduction to Fabric CI/CD](https://learn.microsoft.com/en-us/fabric/cicd/cicd-overview)  
[S13] Microsoft Learn — [Choose a Fabric CI/CD deployment workflow](https://learn.microsoft.com/en-us/fabric/cicd/manage-deployment)  
[S14] Microsoft Learn — [Fabric variable library](https://learn.microsoft.com/en-us/fabric/cicd/variable-library/variable-library-overview)  
[S15] Microsoft Learn — [Fabric Git integration and supported items](https://learn.microsoft.com/en-us/fabric/cicd/git-integration/intro-to-git-integration)  
[S16] Microsoft Learn — [Fabric CI/CD best practices](https://learn.microsoft.com/en-us/fabric/fundamentals/understand-best-practices-fabric-cicd)  
[S17] Microsoft Learn — [Fabric overview: OneLake based on ADLS Gen2](https://learn.microsoft.com/en-us/fabric/fundamentals/microsoft-fabric-overview)  
[S18] Microsoft Learn — [Fabric Lakehouse overview](https://learn.microsoft.com/en-us/fabric/data-engineering/lakehouse-overview)  
[S19] Microsoft Learn — [Introduction to Azure Data Lake Storage / Blob](https://learn.microsoft.com/en-us/azure/storage/blobs/data-lake-storage-introduction)  
[S20] Microsoft Learn — [Fabric OneLake shortcuts](https://learn.microsoft.com/en-us/fabric/onelake/onelake-shortcuts)  
[S21] Microsoft Learn — [Lakehouse SQL analytics endpoint](https://learn.microsoft.com/en-us/fabric/data-engineering/lakehouse-sql-analytics-endpoint)  
[S22] Microsoft Learn — [SQL database in Fabric](https://learn.microsoft.com/en-us/fabric/database/sql/overview)  
[S23] Microsoft Learn — [Getting started with Real-Time Intelligence](https://learn.microsoft.com/en-us/fabric/real-time-intelligence/get-started-with-real-time-intelligence)  
[S24] Microsoft Learn — [Fabric Data Factory](https://learn.microsoft.com/en-us/fabric/data-factory/data-factory-overview)  
[S25] Microsoft Learn — [Differences between Fabric Data Factory and Azure Data Factory](https://learn.microsoft.com/en-us/fabric/data-factory/compare-fabric-data-factory-and-azure-data-factory/)  
[S26] Google Cloud — [Dataform: Git-managed SQL transformations](https://docs.cloud.google.com/dataform/docs/overview)  
[S27] Databricks — [Scope of the Databricks platform (updated September 2026)](https://docs.databricks.com/aws/en/lakehouse-architecture/scope)  
[S28] Microsoft Learn — [Fabric Data Warehouse](https://learn.microsoft.com/en-us/fabric/data-warehouse/data-warehousing)  
[S29] Google Cloud — [Data/analytics platform products](https://cloud.google.com/products)  
[S30] Microsoft Azure — [Separate Azure analytics offerings](https://azure.microsoft.com/en-us/products/category/analytics/)  
[S31] Google Cloud — [Data catalog, data analytics, and orchestration products](https://docs.cloud.google.com/docs/data)  
[S32] Microsoft Learn — [Fabric Data Agent concepts](https://learn.microsoft.com/en-us/fabric/data-science/concept-data-agent)  
[S33] Microsoft Learn — [Set up task flows (project visualization)](https://learn.microsoft.com/en-us/fabric/fundamentals/task-flow-create)  
[S34] Microsoft Learn — [Fabric Spark starter pool node limits](https://learn.microsoft.com/en-us/fabric/data-engineering/configure-starter-pools)  
[S35] Microsoft Learn — [Fabric bursting, smoothing and throttling](https://learn.microsoft.com/en-us/fabric/enterprise/throttling)  
[S36] Microsoft Learn — [Fabric Capacity Metrics app](https://learn.microsoft.com/en-us/fabric/enterprise/metrics-app)  
[S37] Microsoft Learn — [Deployment process and non-copied metadata/data](https://learn.microsoft.com/en-us/fabric/cicd/deployment-pipelines/understand-the-deployment-process)  
[S38] Microsoft Learn — [OneLake compute/storage/transaction consumption](https://learn.microsoft.com/en-us/fabric/onelake/onelake-consumption)  
[S39] Microsoft Learn — [Synapse serverless SQL cost management](https://learn.microsoft.com/en-us/azure/synapse-analytics/sql/data-processed)  
[S40] AWS — [AWS Glue pricing](https://aws.amazon.com/glue/pricing/)  
[S41] AWS — [Amazon Athena pricing](https://aws.amazon.com/athena/pricing/)  
[S42] AWS — [Amazon S3 pricing, region- and tier-dependent](https://aws.amazon.com/s3/pricing/)  
[S43] Microsoft Learn — [Azure Blob Storage shortcut](https://learn.microsoft.com/en-us/fabric/onelake/create-blob-shortcut)  
[S44] Microsoft Learn — [ADLS Gen2 shortcut](https://learn.microsoft.com/en-us/fabric/onelake/create-adls-shortcut)  

### Confidence and limitations

**High confidence:** Product architecture, licensing rules, Lakehouse-versus-storage distinction, deployment-versus-data distinction, existence of specific tooling, and unit definitions, because these come from current vendor documentation. **Medium confidence:** Public example list prices, because rates vary by region, negotiated contract, and time; the F4 number is supported by a secondary September 2026 Azure retail-list transcription rather than a machine-readable live Azure price quote. **Low confidence for *real organizational savings*:** The scenarios are intentionally hypothetical and not calibrated to measured work, comparable runtime performance, or any organization's actual charges. No production savings percentage is claimed.

[S1]: https://learn.microsoft.com/en-us/fabric/fundamentals/microsoft-fabric-overview
[S2]: https://learn.microsoft.com/en-us/azure/well-architected/microsoft-fabric/overview
[S3]: https://learn.microsoft.com/en-us/fabric/fundamentals/store-data
[S4]: https://azure.microsoft.com/en-us/pricing/details/microsoft-fabric/
[S5]: https://microsoftmadeeasy.com/microsoft-fabric-licensing-cost
[S6]: https://learn.microsoft.com/en-us/fabric/onelake/onelake-storage-tiers
[S7]: https://learn.microsoft.com/en-us/fabric/enterprise/licenses
[S8]: https://learn.microsoft.com/en-au/azure/data-factory/pricing
[S9]: https://azure.microsoft.com/en-us/pricing/details/databricks/
[S10]: https://www.microsoft.com/en-us/power-platform/products/power-bi/pricing
[S11]: https://techcommunity.microsoft.com/blog/azuresynapseanalyticsblog/metadata-based-ingestion-in-synapse-with-delta-lake/3844900
[S12]: https://learn.microsoft.com/en-us/fabric/cicd/cicd-overview
[S13]: https://learn.microsoft.com/en-us/fabric/cicd/manage-deployment
[S14]: https://learn.microsoft.com/en-us/fabric/cicd/variable-library/variable-library-overview
[S15]: https://learn.microsoft.com/en-us/fabric/cicd/git-integration/intro-to-git-integration
[S16]: https://learn.microsoft.com/en-us/fabric/fundamentals/understand-best-practices-fabric-cicd
[S17]: https://learn.microsoft.com/en-us/fabric/fundamentals/microsoft-fabric-overview
[S18]: https://learn.microsoft.com/en-us/fabric/data-engineering/lakehouse-overview
[S19]: https://learn.microsoft.com/en-us/azure/storage/blobs/data-lake-storage-introduction
[S20]: https://learn.microsoft.com/en-us/fabric/onelake/onelake-shortcuts
[S21]: https://learn.microsoft.com/en-us/fabric/data-engineering/lakehouse-sql-analytics-endpoint
[S22]: https://learn.microsoft.com/en-us/fabric/database/sql/overview
[S23]: https://learn.microsoft.com/en-us/fabric/real-time-intelligence/get-started-with-real-time-intelligence
[S24]: https://learn.microsoft.com/en-us/fabric/data-factory/data-factory-overview
[S25]: https://learn.microsoft.com/en-us/fabric/data-factory/compare-fabric-data-factory-and-azure-data-factory/
[S26]: https://docs.cloud.google.com/dataform/docs/overview
[S27]: https://docs.databricks.com/aws/en/lakehouse-architecture/scope
[S28]: https://learn.microsoft.com/en-us/fabric/data-warehouse/data-warehousing
[S29]: https://cloud.google.com/products
[S30]: https://azure.microsoft.com/en-us/products/category/analytics/
[S31]: https://docs.cloud.google.com/docs/data
[S32]: https://learn.microsoft.com/en-us/fabric/data-science/concept-data-agent
[S33]: https://learn.microsoft.com/en-us/fabric/fundamentals/task-flow-create
[S34]: https://learn.microsoft.com/en-us/fabric/data-engineering/configure-starter-pools
[S35]: https://learn.microsoft.com/en-us/fabric/enterprise/throttling
[S36]: https://learn.microsoft.com/en-us/fabric/enterprise/metrics-app
[S37]: https://learn.microsoft.com/en-us/fabric/cicd/deployment-pipelines/understand-the-deployment-process
[S38]: https://learn.microsoft.com/en-us/fabric/onelake/onelake-consumption
[S39]: https://learn.microsoft.com/en-us/azure/synapse-analytics/sql/data-processed
[S40]: https://aws.amazon.com/glue/pricing/
[S41]: https://aws.amazon.com/athena/pricing/
[S42]: https://aws.amazon.com/s3/pricing/
[S43]: https://learn.microsoft.com/en-us/fabric/onelake/create-blob-shortcut
[S44]: https://learn.microsoft.com/en-us/fabric/onelake/create-adls-shortcut
