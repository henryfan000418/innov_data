# Microsoft Fabric: Architecture, Competitive Alternatives, CI/CD, and F4 Cost Evaluation

**Research date:** 30 October 2026 (native-stack comparison revision)  
**Scope:** Microsoft Fabric workloads, comparable Azure and non-Azure services, practical deployment automation, economics at an existing **F4 / 4-CU capacity**, and the difference between a Fabric Lakehouse and Azure Storage (including Blob and ADLS Gen2).  
**Method:** Vendor technical documentation and published prices are cited throughout. Financial scenarios are transparent *illustrations*, not claims about the organization's measured usage or performance. All amounts are USD, before taxes, enterprise discounts, egress, and ancillary services.

## Executive summary

1. **Three end-to-end, same-vendor-family designs:** **Fabric:** Fabric Pipeline/Copy → OneLake Lakehouse → Fabric Spark → Fabric SQL analytics → Power BI. **Azure:** Azure Data Factory Copy → Azure Blob Hot LRS → ADF Mapping Data Flow (managed Spark) → Synapse serverless SQL → traditional Power BI Pro. **AWS:** AWS Glue ingestion → S3 Standard → AWS Glue Spark → Athena → **Amazon QuickSight**. All platform components belong to their respective Microsoft Fabric, Azure, or AWS families; the previous AWS + Power BI hybrid comparison has been retired. [S1][S24][S63][S65][S75]
2. **Storage alone is almost a wash, using quoted/reference hot-tier rates:** for **1,000 decimal GB held throughout a month**, OneLake Hot is **$23.00**, Azure Blob Hot LRS **$20.80** and AWS S3 Standard **$23.00**. Azure's $0.0208/GB is from a Microsoft *illustrative/fictitious pricing example*, not a verified live East US quote; S3 uses the public US East reference. Azure LRS and multi-AZ S3 Standard provide different redundancy. Storage requests, OneLake capacity-backed operations and data transfer remain outside these storage-only figures. [S6][S38][S42][S67][S68]
3. **At the specified workload levels and regional price assumptions, Fabric F4 is not automatically cheaper.** Modeled monthly totals for light / moderate / heavy workload are **Fabric F4 PAYG $688.60 / $688.60 / $688.60**, **Fabric F4 reserved-equivalent $475.67 / $475.67 / $475.67**, **separate Azure $283.60 / $553.96 / $1,178.52**, and **AWS-native $199.20 / $432.60 / $975.80**. These are arithmetical simulations, **not verified price quotes or performance-equivalent benchmarks**; the heavy scenario especially may exceed F4's acceptable SQL/BI responsiveness. Storage, reporting and engine assumptions are explicit in Section 3. [S4][S5][S8][S10][S40][S41][S63][S65]
4. **The principal economic difference is billing behavior.** F4 is a continuously charged **shared 4-CU capacity**, even when no ETL jobs run unless PAYG is paused. Azure ADF Data Flows, Synapse serverless SQL, AWS Glue and Athena are metered by active compute or data scanned; QuickSight and Power BI add different per-user pricing. **F4 CUs are not four universal CPU cores, and the 8-vCore/64-GB Fabric Spark node is not dedicated to SQL and Power BI.** [S4][S45][S46][S48][S63][S65][S69][S70]
5. **CI/CD and Lakehouse architecture are unchanged:** Git, Fabric deployment pipelines, variable libraries and APIs provide deployment automation for definitions; **underlying Lakehouse data is not copied on deployment**. OneLake Lakehouse adds managed Delta/Spark/SQL capabilities atop logical storage, unlike a general-purpose Blob/S3 bucket. A direct Power BI Pro/PPU versus Fabric F4 performance comparison remains separately documented in Sections 3.4–3.6. [S12][S13][S14][S18][S19][S37][S59]

**Revision note:** This update replaces the earlier two-vendor-hybrid financial scenarios with a reproducible **Fabric vs standalone Azure vs AWS-native** analysis, corrects the storage rates, and shows low/moderate/heavy utilization. The general component-capability and CI/CD comparisons elsewhere in the report remain applicable.

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

## 3. Hardware / compute specifications and a clearer cost comparison

### 3.1 First principle: CU, vCPU, memory, DIU, DPU and DBU mean different things

**Do not translate F4 = 4 CUs into “a 4-core server with X GB RAM.”** A CU is Fabric's multi-workload capacity/billing resource. Spark, Power BI, SQL, pipelines, and other operations draw from that shared capacity. **Spark-specific** documentation maps one CU to two Spark vCores, while **Power BI-specific** documentation calls F4 approximately **0.5 Power BI vCores** in a SKU comparison. These are different workload-specific capacity descriptions, **not additive entitlements** and not a claim about a physical CPU. Likewise, the Fabric Spark pool's **64 GB memory** is the specification of a *medium Spark node*, not RAM reserved for Power BI, Warehouse, or the whole Fabric capacity. [S45][S46][S47][S48]

**vCPU/vCore:** a virtual processor allocation for a chosen VM or worker; equal vCPU counts do not establish equal CPU-generation, frequency, instructions-per-cycle, local I/O, network bandwidth or query speed. **DIU:** ADF Copy's managed combination of CPU, memory and networking without a fixed public physical-core/RAM conversion. **DPU:** AWS Glue's published 4 vCPU + 16 GiB per standard DPU. **DBU:** a Databricks software processing/billing unit *on top of* Azure VM pricing; it is not a CPU core. Serverless Synapse and Athena on-demand do not furnish an exclusive fixed VM to the customer. [S49][S50][S51][S52][S9]

