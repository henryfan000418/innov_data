# Microsoft Fabric F-SKU Capacity Sizing, Memory, Pricing, and Pause/Resume Cost Optimization

**Research as of:** November 2, 2026  
**Scope:** Microsoft Fabric F2–F1024 capacities, with special attention to the existing F4 (4-CU) baseline.  
**Currency/pricing reference:** USD, US public-reference PAYG rate of **$0.18 per CU-hour**, 730 hours/month. One-year reservation reference of **$938 per CU-year**. These are publicly documented reference prices, **not an organization-specific Azure quote**. Actual SKU price varies by region, contract, discounts, taxes, and time. [1][2][3]

## Executive summary

Microsoft Fabric F-SKUs provide a **shared compute capacity**, not a fixed VM with a single CPU and RAM specification. Higher SKUs increase capacity units (CUs), baseline Fabric Spark vCore budget, Spark pool scalability, and Power BI semantic-model limits. **They do not guarantee a particular number of dedicated physical CPU cores or a pool of universal RAM usable interchangeably by Power BI, Spark, Warehouse, and Dataflows.** [4][5][6]

**Yes, you can stop paying for idle PAYG compute by pausing the Fabric *capacity*.** Fabric does **not** offer a separate on/off billing switch for each workspace: multiple workspaces can share one capacity, and pausing it affects all assigned workspaces. Storage continues to accrue charges while capacity is paused; Fabric content and requests can be unavailable. Pausing can also settle accumulated smoothing/overage charges. [7][8][9][10]

At the reference F4 rate, keeping capacity active for 730 hours costs **$525.60/month** in compute. Running only 8 hours on 22 weekdays (176 hours) would cost **$126.72/month** in normal PAYG capacity charges—a modeled **75.9% reduction in compute cost**. With 1,000 GB OneLake Hot stored at a reference $0.023/GB-month, add **~$23/month in either case**, plus licensing, operations, and any eligible usage settlements. This is a *billing arithmetic example*, not a guarantee that a specific production workload can be paused safely. [1][7][8][9]

## 1. What a CU means—and why Spark RAM must be described separately

- **CU (Capacity Unit):** Fabric's unit of shared, metered processing capacity. SKU `F4` provides 4 CUs; `F64` provides 64 CUs. CU-second consumption is shared by Fabric workloads assigned to a capacity. [1][4]
- **Spark vCore baseline:** Microsoft documents **2 baseline Spark vCores per 1 CU** under standard capacity-billed Spark; this is a Spark workload mapping, not a universal physical CPU specification for the entire service. Other engines have distinct mappings. [5][6]
- **Spark RAM:** A Spark **node** has its own configured size. A Medium node is **8 vCores/64 GB RAM**; Large is **16 vCores/128 GB**. RAM is not a single enterprise-wide allowance shared across Fabric services. [5]
- **Power BI memory:** Upper limits for **one semantic model**, and limits for an individual query, scale differently from Spark. For instance, F4 supports up to **3 GB semantic-model memory** and **1 GB per query**, even though a Medium Spark node has 64 GB. [11]
- **Bursting, smoothing and concurrency:** Spark may use burst capacity above the baseline temporarily. This does not provide indefinitely free extra processing; accumulated consumption is subject to smoothing, backlog, queueing and potentially throttling or billed overages. [6][10]

### Spark node sizes (this is *per node*, not per F-SKU)

| Spark node size | vCores per node | RAM per node | CU-equivalent of running one node at the normal Spark mapping |
|---|---:|---:|---:|
| Small | 4 | 32 GB | 2 CUs |
| Medium | 8 | 64 GB | 4 CUs |
| Large | 16 | 128 GB | 8 CUs |
| X-Large | 32 | 256 GB | 16 CUs |
| XX-Large | 64 | 512 GB | 32 CUs |

Source: Microsoft Fabric Spark compute documentation. The CU-equivalent column is `vCores / 2` and indicates nominal rate of resource consumption, **not a separately purchased VM or a promised per-workspace pool**. Spark sessions use a driver and executor(s); with a single-node configuration the driver and executor divide that node's resources. [5]

## 2. Fabric SKU price and compute comparison

All numbers below are **reference list-price calculations**. `PAYG / month` assumes 730 continuously active hours, not usage-based scheduling. `Reserved equivalent / month` divides the annual commitment by 12; it is **not** automatically reduced when the capacity is paused. [1][2][3][12]

