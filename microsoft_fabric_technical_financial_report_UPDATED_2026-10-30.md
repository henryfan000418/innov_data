# Microsoft Fabric: Architecture, Competitive Alternatives, CI/CD, and F4 Cost Evaluation

**Research date:** 30 October 2026 (updated comparison)  
**Scope:** Microsoft Fabric workloads, comparable Azure and non-Azure services, practical deployment automation, economics at an existing **F4 / 4-CU capacity**, and the difference between a Fabric Lakehouse and Azure Storage (including Blob and ADLS Gen2).  
**Method:** Vendor technical documentation and published prices are cited throughout. Financial scenarios are transparent *illustrations*, not claims about the organization's measured usage or performance. All amounts are USD, before taxes, enterprise discounts, egress, and ancillary services.

## Executive summary

1. **Fabric is an integrated analytics SaaS platform, not merely a no-code ETL tool.** It unifies ingestion, orchestration, Spark-based engineering, Delta lakehouses, SQL warehousing, event and KQL analytics, machine learning, transactional SQL, semantic modeling, BI, and governance around OneLake and a common capacity system. It reduces integration overhead but does not eliminate specialized products or their feature advantages. [S1][S2][S3]
2. **F4 is a shared compute budget, not four dedicated CPU cores per service.** Microsoft documents 8 Spark vCores (a default 8-vCore/64-GB medium node in a starter pool), while the BI workload has a separate 3-GB semantic-model memory upper bound and a 0.5 Power BI vCore *planning equivalence*; these are not simultaneously dedicated VM guarantees. [S45][S46][S47][S48] Published East US list-price reporting dated September 2026 gives approximately **$0.72/hour = $525.60/month** for a continuously running F4; a one-year reservation corresponds to **$312.67/month** in committed capacity charges. OneLake storage and certain user licenses remain separate. Region, contract, reservations, newer on-demand compute options, and enabled features can change the bill. [S4][S5][S6][S7]
3. **F4 does not unconditionally save money.** In the two specified *hypothetical* workload mixes below, the fully itemized Fabric F4 estimate is **$688.60/month** PAYG or **$475.67/month** with reservation, both including 1,000 GB OneLake hot storage and ten Power BI Pro users. Against **specified two-node Databricks + ADF + Synapse + Power BI** and **four-DPU Glue + Athena + Power BI** examples, F4 costs more for the light-use scenario and less for the busier scenario *only if the jobs and report queries meet the same SLAs on F4*. The specifications and limits differ materially, so no benchmark-grade savings percentage is supported. These are billing illustrations, **not throughput-equivalent benchmarks**. [S4][S5][S6][S7][S8][S9][S10][S11]
4. **Fabric CI/CD is genuine DevOps applied to workspace *definitions*.** Git integration, PR validation, Fabric deployment pipelines, variable libraries, REST APIs, fabric-cicd, CLI, and Terraform support Dev → Test → Prod. **Deployment does not copy lakehouse or warehouse data**; it does not automatically transplant credentials, schedules, or permissions. [S12][S13][S14][S15][S16]
5. **OneLake / Lakehouse is not simply another Blob container.** Azure Blob is a general-purpose storage service; ADLS Gen2 adds hierarchical filesystem semantics atop Blob; OneLake is Fabric's managed, tenant-wide logical lake built on ADLS Gen2; a Lakehouse is a managed *analytics item* over OneLake with Delta tables, Spark, and an automatically provisioned read-only SQL analytics endpoint. Blob and ADLS data can be exposed through shortcuts without wholesale copying. [S17][S18][S19][S20][S21]

**New in the 30 October update:** A hardware/CPU/RAM matrix specifies the actual **F4 Spark (8 vCores/64-GB node)**, **two-VM Databricks (8 vCPUs/56 GiB)**, and **four-DPU AWS Glue (16 vCPUs/64 GiB)** examples and identifies serverless components that have **no disclosed dedicated RAM/CPU**. The previously simplified single-VM Databricks charge is corrected to a two-node driver-worker illustration. A new **Power BI Pro vs PPU vs F4** section shows licensing, model-size constraints, and an **evidence-based but non-numeric** Direct Lake/Import/DirectQuery performance comparison; a controlled benchmark protocol is included. [S45][S46][S48][S51][S57][S59]

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