### 3.2 Native end-to-end architecture: exactly which services are included

The three stacks use products from their **own vendor's family only**. External source data is assumed to exist already, with no charge to provision a source application or database. Processing transforms file-based raw/bronze data into curated Parquet/Delta datasets; the BI model covers the same business metrics, though the vendors' BI semantic models are not software-identical. [S1][S24][S18][S63][S65][S75]

| Processing step | **Microsoft Fabric (single F4)** | **Separate Azure-native services (no Fabric)** | **AWS-native services (no Power BI)** |
|---|---|---|---|
| Ingest & orchestrate | **Fabric Data Factory Pipeline + Copy activity** | **Azure Data Factory Pipeline + Copy activity** | **AWS Step Functions Standard + AWS Glue ingestion job** |
| Hot data store | **OneLake Lakehouse (Files + curated Tables)** | **Azure Blob Storage Hot LRS** (raw + curated Parquet files; ADF Parquet sink supported [S78]) | **Amazon S3 Standard** (raw + curated Parquet) |
| Transform | **Fabric Spark notebook**, single medium node | **Azure Data Factory Mapping Data Flow**, *Small* eight-vCore managed Spark execution cluster | **AWS Glue ETL Spark**, four standard DPUs |
| SQL serving/query layer | **Fabric Lakehouse SQL analytics endpoint** over Delta tables (read-only analytics) | **Azure Synapse serverless SQL** querying curated Azure Blob Parquet with OPENROWSET [S79] | **Amazon Athena** querying curated S3 Parquet via Glue Data Catalog |
| BI semantic model and dashboard | **Fabric Power BI**, Pro users; Import or Direct Lake where eligible | **Traditional Power BI Pro**, Import from Synapse serverless | **Amazon QuickSight**, Author/Reader user licenses, Athena-backed direct query or suitably small SPICE aggregates |
| Billing distinction | Pipeline, Spark, SQL, BI operations normally consume **shared F4 CU budget**; hot storage and Pro user licensing are extra | Copy = DIU-hours; Mapping Data Flow = vCore-hours; SQL = TB processed; storage and Pro licenses separately | Glue = DPU-hours; Athena = TB scanned; QuickSight = per user, S3 storage separately; Step Functions transitions |

**Why ADF Mapping Data Flow instead of Azure Databricks?** For this *fully native-family, independently verifiable-price* scenario, Mapping Data Flows are Azure-managed Spark clusters and have an explicit Microsoft-published **eight-vCore small configuration** and a **$0.274/vCore-hour general-purpose example rate**. Azure Databricks remains a valid Azure-sold specialist alternative for a Python/Scala-centric engineering team, but it changes the infrastructure and pricing assumptions and is not silently mixed into this scenario. Azure Synapse Spark is also natively Azure, but **its documented minimum instantiated cluster is three nodes** (one head plus two workers), so modeling it as a two-node cluster would materially understate its resource footprint. An independently sized Synapse Spark pilot is preferable if full Spark programming parity is required. [S63][S64][S75][S76][S77]

#### Hardware and billing transparency

| Service/configuration | Configured compute / memory | Price input used | Important qualification |
|---|---|---|---|
| **Fabric F4 shared capacity** | **4 CUs**; not a dedicated four-vCPU machine | **$0.72/F4-hour**, **$525.60 at 730 hours**, or **$312.67/month reservation equivalent** | Uses previously documented secondary Sep-2026 retail reference; regional contract quote required. All Fabric workloads draw from same capacity. [S4][S5][S70] |
| **Fabric single-medium-node Spark session** | **8 Spark vCores / 64 GB memory**; single-node driver + executor share resources | Inside F4 **4-CU** budget with capacity-billed Spark selected | Not 64 GB available to Power BI; Spark autoscale *separate billing* is switched **off** for this model. [S47][S53][S70] |
| **Azure ADF Copy** | **4 DIUs** selected for copying; DIUs do not provide a disclosed CPU/RAM-to-DIU equivalence | **$0.25/DIU-hour**; four DIUs cost **$1 per copy-active hour** | No claim of guaranteed same copy throughput as Fabric. [S8][S49][S63] |
| **Azure ADF Mapping Data Flow — Small** | **4 driver vCores + 4 worker vCores = 8 total**; exact RAM allocation not stated in selected public specification | **8 × $0.274 = $2.192 per active cluster-hour** (general-purpose rate) | The driver consumes half the cores; **not** eight executor vCores. Startup/TTL must be counted in billed hours. [S63][S64][S75] |
| **AWS Glue ingestion job** | **2 DPUs = 8 vCPUs / 32 GiB** | **2 × $0.44 = $0.88 per active hour** | Ingestion is accounted for, not quietly assumed free. [S40][S51] |
| **AWS Glue transformation job** | **4 DPUs = 16 vCPUs / 64 GiB** | **4 × $0.44 = $1.76 per active hour** | More nominal vCPUs than either Fabric single-node Spark or ADF's Small Data Flow; not a performance match. [S40][S51] |
| **Fabric SQL endpoint / Power BI** | Managed shared F4 compute; F4's **3-GB semantic model** and **1-GB per-query** model memory guardrails differ from Spark memory | Inside capacity, assuming no paid overage, scaling, or separately billed autoscale | Unlike Athena or serverless SQL there is no per-TB scan cost line, but SQL/BI operations consume CUs and may contend or throttle. [S48][S69][S70] |
| **Azure Synapse serverless SQL** | **Managed serverless**: no dedicated user VM CPU/RAM SKU | **$5 per TB processed** (planning reference) | TB scanned/processed can be much greater than stored TB. [S11][S39][S71] |
| **Amazon Athena on-demand** | **Managed serverless**: no dedicated user VM CPU/RAM SKU | **$5 per TB scanned** | Projection, compression, partitioning and caching influence bills/performance. [S41][S73] |
| **Fabric / Azure / AWS storage** | Storage; *no processing CPU/RAM included* | **OneLake Hot $0.023/GB-month; Blob Hot LRS $0.0208; S3 Standard $0.023** | 1,000 GB held; Azure rate is a Microsoft **example**, not a binding East US quote. Redundancy differs; requests/transfer not modeled. [S6][S42][S67][S68][S74] |
| **Azure and Fabric Power BI Pro** | Microsoft-managed BI query engine; no fixed per-user VM allocated | **$14 × 10 users = $140/month** paid annually | Ten Pro licenses even with F4; existing E5 license entitlements could reduce *incremental* expense. [S7][S10][S72] |
| **AWS QuickSight** | Managed reporting service; no dedicated per-user CPU/RAM | **2 Authors × $24 + 8 Readers × $3 = $72/month** | Basic Author/Reader, not Pro/Q features; two Authors include **20 GB SPICE capacity** together, if using SPICE. [S65] |
| **AWS Step Functions Standard** | Managed orchestration; CPU/RAM not allocated to user | **$0.000025 per state transition**, with **4,000 free transitions/month** | 120 transitions/month modeled, so **$0 incremental** under ordinary applicable free-tier conditions. [S66] |