| SKU | CUs | Base Spark vCores | PAYG $/hour | PAYG $/730 h month | 1-year reserved $/month equivalent |
|---|---:|---:|---:|---:|---:|
| F2 | 2 | 4 | $0.36 | $262.80 | $156.33 |
| **F4 (current baseline)** | **4** | **8** | **$0.72** | **$525.60** | **$312.67** |
| F8 | 8 | 16 | $1.44 | $1,051.20 | $625.33 |
| F16 | 16 | 32 | $2.88 | $2,102.40 | $1,250.67 |
| F32 | 32 | 64 | $5.76 | $4,204.80 | $2,501.33 |
| F64 | 64 | 128 | $11.52 | $8,409.60 | $5,002.67 |
| F128 | 128 | 256 | $23.04 | $16,819.20 | $10,005.33 |
| F256 | 256 | 512 | $46.08 | $33,638.40 | $20,010.67 |
| F512 | 512 | 1,024 | $92.16 | $67,276.80 | $40,021.33 |
| F1024 | 1,024 | 2,048 | $184.32 | $134,553.60 | $80,042.67 |

**Pricing formula:** `PAYG per hour = CUs × $0.18`; `continuous monthly = PAYG per hour × 730`; `reservation monthly equivalent = CUs × $938 / 12`. The Microsoft retail pricing page renders region-dependent price cells dynamically; the fixed $0.18 and $938 parameters are supported by Microsoft's reference materials and published 2026 Azure retail-rate transcriptions. Validate your tenant's region and negotiated quote using the Azure pricing calculator before purchasing. [1][2][3]

**Not included in the compute table:** OneLake storage, applicable Power BI Pro/PPU user licenses, networking/egress, optional Spark Autoscale Billing, Fabric capacity overage, and other separately billed services. [1][8][12]

### RAM and concurrency: practical allocations by SKU

This table illustrates **configurable Spark starter-pool maximum Medium-node counts**, **not guaranteed permanent, dedicated total RAM**. Each Medium node has 8 vCores and 64 GB RAM. For F2, Microsoft allows a Medium starter node even though its **baseline** allocation is only four Spark vCores; bursting/smoothing and workload limits therefore matter. Do not infer that all the memory in these nodes is available simultaneously to Power BI or to a single notebook. [5][13]

| SKU | Published baseline Spark vCores | Max Medium starter nodes | Sum of RAM across configured max nodes* |
|---|---:|---:|---:|
| F2 | 4 | 1 | 64 GB* |
| F4 | 8 | 1 | 64 GB* |
| F8 | 16 | 2 | 128 GB* |
| F16 | 32 | 4 | 256 GB* |
| F32 | 64 | 8 | 512 GB* |
| F64 | 128 | 16 | 1,024 GB* |
| F128 | 256 | 32 | 2,048 GB* |
| F256 | 512 | 64 | 4,096 GB* |
| F512 | 1,024 | 128 | 8,192 GB* |
| F1024 | 2,048 | 200 | 12,800 GB* |

`*` Calculated **potential aggregate RAM if all specified nodes are instantiated**. These values are **not a published Fabric SKU total-RAM entitlement**. Starter pools are best-effort; the default maximum is often lower than the configured maximum, and nodes may be temporarily unavailable. Driver/executor overhead, session scheduling and capacity contention reduce usable RAM. At F1024 the documented starter-pool maximum is **200 nodes**, not the 256 nodes that would fit the nominal base vCore calculation; this is why memory does not scale indefinitely with a simple `8 GB/vCore` formula. [5][13]

### Power BI memory limits on the same SKUs

| SKU | Maximum memory per semantic model | Max memory per query | Published Direct Lake maximum model size on OneLake |
|---|---:|---:|---:|
| F2 | 3 GB | 1 GB | 10 GB |
| **F4** | **3 GB** | **1 GB** | **10 GB** |
| F8 | 3 GB | 1 GB | 10 GB |
| F16 | 5 GB | 2 GB | 20 GB |
| F32 | 10 GB | 5 GB | 40 GB |
| F64 | 25 GB | 10 GB | Unlimited† |
| F128 | 50 GB | 10 GB | Unlimited† |
| F256 | 100 GB | 10 GB | Unlimited† |
| F512 | 200 GB | 20 GB | Unlimited† |
| F1024 | 400 GB | 40 GB | Unlimited† |

Source: Microsoft Power BI/Fabric capacity and Direct Lake guardrail tables. **Per-model memory is an upper bound, not a guaranteed usable in-memory dataset size**: refreshes and queries consume additional memory. `Unlimited†` means that particular *on-disk Direct Lake model-size* guardrail is not capped in that table, **not** unlimited query memory, compute, data rows, or performance. This table should not be confused with Power BI model *offline file-format* limits. [11][14]