### 3.2 Specific infrastructure configurations used in the modeled stacks

| Role / product and **exact reference configuration** | CPU capacity | RAM | Billing unit and example price | What that spec **does not** imply |
|---|---|---|---|---|
| **Fabric F4 capacity** (one capacity; 4 CUs; all Fabric workloads share it) | **4 CUs**; mapping for Spark: **8 Spark vCores** at base capacity. Spark bursting may permit up to **24 Spark vCores** with suitable pool configuration/admission | **No publicly advertised universal “F4 RAM” number** | **$0.72/F4 hour** quoted East US 2026 reference; **$525.60 / 730-hour month**; one-year reserve equivalent **$312.67/month** | Neither 8 continuously dedicated CPUs for every service nor 64 GB RAM for Power BI; query workload and ETL compete for the same capacity. [S4][S5][S45][S47] |
| **Fabric F4 Spark starter pool**, *default* one medium node | **8 Spark vCores** on the default medium node; F4 starter pool documented **1 maximum node** | **64 GB** per medium Spark node | Spark execution typically draws from the **same F4 CU budget**, subject to runtime, smoothing and configuration. Not an additional $0.72/hour VM | The Spark node specification does **not** describe the underlying hardware dedicated to Fabric Warehouse or Power BI. Custom pools, overprovisioning and autoscale billing change the picture. [S34][S46][S47][S53] |
| **Fabric F4 Power BI semantic workload** | F4 = **0.5 Power BI vCore** *reference equivalence* in Microsoft licensing table (not a user-allocated CPU) | **3 GB** max semantic-model memory upper bound; **1 GB/query** memory upper bound | Uses **F4 capacity budget** *plus* Pro licenses for authors/viewers on F4 | Not 8 Spark vCores/64 GB allocated to DAX; 3 GB is not free unused host RAM. [S7][S48] |
| **Azure Data Factory Copy on Azure Integration Runtime**, DIU-based | Configurable **4–256 DIUs** in many copy cases; **no stable public vCPU mapping** | **Not published per DIU** | Reference **$0.25/DIU-hour** and **$1/1,000 orchestration activity runs**; a hypothetical 4-DIU copy running one hour costs **$1.00** for movement | A DIU is not 1 vCPU; costs change by activity and runtime; ADF Mapping Data Flows use separately charged vCore-hours. [S8][S49][S54] |
| **Azure Databricks: 2 × Standard_DS12_v2**, one driver plus one worker, *selected reference tier* | **2 × 4 = 8 Azure VM vCPUs total** | **2 × 28 = 56 GiB total** | **Illustrative selected published VM+DBU figure $489.10 / 730h / node ≈ $0.67/node-hour**, or **$1.34/hour for two nodes**; **$160.80 / 120 active hours** | CPU/RAM are split between driver and worker; only part available to executors. DBU rate depends on workload/tier, SKU and region. The published Databricks pricing page contains **multiple pricing tables for the same VM SKU**; this is not a contract quote. [S9][S55] |
| **Azure Synapse serverless SQL pool** | **No committed user VM/vCPU number**; Microsoft runs its serverless query engine | **No user-configurable fixed RAM** | Example reference **$5 per TB processed**: 20 TB = **$100** | Not a dedicated warehouse, and $/TB processed cannot be converted into CPU-hour without actual job measurements. [S11][S39][S56] |
| **AWS Glue Spark: 4 standard DPUs**, selected model | **4 × 4 = 16 vCPU total** | **4 × 16 = 64 GiB total** | Reference **$0.44 / DPU-hour**; **$1.76 per 4-DPU job-hour**; **$211.20 / 120 active hours** | This selection has **twice the nominal CPU count** of the Fabric F4 Spark base allowance; equal 64-GB memory is **not** throughput equality. [S40][S51] |
| **Amazon Athena on-demand SQL** | **No user-dedicated vCPU allocation** in scan-based mode | **No fixed user-allocated RAM** | **$5/TB scanned** reference; 20 TB = **$100** | A different reservation model exposes Athena DPUs (typically 4 vCPUs + 16 GB) and charges separately; *not* used in this model. [S41][S52] |
| **OneLake**, **Azure Blob/ADLS**, or **S3**, model assumes 1,000 GB hot storage | Storage **does not supply data-processing CPU** | Storage **does not supply query RAM** | **$23/1,000 GB-month** placeholder in each stack, from an illustrative OneLake hot-rate example | The non-Fabric **$23 is a parity assumption**, not a regional Blob or S3 price quote. Requests, replicas, transfer, tiers, and caches can change total. [S6][S38][S42] |
| **Traditional Power BI Pro** (10 named users, shared-capacity workspace) | **No dedicated user CPU/vCPU allocation published** | **1-GB maximum published semantic-model size**, not a 1-GB physical VM | **$14/user/month** paid yearly; **10 users = $140/month**, *no separately purchased F4* | Includes reporting service, not the separate cloud ETL/SQL infrastructure; shared tenant performance varies. [S10][S57] |
| **Traditional Power BI Premium Per User (PPU)** (10 named users) | **No dedicated user CPU/vCPU allocation published** | **100-GB published maximum semantic-model size**, not guaranteed 100-GB continuously resident RAM | **$24/user/month** paid yearly; **10 users = $240/month** | PPU is per-user licensing with shared infrastructure, **not an F4 capacity**, and a PPU workspace normally requires PPU access for viewers. [S10][S57] |