### 3.3 Reproducible, native-family monthly cost simulations

#### Common workload assumptions and accounting rules

All scenarios assume **1,000 GB of total active data stored** per vendor, including raw and curated data (no uncounted second full copy), 30 scheduled daily pipeline invocations with **4 orchestration activity/state transitions each**, one batch-ingestion component, one transformation component, a serverless SQL/read layer and **10 dashboard users** (**2 report authors, 8 readers**). The BI content fits a **≤1-GB Power BI Import semantic model** or comparably small QuickSight SPICE aggregate; this is not a guarantee that 1 TB can be imported into either BI service. Query scan totals represent all analyst/report reads **combined**, so we do not double-count BI scans. [S10][S48][S58][S65]

**Runtime assumption:** source data and query work are nominally the same business workload; ingestion hours, transformation cluster hours and SQL-scanned TB **are assumed**, not measured or proven equal across engines. All hourly costs are charged based on each vendor's **own selected compute configuration**. No source-system cost, intercloud egress, raw-file upload bandwidth, backups/replication extras, request overages, private networking, alerts/logging, operational labor or taxes are charged in the base case. Storage cost is **$23.00 OneLake, $20.80 Azure Blob and $23.00 S3** (not the old parity-placeholder $23 for all). [S6][S38][S40][S42][S63][S65][S67]

| Simulation input | **Light** | **Moderate** | **Heavy / stress test** |
|---|---:|---:|---:|
| Hot data retained | 1,000 GB | 1,000 GB | 1,000 GB |
| Daily pipeline runs | 30/month | 30/month | 30/month |
| Ingestion runtime per month | 10 h | 30 h | 60 h |
| Transformation compute-cluster runtime per month | 40 h | 120 h | 300 h |
| Analytical SQL data scanned per month (Azure, AWS) | 5 TB | 20 TB | 60 TB |
| Report authors / readers | 2 / 8 | 2 / 8 | 2 / 8 |
| Fabric F4 assumed paid availability | 730 h | 730 h | 730 h |

#### Component-by-component costs (USD per month)

| Item | **Fabric F4 PAYG** (all scenarios) | **Azure light** | **AWS light** | **Azure moderate** | **AWS moderate** | **Azure heavy** | **AWS heavy** |
|---|---:|---:|---:|---:|---:|---:|---:|
| Fabric 4-CU capacity, 730h | $525.60 | — | — | — | — | — | — |
| Ingestion compute, 4 DIU Azure / 2 DPU AWS | Included in CU | $10.00 | $8.80 | $30.00 | $26.40 | $60.00 | $52.80 |
| Transform, 8-vCore Azure / 4-DPU AWS | Included in CU | $87.68 | $70.40 | $263.04 | $211.20 | $657.60 | $528.00 |
| SQL querying (Synapse/Athena) | Included in CU | $25.00 | $25.00 | $100.00 | $100.00 | $300.00 | $300.00 |
| Pipeline orchestrator | Included in CU | $0.12 | $0.00* | $0.12 | $0.00* | $0.12 | $0.00* |
| **Hot storage: actual selected tier** | **$23.00** | **$20.80** | **$23.00** | **$20.80** | **$23.00** | **$20.80** | **$23.00** |
| **Native-family BI licenses** | **$140.00 Pro** | **$140.00 Pro** | **$72.00 QuickSight** | **$140.00 Pro** | **$72.00 QuickSight** | **$140.00 Pro** | **$72.00 QuickSight** |
| **TOTAL, estimated per month** | **$688.60** | **$283.60** | **$199.20** | **$553.96** | **$432.60** | **$1,178.52** | **$975.80** |