**Power BI licensing threshold:** On **F2–F32**, consumers of organization-shared Power BI content ordinarily require Pro/PPU or an eligible trial license. At **F64+**, eligible users with Fabric Free licenses can view shared Power BI content, although authors/publishers still require appropriate licenses. This can materially affect the economics of large reporting audiences. [15]

## 3. Can a workspace be switched off to save money?

**Correct unit of control: capacity, not workspace.** A workspace is a logical container for Fabric items and is *assigned to a capacity*. The capacity is the Azure billable compute resource. There is no equivalent to `Stop this individual workspace and save its share of compute` while the shared capacity continues running. [7][16]

| Control | Supported? | Billing effect | Operational consequence |
|---|---|---|---|
| Pause the **PAYG capacity** | Yes | Running compute billing stops (subject to settlement of prior usage) | **All assigned Fabric workspaces** can be unavailable |
| Resume the capacity | Yes | PAYG compute billing resumes | Workspaces' Fabric content becomes available |
| Resize capacity F4 → F2 or F8 | Yes | Future active-hour rate changes | Workload limits and performance change |
| Pause just one workspace while others keep running on same capacity | **No independent workspace-level capacity pause** | No dedicated workspace compute saving | Use separate capacities if independent schedules matter |
| Stop a Spark session or use Spark auto-pause | Yes | Frees Spark resource consumption; doesn't remove an active PAYG F4's fixed capacity rate | Spark jobs/session stop; rest of capacity remains active |
| Pause capacity covered by a **1-year reservation** | Yes | Does **not** cancel reservation commitment; unused reserved coverage is lost unless usable by another matching capacity | Fabric workspaces can be unavailable without corresponding commitment savings |

Sources: [7][8][9][12][16][17].

### Exactly how to pause/resume manually