**CPU power comparison rule:** **8 Spark vCores / 64 GB** for an F4 Spark *node*, **8 Azure VM vCPUs / 56 GiB** across a *two-node Databricks driver-worker cluster*, and **16 AWS Glue vCPUs / 64 GiB** across *four DPUs* are **published provisioning specifications**, not benchmark-equivalent CPU power. Fabric's capacity is also consumed by other workloads, whereas the separate jobs are independently metered. In particular, the earlier report's single-DS12 Databricks $0.67/h example counted only **one VM**, not the reference two-node driver/worker setup; the revised cost model below uses **two nodes**. [S45][S46][S51][S55]

### 3.3 Rates, user licenses, and monthly infrastructure cost

All examples use **730 hours/month** if 24/7, **1,000 GB billable hot data**, **10 Power BI report users**, published or indicated reference prices, and **USD**. F4 is assumed active all month (PAYG) or committed for a year (reserved). The **10 Pro license cost** is included in every full analytics stack so it is not accidentally attributed as a Fabric-only expense. If these users already receive Pro through Microsoft 365 E5, the incremental licensed cost may be **$0 on all those configurations**. Storage parity and job parameters are assumptions; **not bid quotations**. [S4][S5][S6][S10][S57]

**Power BI only, *without replacing ETL or SQL tools*: (a)** 10 Power BI Pro users = **$140/month**; **(b)** 10 PPU users = **$240/month**. These are valid **BI-only** baselines and are **not** comparable with F4's end-to-end ETL/SQL platform. Power BI Desktop is free to author local reports; publishing/sharing requires appropriate licensing. Existing P1/P2 Premium capacities are a separate legacy arrangement: Microsoft no longer sells new P SKUs and recommends F capacities for new deployments. [S10][S57]

**Fabric all-in reference:** $0.72 × 730 h **$525.60 F4 PAYG** (or **$312.67/month 1-year F4 reservation equivalent**) + $23 OneLake storage + $140 Pro for 10 users = **$688.60/month PAYG** or **$475.67/month reserved equivalent**. Under **F4**, Pro/PPU viewer licenses are still required; **the free-viewer exception normally starts at F64**. [S4][S5][S6][S7][S10]