*AWS Step Functions: 30 × 4 = 120 state transitions, covered by the documented 4,000 free transitions/month if available; without that allowance the cost would be **$0.003** (less than one cent), immaterial after rounding. Azure ADF orchestration: 120 × ($1/1,000) = **$0.12**. Detailed published usage meters: [S8][S40][S41][S63][S65][S66][S73].

A **Fabric F4 one-year reserved-capacity equivalent** substitutes **$312.67** for **$525.60**, giving **$475.67/month** in *all three scenarios*, but it is a commitment; it does not mean Spark, SQL, and BI can all execute at required speeds on F4. Unlike Azure and AWS, Fabric still accrues 24/7 capacity charges when workloads are idle, unless PAYG capacity is paused; if paused, users cannot expect uninterrupted BI availability. [S4][S5][S35][S70]

#### Total-cost visual and interpretation

| Scenario | **Fabric PAYG** | **Fabric reserved eq.** | **Azure-only** | **AWS-only** | Lowest *modeled* total |
|---|---:|---:|---:|---:|---|
| Light | $688.60 | $475.67 | $283.60 | **$199.20** | AWS |
| Moderate | $688.60 | $475.67 | $553.96 | **$432.60** | AWS |
| Heavy / stress test | $688.60† | **$475.67†** | $1,178.52 | $975.80 | Fabric† |

†**Feasibility not established**: the Fabric costs remain flat only if the same F4 supplies enough total and *peak* CU compute, query responsiveness and concurrency, without overage, opt-in Spark autoscale billing, upgrading to F8+, or extending processing beyond SLA. Heavy is deliberately a **stress case**, not a claimed achievable F4 production benchmark. AWS and Azure clusters have different driver/executor allocations, so don't infer runtime equality from virtual core counts. [S35][S36][S48][S63][S64][S70]

**Interpretation:** For light usage, the pay-as-you-go F4 stack is substantially more expensive than low-utilization, metered Azure or AWS engines; under the moderate assumptions, F4 PAYG **also** costs more than the separately metered Azure and AWS stacks, although the F4 *reservation equivalent* is cheaper than Azure and slightly costlier than AWS. For the *hypothetical heavy* run, fixed-capacity economics favor Fabric **only if** F4 can meet the workload's CPU, memory, SQL, dashboard and concurrency requirements. This report **does not establish any real performance-normalized savings**. [S4][S35][S36][S63][S65]

#### Line-item formulas (repeatable with your real invoices)

Let `I = ingest compute hours`, `T = transformation cluster hours`, `Q = TB scanned`, and assume 30 pipeline runs × four activities/transitions. USD/month:

- **Fabric F4 PAYG:** `730 × $0.72 + 1,000 × $0.023 + 10 × $14 = $688.60`, *conditional on measured CU and SLA fit*. **Reserved equivalent:** replace `$525.60` with `$312.67`, giving `$475.67`. [S4][S5][S6][S10]
- **Azure native:** `1,000×$0.0208 + (4 DIU×$0.25×I) + (8 vCore×$0.274×T) + ($5×Q) + (120×$1/1,000) + (10×$14)` = **`$160.92 + $1.00×I + $2.192×T + $5×Q`**. [S8][S10][S11][S63][S67]
- **AWS native:** `1,000×$0.023 + (2 DPU×$0.44×I) + (4 DPU×$0.44×T) + ($5×Q) + (2×$24 + 8×$3) + Step Functions (free-tier assumption)` = **`$95.00 + $0.88×I + $1.76×T + $5×Q`**. [S40][S41][S42][S65][S66]

**Cost-break-even sensitivity:** Hold ingestion at **I = 30 h** and scanned data at **Q = 20 TB**, varying only transformation active hours **T**. Azure's modeled total becomes `$290.92 + $2.192×T`, AWS's `$221.40 + $1.76×T`, against F4 **$688.60 PAYG** or **$475.67 reserved equivalent**. Hence:

| Which F4 tariff? | Azure transform-hours/month to reach F4 cost | AWS transform-hours/month to reach F4 cost |
|---|---:|---:|
| F4 PAYG $688.60 | **181.4 h** | **265.5 h** |
| F4 reserved-equivalent $475.67 | **84.3 h** | **144.5 h** |

These are *billing crossover points only*, not throughput or performance equivalence. If a more expensive Azure or AWS configuration finishes the job faster, the hour assumptions would change; if F4 needs F8 or autoscale, the thresholds change. Existing Power BI E5 entitlements, QuickSight Pro selections, or SPICE beyond the two Authors' included 20 GB also change the numbers. [S4][S35][S36][S40][S65][S70]

#### Fabric capacity-feasibility calculation — not a fabricated benchmark

A fully available F4 provides **4 CU × 730 h = 2,920 nominal CU-hours** in the reference month. For illustration, a single 8-vCore Fabric Spark node mapped to 4 CUs for `T` runtime hours uses `4×T` CU-hours; a one-resource Fabric Copy activity uses approximately **1.5 CU-hours per active copy hour**, and **120 non-copy pipeline activities × 0.0056 CU-hours = 0.672 CU-hours**. These ratios are Microsoft-documented meters, but your actual copy throughput resources may be more than one. This leaves the **SQL endpoint, Power BI, OneLake read/write transactions and any other workloads** unmeasured. [S53][S69][S70][S38]

| Scenario | Spark estimated CU-h | Copy estimated CU-h | 120 activity CU-h | Nominal CU-h left before SQL/BI/storage transactions |
|---|---:|---:|---:|---:|
| Light | 160 | 15 | 0.672 | **2,744.328** |
| Moderate | 480 | 45 | 0.672 | **2,394.328** |
| Heavy | 1,200 | 90 | 0.672 | **1,629.328** |