1. Go to [Azure portal](https://portal.azure.com), search for **Microsoft Fabric**, and open the **Fabric capacity resource** (not the Fabric workspace screen).
2. Select **Pause**, then confirm. Make sure no needed refresh, pipeline, report or SQL workload is active or due to run.
3. When ready, select **Resume**. Billing restarts when capacity resumes; allow time for availability checks.

Required Azure RBAC includes `Microsoft.Fabric/capacities/read`, `write`, `suspend/action`, and `resume/action`. Use a limited-scope custom role where possible. [7]

### Schedule an automatic stop/start

Microsoft documents **Azure Automation runbooks** for scheduling capacity pause/resume and exposes pause/resume REST APIs. A practical **development-only capacity** pattern is to resume at 09:00 and pause at 17:00 Monday–Friday in the capacity's intended local timezone, using a managed identity and minimum required Azure permissions. Set alerts on runbook failures and verify state. [7]

**Operational prerequisites:** Keep production and business-hours-independent jobs on a different always-available capacity; avoid pausing shared resources when users access reports overnight; handle public holidays, daylight saving, approval windows and queued jobs; verify resume before the next job; inspect capacity metrics after each pilot week. **Do not schedule automatic pause on production without a workload inventory and a rollback process.** [7][10][16]

## 4. Savings simulation: 24/7 vs business-hours PAYG

**Assumptions:** $0.18/CU-hour; same SKU when active; 730 h for continuous month; 22 workdays × 8 h = **176 active hours**; no extra active time, no paused-content dependencies, no incremental capacity overage. Storage and user licensing stay outside the compute savings arithmetic. [1][2][7][8]

| SKU | 24/7 compute (730h) | Scheduled compute (176h) | Monthly compute saved |
|---|---:|---:|---:|
| F2 | $262.80 | $63.36 | $199.44 |
| **F4** | **$525.60** | **$126.72** | **$398.88** |
| F8 | $1,051.20 | $253.44 | $797.76 |
| F16 | $2,102.40 | $506.88 | $1,595.52 |
| F32 | $4,204.80 | $1,013.76 | $3,191.04 |
| F64 | $8,409.60 | $2,027.52 | $6,382.08 |
| F128 | $16,819.20 | $4,055.04 | $12,764.16 |

**F4 with 1,000 GB ordinary OneLake hot storage (~$23/month reference):**

| F4 operating mode | Compute | OneLake storage | Total of these two line items only |
|---|---:|---:|---:|
| PAYG 24/7 | $525.60 | $23.00 | **$548.60** |
| PAYG 176 h/month | $126.72 | $23.00 | **$149.72** |
| 1-year reservation, 24/7 | $312.67 | $23.00 | **$335.67** |

Do **not** add a separate storage fee for all Power BI Import models by default: their native semantic-model storage can be covered by the Power BI licensing/capacity model, while normal Lakehouse/Warehouse OneLake storage is charged separately. OneLake storage stays billed when capacity is paused. [8]

### PAYG scheduling vs 1-year reservation break-even

Using the published reference of `$938 per CU-year`, a reservation's average cost is `$938 / 12 ≈ $78.17 per CU-month`, while PAYG costs `$0.18 per CU-hour`. Thus the **simple** average-rate crossover is approximately **434.26 active hours per 730-hour month** (`$78.17 / $0.18`), or **59.5%** of continuous operation. An 8h × 22-day development schedule is only **24.1%** of a 730-hour month, so hourly PAYG plus scheduled pause is much cheaper than a full CU reservation for that isolated capacity. [2][3][12]

The threshold is a **price-only heuristic**, not guaranteed: reservations can offset other eligible matching capacity usage, and performance/service-availability requirements may force longer runtime. **A paused reserved capacity does not create extra refundable savings.** [12]

## 5. Operational and financial exceptions

1. **Pause-time usage settlement:** Fabric smooths some activity across future intervals. On pause, accumulated future usage and overages are settled onto the bill. This can produce a visible utilization spike. **A simple active-hours estimate may understate cost**, especially when heavy batch jobs run immediately before pause. [7][9][10]
2. **OneLake storage survives, access does not necessarily survive:** Storage continues billing when paused. Operations against that capacity can be rejected; data in another active capacity accessed via shortcuts may behave differently. Do not confuse compute pause with archive/delete or zero storage cost. [8]
3. **Optional Spark Autoscale Billing:** This is an opt-in, separate meter using PAYG Spark compute; it is **not** the normal shared-CU billing model. It can increase total charges. [1]
4. **Capacity overage billing:** If enabled, Fabric can bill overages instead of throttling; it does not increase your actual SKU allocation, and charges can exceed a configured threshold. [18]
5. **Other charges persist:** Power BI user licenses, networking/egress, certain items' native storage, other Azure resources and commitments remain independently billable. [1][8][12][15]
6. **Autoscale and concurrency do not imply equal service performance:** Moving F4 → F8 doubles CU entitlement and the base Spark vCore mapping but does not guarantee every pipeline runs exactly twice as fast. Workloads may be bottlenecked by IO, source rate limits, model structure, skew, and query execution. [4][6]

## 6. Decision guide for the current F4 (4-CU) setup

**If F4 hosts dev/test only:** Keep F4 PAYG and pilot an 8-hour weekday schedule. With 176 active hours, modeled compute reduces from $525.60 to $126.72/month. Assign only workloads that tolerate being unavailable outside those hours. Track pause-time usage settlement and test restart reliability before automating. [7][9]

**If F4 hosts production reports, scheduled ETL, Direct Lake dashboards or external consumers:** Do **not** pause it simply because staff leave the office. First verify schedules, refreshes, service principals, downstream apps and availability SLAs. Consider separating nonproduction workloads onto a second, pausable F2/F4 capacity while keeping production always available. [7][16]

**If F4 repeatedly queues/throttles Spark jobs or Power BI:** Use the Fabric Capacity Metrics app to attribute CU consumption to operations, then test F8. F8 increases the base Spark budget from 8 to 16 vCores, but **Power BI per-model memory remains 3 GB**, so moving F4 → F8 does not solve a single large semantic model needing 5–10 GB. F16 and F32 materially change those particular Power BI memory limits. [6][11][13]

**If capacity is almost always on:** Compare PAYG against 1-year CU reservations, remembering the commitment is separate from pause/resume. Test a right-sized reservation quantity if active workloads exist elsewhere that could consume reserved CUs. [12]

### Recommended four-step proof-of-concept

1. **Establish a 2-week baseline:** Export F4 usage by workload, report activity outside office hours, average CU, spikes, throttling, refresh timing, and Azure invoice line items.
2. **Inventory dependencies:** Identify what must work 24/7 (reports, data agents, business processes, alerts, scheduled jobs, SQL/OneLake endpoints) and choose a low-risk development-only capacity to pause.
3. **Run a one-week scheduled-pause test:** Resume before work begins; confirm notebook, pipeline and Power BI operation; pause after jobs settle; record Azure charges and the pause-event consumption spike.
4. **Make decision:** Compare actual active-hour savings + settlement against operational failures, analyst disruption and extra admin cost. Consider moving sporadic development workloads to an independently pausable capacity and reserving steady production compute.

## 7. Source and reproducibility register

**Official Microsoft documentation / Microsoft-authored pricing evidence**

1. Microsoft Azure, [Microsoft Fabric pricing](https://azure.microsoft.com/en-us/pricing/details/microsoft-fabric/) — SKU list, PAYG/reservation billing, optional Spark autoscale, charges excluded.
2. Microsoft Fabric Community, [Announcing Microsoft Fabric capacities are available for purchase](https://community.fabric.microsoft.com/blog/fbc_fabricupdatesblogs/announcing-microsoft-fabric-capacities-are-available-for-purchase/5172353) — $0.18 per CU-hour US West 2 reference price and F-SKU table.
3. Microsoft cost-management pricing reference / corroboration, [Published 2026 Azure Retail Prices API transcription](https://www.datastudios.org/post/power-bi-pro-vs-premium-per-user-vs-fabric-f64-capacity-licenses) — $938 per CU-year and $0.18 per CU-hour; secondary source, verify directly with Azure quote.
4. Microsoft Learn, [Plan your capacity size](https://learn.microsoft.com/en-us/fabric/enterprise/plan-capacity) — CUs and sizing workflow.
5. Microsoft Learn, [Apache Spark compute in Fabric](https://learn.microsoft.com/en-us/fabric/data-engineering/spark-compute) — Spark node vCore/RAM types; driver/executor behavior.
6. Microsoft Learn, [Spark job concurrency and queueing](https://learn.microsoft.com/en-us/fabric/data-engineering/spark-job-concurrency-and-queueing) — baseline vCores and bursting/queue limits.
7. Microsoft Learn, [Pause and resume your Fabric capacity](https://learn.microsoft.com/en-us/fabric/enterprise/pause-resume) — permissions, Azure portal, automation, pause-time settlement.
8. Microsoft Learn, [OneLake consumption](https://learn.microsoft.com/en-us/fabric/onelake/onelake-consumption) — OneLake storage vs transactions and paused-capacity billing.
9. Microsoft Learn, [Monitor a paused capacity](https://learn.microsoft.com/en-us/fabric/enterprise/monitor-paused-capacity) — paused events and settlement visualization.
10. Microsoft Learn, [Understand Fabric capacity throttling and smoothing](https://learn.microsoft.com/en-us/fabric/enterprise/throttling) — smoothing, throttling and pause impact.
11. Microsoft Learn, [Power BI Premium/Fabric SKU limitations](https://learn.microsoft.com/en-us/fabric/enterprise/powerbi/service-premium-what-is) — semantic model RAM and per-query memory.
12. Microsoft Learn, [Save costs with Microsoft Fabric Capacity reservations](https://learn.microsoft.com/en-us/azure/cost-management-billing/reservations/fabric-capacity) — commitment, unused reservations and scope.
13. Microsoft Learn, [Configure and manage Fabric starter pools](https://learn.microsoft.com/en-us/fabric/data-engineering/configure-starter-pools) — per-SKU medium-node count limits.
14. Microsoft Learn, [Direct Lake overview](https://learn.microsoft.com/en-us/fabric/fundamentals/direct-lake-overview) — Direct Lake model guardrails.
15. Microsoft Learn, [Understand Microsoft Fabric licenses and capacity](https://learn.microsoft.com/en-us/fabric/enterprise/licenses) — Power BI licenses and F64 viewer threshold.
16. Microsoft Learn, [Fabric capacity throttling policy](https://learn.microsoft.com/en-us/fabric/enterprise/throttling) — multiple workspaces per capacity and capacity-level throttling.
17. Microsoft Learn, [Scale your Fabric capacity](https://learn.microsoft.com/en-us/fabric/enterprise/scale-capacity) — Azure portal resizing, operational considerations.
18. Microsoft Learn, [Fabric capacity overage](https://learn.microsoft.com/en-us/fabric/enterprise/capacity-overage-overview) — overage billing and thresholds.

**Price and calculation audit:** The reference monthly price assumes `730 h`, not a calendar month's actual number of hours; the schedule assumes precisely `22 × 8 = 176 h`. Rounding to 2 decimals is applied only to displayed monetary amounts. The reservation amount is estimated from a 2026 published Azure Retail Prices API transcription because Microsoft's dynamic region price selectors do not render numeric rates in the static public page. See source 3. No pricing or savings claim is based on observed activity from the organization's Azure tenant.