#### Job-volume assumptions — these determine the alternative stacks' charges

| Modeling input | **L: light** | **B: busier** | Measurement unit |
|---|---:|---:|---|
| Databricks two-node cluster | 120 h | 240 h | Both VM nodes active for each billed hour |
| Azure Data Factory Copy | 100 DIU-h | 300 DIU-h | Used DIUs × copy duration |
| Azure Data Factory orchestration | 2,000 runs | 2,000 runs | Activity-run counter |
| Synapse serverless SQL | 20 TB | 80 TB | Data processed; **not volume stored** |
| AWS Glue | 4 DPUs × 120 h | 4 DPUs × 240 h | DPU-hours; **16 vCPU + 64 GiB provisioned** |
| Amazon Athena | 20 TB | 80 TB | TB scanned |
| Power BI | 10 × Pro | 10 × Pro | Users, annual-billed monthly equivalent |
| Storage | 1,000 GB | 1,000 GB | Placeholder hot data |

**Revised cost table (explicit compute configuration):**

| Item / monthly USD | Fabric **F4 PAYG** | Fabric **F4 reserved** | **Azure L**: 2×DS12v2 + ADF + Synapse + Pro | **AWS L**: 4 Glue DPUs + Athena + Pro | **Azure B** | **AWS B** |
|---|---:|---:|---:|---:|---:|---:|
| Shared F4 capacity | $525.60 | $312.67 | — | — | — | — |
| Databricks **2 × 4-vCPU / 28-GiB nodes** | — | — | $160.80 | — | $321.60 | — |
| Glue **4 × (4 vCPU / 16 GiB) DPUs** | — | — | — | $211.20 | — | $422.40 |
| Azure Data Factory Copy + activities | Included¹ | Included¹ | $25 + $2 = $27 | Not included² | $75 + $2 = $77 | Not included² |
| SQL scan service | Included¹ | Included¹ | $100 Synapse | $100 Athena | $400 Synapse | $400 Athena |
| Hot storage (parity placeholder) | $23 | $23 | $23 | $23 | $23 | $23 |
| **10 traditional Power BI Pro licenses** | $140 | $140 | $140 | $140 | $140 | $140 |
| **Modeled stack total** | **$688.60** | **$475.67** | **$450.80** | **$474.20** | **$961.60** | **$985.40** |

¹ **Unverified feasibility assumption:** all listed ETL, analytical queries and BI can fit F4 without paid autoscale, extra capacity, performance shortfall, or additional features/charges. Fabric refresh and jobs compete for CUs; fixed monthly price is **not a promise of equivalent response time**. [S4][S35][S45]  
² AWS orchestration/copy/networking are **not modeled**; the AWS result is a **partial cost floor**, not the operationally complete price of a replacement platform. The Databricks chosen rate is a *selected reference* within a page with several prices; seek a region/workload-specific quote. Storage for Azure/AWS is only a parity placeholder. [S9][S40][S41][S42]

**Conclusion from the *financial illustration* (not measured savings):** At light use, F4 PAYG is **$237.80 more than Azure L** (52.7% higher) and **$214.40 more than AWS L** (45.2% higher). In busier use, **if F4 actually meets equivalent performance**, its PAYG cost is **$273.00 below Azure B** (28.4% lower), and **$296.80 below AWS B** (30.1% lower). The F4 reservation equivalent is cheaper in both busier examples, but requires the full-year commitment. **The cross-vendor savings percentages must not be presented as achieved ROI.** Rates and hardware specifications alone cannot establish equal performance. [S4][S5][S9][S40][S41]

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
6. **Normalize economics at the required SLA:** collect Fabric **CU-seconds** by operation and 30-second peaks, actual Databricks **two-node hours/DBUs**, Azure DIU-hours, Synapse TB processed, Power BI licenses and storage bills. Count pass/fail against user-set service objectives, e.g. **target p95 visual latency ≤ 3 s** and **no throttling in peak test** (illustrative thresholds, **not an industry claim**). [S36][S60][S61]

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