Do not interpret unused monthly CU-hours as proof that F4 can accommodate the **simultaneous** workload. Microsoft smooths background work across time, evaluates capacity in **30-second windows**, and applies overload/throttling. To prove feasibility, use Fabric Capacity Metrics: per-operation CU seconds, peak utilization, queued jobs, DAX p95, query CPU, end-to-end freshness, and job failures. Fabric *opt-in* Spark autoscale billing consumes separately charged serverless CUs and must be added if enabled. [S35][S36][S70]

**Why there is no single "equal hardware" benchmark:** The Fabric example uses 8 Spark vCores / 64 GB on a constrained shared F4 capacity, ADF Data Flow uses 8 vCores split evenly between driver/worker with unquoted RAM, and Glue uses 16 vCPUs / 64 GiB, including its own driver/worker overhead. Serverless Athena/Synapse SQL and managed Power BI/QuickSight do not publish equivalent dedicated per-tenant hardware. A real cost-efficiency ranking requires matched **rows/GB transformed per hour, p95 dashboard latency, concurrency, data freshness, and query cost** from executions, not just listed hourly prices. [S47][S48][S51][S64][S70]

### 3.4 Traditional Power BI vs Fabric F4 — technical capabilities and licensing

**Traditional Power BI is the same BI product** when it runs within Fabric; the differences concern **workspace capacity, semantic-model limits, storage/refresh mode, and availability of Fabric engineering workloads**, not a different visualization engine. A Power BI Pro workspace uses shared Power BI infrastructure, PPU enables Premium-type capabilities per user, and F4 is an organizational **dedicated-capacity allocation** shared with other Fabric workloads. [S1][S7][S57]

| BI decision dimension | **Power BI Pro** (traditional) | **Power BI PPU** (traditional) | **Power BI + Fabric F4** |
|---|---|---|---|
| 10-user cost / month, BI licenses only | **$140** | **$240** | **$140 Pro** *plus* F4 **$525.60 PAYG** or **$312.67 reserved**, and storage if using OneLake |
| Workspace compute | Microsoft-managed **shared** Power BI service; vCPU/RAM not disclosed per tenant | Microsoft-managed **shared Premium-per-user**; vCPU/RAM not disclosed per user | **4 CU shared F4 capacity**; Microsoft lists **0.5 Power BI vCore reference equivalence**, not a dedicated 0.5-core VM |
| Max semantic-model size / memory bound | **1 GB** | **100 GB** semantic-model size | **3 GB max model memory** (and **1 GB/query** limit), may require more headroom for processing |
| Scheduled refresh per semantic model | **8/day** | **48/day** | **48/day** (SKU/feature conditions apply) |
| Native **Direct Lake** semantic model | **No** | **No** without Fabric capacity | **Yes**, requires OneLake Delta-backed source, with F4 guardrails |
| F4 Direct Lake extra guardrails | N/A | N/A | **10 GB maximum model size on OneLake**, and **300 million rows/table**, among other file/row-group limits |
| Power BI user consumption | Pro/PPU per user | PPU per PPU workspace consumer | **Pro/PPU needed for F4 consumers**; free viewing normally only on **F64+** |
| Other Fabric compute: Spark / pipelines / Lakehouse / Warehouse | **Not provided** by Pro license alone | **Not provided** by PPU license alone | **Available**, competing for F4 CUs and with item-specific restrictions |
| Best-fit use case | Team reporting on a relatively small imported model | BI-heavy teams needing large imported models and Premium features | Integrated ETL + lakehouse + BI; models within F4 memory and acceptable peak concurrency |

**Evidence:** Microsoft's Power BI licensing guidance gives 1-GB Pro, 100-GB PPU, 8/day and 48/day refresh, and shared versus dedicated-capacity status; the Fabric capacity documentation gives F4's specific BI memory and Direct Lake guardrails. PPU's 100 GB is an **upper supported semantic-model size**, not RAM or dedicated CPU awarded to a user. [S48][S57][S58][S59]

**Important corrective implication:** **A 100-GB-capable PPU semantic model is not an F4-sized Power BI model**. F4's **3-GB** model-memory upper bound can make F4 less suitable than PPU for a large, BI-only workload, even though Fabric includes many more engineering features. PPU is not a substitute for a Lakehouse/Spark platform. [S48][S57]

### 3.5 Power BI performance — what is actually faster, and under which conditions?

**No direct published “Fabric F4 is X% faster than Power BI Pro/PPU” independent matched benchmark was established for this report.** Microsoft publishes **architectural guidance and SKU resource guardrails**, not a universal query-latency league table, and this report has not executed comparable BI workloads. The strongest supportable comparisons are by **storage mode and workload contention**. [S48][S59][S60]

| Report/query path | Source and query-processing architecture | Expected comparative behavior **according to Microsoft guidance** | Principal risk / requirement |
|---|---|---|---|
| **Traditional Pro or PPU — Import** | Power Query refresh materializes a compressed **VertiPaq** semantic model; visuals execute against imported in-memory model | **Usually excellent interactive latency for a warm, appropriately sized model**, often better than DirectQuery; however, fresh source data waits for imports/refresh | Refresh CPU/memory and refresh window; 1-GB Pro or 100-GB PPU model-size limits, source gateway or load time. [S57][S59][S60] |
| **Fabric F4 — Import** | **Same VertiPaq/DAX engine** as traditional Power BI Import; BI workload runs on capacity shared with Fabric ETL and SQL | **No inherent reason for automatic speedup** versus a well-sized Pro/PPU Import model. It can win or lose depending on capacity load, model design, cache, and size | F4's **3-GB** model bound, **1-GB/query** memory limit, and contention from simultaneous Spark/pipeline jobs. [S48][S57][S60] |
| **Fabric F4 — Direct Lake** | Model references **Delta/Parquet tables in OneLake**. Data pages load into VertiPaq on demand; **framing refresh** updates references instead of re-importing a copy | Microsoft says query performance is **comparable to Import** for suitable data and **normally outperforms DirectQuery**; metadata-only framing can take **seconds**, avoiding large Import reloads | **Cold-page loading**, data layout/Delta optimization, file/row-group counts, **3-GB** memory resource and **10-GB** OneLake model guardrail on F4. Direct Lake on SQL can fall back to DirectQuery if enabled. [S48][S59] |
| **Traditional Pro/PPU or Fabric F4 — DirectQuery** | BI generates SQL/query requests against an external source, Warehouse or SQL analytics endpoint per interaction | **Frequently slower than Import / Direct Lake** because upstream query/network time is added; a strong upstream data engine can still make it good | Concurrency, source indexing/pruning, network, DirectQuery limits, and Fabric SQL engine competition for CUs when source is inside F4. [S59][S60] |

**Clarification of “performance”:**

- **Visual responsiveness / DAX query latency:** an end-user interactive KPI; Import versus Direct Lake is often *comparable* when hot, but F4 can be worse under Spark/SQL contention. No fixed ms or speedup should be invented. [S59][S60]
- **Data freshness:** Direct Lake can avoid a costly data-copy Import refresh (it still requires Delta commits and current metadata/framing), which can meaningfully shorten **end-to-end source-to-report freshness** even if a warm DAX query is not faster. [S59]
- **Batch ETL processing throughput:** traditional Power BI Pro/PPU is not a substitute for Fabric's Spark/warehouse ingestion and transforms. Compare F4 Spark against a real Databricks/Glue cluster instead of a Power BI license. [S1][S45][S51]
- **Concurrency/reliability:** Microsoft's F4 semantic-model guardrails include **five DirectQuery connections per model**, **one DirectQuery parallelism**, and **two live connections/second** in the documented SKU table; these are distinct constraints, not a universal “five users maximum.” ETL/background workloads can create CU pressure and throttling. [S48][S35]

### 3.6 Controlled benchmark needed to make an evidence-backed performance verdict

A fair test should compare **the same data, DAX measures, transformations, model relationships, report pages, security rules and concurrency**, and explicitly name *storage mode* and *SKU*. Follow Microsoft's Performance Analyzer and Fabric Capacity Metrics documentation. [S60][S36][S61]

1. **Baseline three to four configurations:** (A) Pro **Import**; (B) PPU **Import** if available; (C) F4 **Import**; (D) F4 **Direct Lake** over a Gold Delta table. Add F4 DirectQuery only if it is an actual candidate. Use a model that fits **all** compared SKU guardrails; run an additional **larger-data** experiment separately when comparing PPU to F4. [S57][S59]
2. **Lock workload and test conditions:** identical ~1M/10M/100M-row datasets *where feasible*, identical DAX/query definitions, row-level security, refresh schedule and data source region, and report visual count. Prevent test jobs from accidentally sharing a cache between runs. Include both cold-start and warm-cache observations. The dataset sizes here are **proposed tests, not empirical evidence**. [S48][S59][S60]
3. **Measure interactive latency:** export Power BI Performance Analyzer visual durations, including DAX query duration and visual display time, for **at least 30 repeat measurements per case**. Report **p50, p95, p99**, first-visit/warm medians, failure rate and model size. Record if UI or network time dominates. Thirty repetitions are a proposed minimum protocol, not a Microsoft-defined statistical standard. [S60]
4. **Measure multiuser concurrency:** replay a scripted load with **1, 5, 10, and 20 simulated active users**, or a representative production profile; collect p95 report response and model/source-query duration, queue time, errors and throttling. F4 must be evaluated while **representative pipelines/notebooks are running concurrently**, as well as alone. These concurrency points are **test scenarios, not published capacity limits**. [S35][S36][S48]
5. **Measure freshness and ETL separately:** compare **end-to-end time from source update to correct report visual**, Import refresh CPU/wall time, Direct Lake framing delay, SQL endpoint performance, and Spark ETL duration. Do not declare F4 “faster” because Direct Lake has cheaper *refresh* if the question is *warm visual click latency*. [S59]
6. **Normalize economics at the required SLA:** collect Fabric **CU-seconds** by operation and 30-second peaks, actual Azure ADF **data-flow vCore-hours**, Azure DIU-hours, Synapse TB processed, AWS Glue **DPU-hours**, Athena TB scanned, vendor-native BI user licenses, and storage bills. Count pass/fail against user-set service objectives, e.g. **target p95 visual latency ≤ 3 s** and **no throttling in peak test** (illustrative thresholds, **not an industry claim**). [S36][S60][S61]

**Conditional recommendation for this F4 workspace:** For a **small BI-only department** with existing cleaned data and modest refresh needs, **traditional Pro at $14/user/month** is usually the cleaner cost architecture, provided the data model stays under 1 GB and the shared service meets performance requirements. If the team requires **very large imported semantic models or PPU features**, evaluate PPU at $24/user/month before assuming F4 will be faster. If the organization needs **Fabric ETL, Spark, Lakehouse and Direct Lake near-source freshness in one platform**, F4 is attractive *only after* demonstrating acceptable concurrency, memory headroom, and end-to-end SLA. [S10][S48][S57][S59]

### 3.7 Model boundaries and exact savings calculation

To make an **actual** savings claim calculate:

`(observed baseline monthly cost of matched workload − observed Fabric monthly cost of matched workload) / observed baseline monthly cost of matched workload`.

Include Azure/AWS transfer, requests, storage redundancy/transactions, BI user licenses, source databases, ADF Data Flow/SSIS and AWS orchestration where actually used, gateways, CI/CD operations, reservations, engineer labor, and F4 overage/scale-up if required. A capacity can be paused under PAYG when unused (subject to billing/operations), while reservations are financially committed. For reference, 4 CUs × 730 h × 3,600 s/h = **10,512,000 nominal CU-seconds** in the illustrated month; Microsoft's bursting/smoothing/throttling rules apply. A F4 mirrored-database replica storage allowance is **not** free general OneLake storage. [S4][S7][S35][S36][S38][S57]

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

**Recommended initial direction: retain F4 for a measured proof of value, not a forced all-service replacement; compare Pro-only and PPU separately for BI-only workloads.** Put representative ingestion, a low-code Dataflow, one Spark notebook, Lakehouse Delta table, optional warehouse SQL, and a Power BI semantic model on the same capacity. Record CU consumption, runtimes, query response time, refresh success, peak BI concurrency, and any throttling in Capacity Metrics. F4 is particularly useful to test whether unified developer workflow and lower operational overhead matter more than raw compute price. [S1][S34][S35][S36]

**Workload boundaries:** Keep existing Azure Storage if it already serves other consumers; shortcut it before paying to duplicate raw data. Keep Azure Data Factory when a connector, SSIS migration, managed integration runtime, or networking requirement is not covered appropriately by Fabric. Retain Azure Databricks or Azure ML where separately scaled high-intensity Spark/ML is required. Treat Fabric's Data Agent as a governed **data Q&A** interface—not a substitute for arbitrary enterprise RAG pipelines, custom vector search, or model-serving application architecture. [S20][S24][S25][S27][S32]

**CI/CD minimum viable pattern:** Git-backed Dev workspace → required PR approvals/checks → Fabric native Deployment Pipeline → Test execution and data checks → Production promotion; manage environment configuration through Variable Library and deployment rules, then separately validate operational data, scheduling, gateway connections, refresh credentials and roles. Set up separate production capacity if isolation is a hard SLA. [S12][S13][S14][S15][S16][S37]

**Finance decision gate:** Use the report's three vendor-native scenarios as transparent assumptions, not quotes. Export actual Azure/AWS/vendor bills and Fabric Capacity Metrics, build a line-item cost table at identical data volume and service quality, then recalculate with existing enterprise Power BI licenses, real storage retention, egress and required 24×7 availability. Only accept a cost-savings claim when **F4 throughput and reliability are demonstrated under the exact benchmark**. [S4][S7][S35][S36]

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

[S63] Microsoft Learn — [Azure Data Factory pricing: Copy at $0.25/DIU-hour; general-purpose Mapping Data Flow at $0.274/vCore-hour; $1/1,000 orchestration runs](https://learn.microsoft.com/en-au/azure/data-factory/pricing)  
[S64] Microsoft Learn — [ADF Mapping Data Flow cluster sizing: Small = 4 driver + 4 worker cores](https://learn.microsoft.com/en-us/azure/data-factory/concepts-integration-runtime-performance)  
[S65] AWS — [Amazon QuickSight pricing: Authors $24, Readers $3 and 10 GB included SPICE per Author](https://aws.amazon.com/quick/quicksight/pricing/)  
[S66] AWS — [Step Functions Standard: $0.000025 per transition and 4,000 free transitions/month](https://aws.amazon.com/step-functions/pricing/)  
[S67] Microsoft Learn — [Azure Blob Storage cost estimate; $0.0208 Hot LRS is explicitly a **fictitious sample**, not a live price quote](https://learn.microsoft.com/en-us/azure/storage/blobs/blob-storage-estimate-costs)  
[S68] Microsoft Learn — [OneLake tiered storage: $0.023/GB Hot in documented example](https://learn.microsoft.com/en-us/fabric/onelake/onelake-storage-tiers)  
[S69] Microsoft Learn — [Fabric Data Factory pipeline meters: 1.5 CU-hours per copy resource-hour; 0.0056 CU-hours per non-copy activity run](https://learn.microsoft.com/en-us/fabric/data-factory/pricing-pipelines)  
[S70] Microsoft Learn — [Fabric Spark CU billing: 2 Spark vCores per CU; capacity billed vs opt-in autoscale billing](https://learn.microsoft.com/en-us/fabric/data-engineering/billing-capacity-management-for-spark)  
[S71] Microsoft Learn — [Synapse serverless SQL: billed by data processed](https://learn.microsoft.com/en-us/azure/synapse-analytics/sql/data-processed)  
[S72] Microsoft — [Power BI current Pro/PPU pricing: $14/$24 per user monthly paid annually](https://www.microsoft.com/en-us/power-platform/products/power-bi/pricing)  
[S73] AWS — [Athena scan-based on-demand query pricing](https://aws.amazon.com/athena/pricing/)  
[S74] AWS — [S3 pricing, region/class/transaction dependent](https://aws.amazon.com/s3/pricing/)  
[S75] Microsoft Learn — [Azure Data Factory Mapping Data Flows execute on Microsoft-managed Spark clusters](https://learn.microsoft.com/en-us/azure/data-factory/concepts-data-flow-overview)  
[S76] Microsoft Learn — [Azure Synapse Spark minimum: head node plus two workers (3 nodes)](https://learn.microsoft.com/en-us/azure/synapse-analytics/spark/apache-spark-pool-configurations)  
[S77] Microsoft Learn — [Synapse Spark minimum cluster / pricing alternative via Azure-native Spark](https://learn.microsoft.com/en-us/azure/synapse-analytics/plan-manage-costs)  
[S78] Microsoft Learn — [ADF Mapping Data Flow supports reading and writing Parquet in Azure Blob](https://learn.microsoft.com/en-us/azure/data-factory/format-parquet)  
[S79] Microsoft Learn — [Synapse serverless SQL queries Parquet in Azure storage via OPENROWSET](https://learn.microsoft.com/en-us/azure/synapse-analytics/sql/query-parquet-files)  

### Confidence and limitations

**High confidence:** Native architecture, unit definitions, Power BI/QuickSight public list licensing, F4 shared-CU semantics, and conditional CI/CD and storage distinctions, backed by official documentation. **Medium confidence:** Illustrated public list prices; the F4 regional number is secondary, Azure Blob $0.0208/GB is explicitly a *fictitious Microsoft documentation example*, and serverless per-TB rates are reference prices. Confirm all regional price quotes and discounts. **Low confidence for actual savings or equal-performance claims:** The three scenarios intentionally use hypothetical usage and different engine hardware, not measured same-job run times or a capacity SLA assessment. Heavy F4 feasibility is unverified; no real production savings percentage is claimed.

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


[S45] Microsoft Learn — [Spark job concurrency, F4 base eight Spark vCores and up-to-24-core burst](https://learn.microsoft.com/en-us/fabric/data-engineering/spark-job-concurrency-and-queueing)  
[S46] Microsoft Learn — [Spark starter pools by SKU: F4 eight Spark vCores, one medium starter node](https://learn.microsoft.com/en-us/fabric/data-engineering/configure-starter-pools)  
[S47] Microsoft Learn — [Fabric Spark compute node sizes: Medium 8 vCores, 64 GB RAM](https://learn.microsoft.com/en-us/fabric/data-engineering/spark-compute)  
[S48] Microsoft Learn — [Fabric/Power BI SKU limits, F4 3 GB semantic model, 1 GB/query, DQ connections](https://learn.microsoft.com/en-us/fabric/enterprise/powerbi/service-premium-what-is)  
[S49] Microsoft Learn — [Data Integration Units in ADF Copy activity (CPU, memory, network combination)](https://learn.microsoft.com/en-us/azure/data-factory/copy-activity-performance-features)  
[S50] Microsoft Learn — [Azure Data Factory Integration Runtime is managed, serverless](https://learn.microsoft.com/en-us/azure/data-factory/concepts-integration-runtime)  
[S51] AWS — [AWS Glue price and standard DPU specification: 4 vCPU, 16 GB](https://aws.amazon.com/glue/pricing/)  
[S52] AWS — [Amazon Athena DPU specification and capacity reservations](https://docs.aws.amazon.com/athena/latest/ug/capacity-management.html)  
[S53] Microsoft Learn — [Spark notebook kernels, single-node and core/CU options](https://learn.microsoft.com/en-us/fabric/data-engineering/fabric-notebook-selection-guide)  
[S54] Microsoft Learn — [ADF Mapping Data Flow compute sizes and vCore-hour billing](https://learn.microsoft.com/en-us/azure/data-factory/concepts-integration-runtime-performance)  
[S55] Microsoft Azure — [Azure Databricks pricing table, DS12 v2: 4 vCPU, 28 GiB; tier-dependent DBU+VM rates](https://azure.microsoft.com/en-us/pricing/details/databricks/)  
[S56] Microsoft Learn — [Synapse serverless SQL costs by data processed](https://learn.microsoft.com/en-us/azure/synapse-analytics/sql/data-processed)  
[S57] Microsoft Learn — [Power BI Pro vs PPU vs Fabric capacities: models, refreshes, license, dedicated compute, lifecycle](https://learn.microsoft.com/en-us/power-bi/guidance/powerbi-implementation-planning-subscriptions-licenses-trials)  
[S58] Microsoft Learn — [Power BI service description: 1 GB Pro, 100 GB PPU, refresh limits](https://learn.microsoft.com/en-us/office365/servicedescriptions/power-bi-service-description)  
[S59] Microsoft Learn — [Power BI Direct Lake overview: Direct Lake vs Import vs DirectQuery performance and F4 limits](https://learn.microsoft.com/en-us/power-bi/enterprise/directlake-overview)  
[S60] Microsoft Learn — [Power BI DirectQuery model optimization guidance](https://learn.microsoft.com/en-us/power-bi/guidance/directquery-model-guidance)  
[S61] Microsoft Learn — [Power BI Performance Analyzer: visual, DAX, rendering durations](https://learn.microsoft.com/en-us/power-bi/desktop-performance-analyzer)  
[S62] Microsoft Learn — [Fabric Capacity Metrics app: 14-day compute, 30-day storage, throttling](https://learn.microsoft.com/en-us/fabric/enterprise/metrics-app)  
