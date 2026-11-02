# Microsoft Fabric F4: Pay-As-You-Go vs. Reserved Capacity
## Pause/Resume Behavior, Cost Break-Even, and Operating Recommendation

**Research date:** November 2, 2026  
**Scope:** Microsoft Fabric F-SKU capacity (F4 baseline), usage-based versus reserved billing, operational pause/resume, and F4 financial scenarios  
**Currency:** USD  
**Status:** Decision-support model; prices are planning assumptions, not an Azure contract quote.

---

## Executive summary

**Both PAYG and reservation-covered Fabric capacities can be paused and resumed.** The difference is **what happens financially during the pause**:

- **Pay-as-you-go (PAYG):** the provisioned F-SKU capacity compute meter stops while paused, reducing capacity compute charges for inactive periods. Consumption accumulated before pause can be reconciled and billed at pause time. Storage and other separately metered services continue to cost money. [S1] [S4] [S5]
- **Reserved:** a Fabric reservation is **a financial commitment for eligible CU usage, not a distinct always-on compute resource**. You may pause the underlying F4 capacity, but the **reservation purchase obligation continues**. If no eligible active capacity uses the reserved CUs, the discount benefit for that hour is lost; unused reserved hours do not roll over. [S2] [S3]
- **F4 model:** using the prior comparison's *illustrative* US reference figures of **$0.72 per active F4 hour PAYG** and **$312.67/month for a 4-CU one-year reservation monthly equivalent**, the compute-only break-even is approximately **434.3 active hours/month**. The reservation is economically preferable above that threshold, subject to actual purchase price and eligible utilization.
- **Recommendation:** use PAYG with automated pause/resume for isolated development, testing, and daytime ETL capacities; use reservation coverage for consistently running, business-critical capacities (especially always-accessible Power BI reports), unless another eligible capacity can absorb unused reservation coverage. [S1] [S2] [S3]

> **Important distinction:** You **pause the Fabric capacity**, not an individual workspace. If several workspaces share the same F4, pausing F4 can make content and jobs in all those workspaces unavailable. [S1] [S8]

## 1. What does “Reserved Fabric” actually mean?

Microsoft Fabric capacity is provisioned as an **Azure F-SKU resource** (such as F4 with **4 Capacity Units**). It runs Fabric workloads including notebooks, pipelines, warehouses, and Power BI. You have two principal ways to pay for this compute:

1. **PAYG:** pay for the F-SKU while it is active; Azure bills per second with a one-minute minimum. [S6]
2. **Reservation discount:** buy a commitment for a specific quantity of Fabric **CUs** in an eligible Azure region/scope, currently offered in one- and three-year commitments. That discount is matched to eligible running capacity usage. Reservations are not separate clusters, nor do they permanently attach to a particular workspace. [S2] [S3]

The underlying F4 resource remains pauseable in either case. Pausing the resource stops its ordinary capacity-usage emission, but **does not cancel or prorate the reservation contract**. If other eligible running F-SKU capacity usage exists in the reservation's matched scope and region, the unused reservation can apply there instead. [S2] [S3]

| Question | F4 PAYG | F4 covered by reservation |
|---|---|---|
| Can the **capacity** be paused/resumed? | **Yes** | **Yes** |
| Is the **workspace** independently powered off? | No; capacity-level action | No; capacity-level action |
| Does ordinary provisioned compute usage stop while paused? | **Yes** | **Yes** |
| Does the payment commitment stop when paused? | **Yes, for inactive PAYG compute time** | **No, reservation commitment continues** |
| Can unused reservation hours be saved for another day? | Not applicable | **No** |
| Can the reserved discount help another matching running capacity? | Not applicable | **Yes, under reservation scope/region rules** |
| Additional charges possible? | Storage, other services, pause-time settlement and enabled overage | Same, plus ongoing reservation commitment |
| Most economical when... | Capacity is only needed for part of the month | Capacity is used consistently for most hours |

**Evidence:** Microsoft's [Fabric pause/resume documentation][S1], [reservation rules][S2], and [hourly discount application examples][S3].

## 2. F4 monthly costs: transparent planning assumptions

The figures below intentionally preserve the reference prices from our earlier F4 analysis so the decision is comparable. **Azure's public pricing webpage renders region- and agreement-specific dollar values dynamically; the $0.72 and $312.67 rates should be checked in Azure Pricing Calculator / your contracted pricing before procurement.** The authoritative service [pricing page][S7] gives the current billing structure and available reservation terms.

| Model input | Assumed value | Meaning |
|---|---:|---|
| SKU | F4 | 4 CUs provisioned |
| PAYG hourly compute reference | **$0.72 / active hour** | Applies only while the capacity runs |
| Reservation reference | **$312.67 / month** | Equivalent monthly cost of reserved 4 CUs; still owed when paused |
| Full month | **730 hours** | Convenient average-month denominator |
| PAYG full-month cost | **$525.60** | $0.72 × 730 |
| One-year reservation annualized | **$3,752.04** | $312.67 × 12; rounding of monthly equivalent may differ from actual invoice |
| Out-of-scope cost categories | Not included | OneLake storage, Power BI user licenses, networking, optional Spark autoscale, overage and pause-time settlement |

**Model equations** (assuming no eligible concurrent capacity that can utilize an otherwise idle reservation):

- `PAYG_monthly_compute = $0.72 × active_capacity_hours`
- `Reserved_monthly_commitment_equivalent = $312.67`
- `Break_even_hours = $312.67 / $0.72 = 434.26 hours/month`
- `Break_even_uptime = 434.26 / 730 = 59.49%`

**Interpretation:** below approximately **434 active hours/month**, PAYG has a lower *modeled compute cost*. Above that level, the reserved commitment is lower. Do not generalize this threshold across regions or different reservations without recalculating.

## 3. F4 usage scenarios: monthly compute-only cost

| Operation pattern | Active hours/month | F4 PAYG | Reserved monthly equivalent | Lower modeled cost |
|---|---:|---:|---:|---|
| Paused entire month | 0 | $0.00 | $312.67 | PAYG |
| 4 hours × 22 workdays | 88 | $63.36 | $312.67 | PAYG |
| **8 hours × 22 workdays** | **176** | **$126.72** | **$312.67** | **PAYG** |
| 12 hours × 22 workdays | 264 | $190.08 | $312.67 | PAYG |
| 16 hours × 22 workdays | 352 | $253.44 | $312.67 | PAYG |
| 20 hours × 22 workdays | 440 | $316.80 | $312.67 | Reserved (slightly) |
| 24/7 reference month | 730 | $525.60 | $312.67 | Reserved |

### Worked example — development/ETL only during business hours

If a dedicated F4 capacity runs **8 hours each of 22 working days** and is paused at other times:

- **Always-on PAYG:** $0.72 × 730 = **$525.60/month**.
- **Scheduled PAYG:** $0.72 × 176 = **$126.72/month**.
- **Potential reduction vs always-on PAYG:** **$398.88/month**, or **75.9%** of the baseline compute meter.
- **Reserved commitment:** approximately **$312.67/month** regardless of whether *this particular* F4 is paused, unless the reservation discount is used by other eligible capacities.
- **PAYG vs reserved at 176 hours:** **$185.95/month** lower modeled cost for PAYG.

All figures exclude storage and other costs; they are not predictions of an actual Azure bill. In real schedules, allow extra warm-up, job completion, and recovery time. For instance, **9 hours × 22 days = 198 hours** would cost **$142.56** in modeled F4 PAYG compute.

### Worked example — 24/7 production reporting

For an F4 that must be continuously active, the modeled monthly compute costs are **$525.60 PAYG** versus **$312.67 reservation**, for a **$212.93/month** apparent reservation advantage. This does not include the risk of underutilization over the reservation term, and does not prove that F4 itself meets the application performance SLA.

## 4. What does pausing actually stop — and what keeps costing money?

| Item | Paused-capacity behavior | Budget implication |
|---|---|---|
| PAYG F4 provisioned compute | Meter stops while paused [S1] [S6] | Primary savings source |
| Existing Fabric reservation contract | Financial obligation remains [S2] [S3] | Cannot recover unused reserved hours |
| OneLake stored bytes | Storage remains billed, even when paused [S4] | No pause-based storage saving |
| OneLake transaction requests against paused capacity | Rejected while paused [S4] | No ordinary OneLake transaction CU usage on that paused capacity |
| Power BI reports and Fabric workloads assigned to paused capacity | Can become unavailable [S1] | May break user access and SLAs |
| In-flight Warehouse requests | Can be canceled; cache is cold after resume [S5] | Protect jobs from abrupt pause |
| Previously smoothed/background usage | May be reconciled and billed on pause [S1] [S9] | A one-time charge can occur |
| Optional capacity overage | Separate excess-CU charges when enabled [S10] | Budget alerts are still required |
| Optional Spark autoscale billing | Separately metered, not covered by reservation [S7] | Analyze outside base F4 model |

**Nuance:** Pausing a busy capacity can reconcile accumulated usage sooner rather than make that historical consumption free. The **Capacity Metrics app** shows pause events, carryforward and reconciliation spikes. [S9]

## 5. When should you choose PAYG versus reserved?

### Use PAYG with scheduled pause/resume if:

- The capacity is a **dedicated dev/test workspace pool** or scheduled ETL environment, not user-facing 24/7 BI.
- Work is concentrated into predictable daily or weekly windows.
- You can tolerate job waits and service unavailability while paused.
- Expected active hours are **under the recalculated break-even** after accounting for real bill rates and costs.
- No existing reserved CUs would otherwise go unused and could economically cover the capacity.

### Consider reservation coverage if:

- The capacity operates most of the month or business-critical reporting requires continuous availability.
- The organization has stable baseline CU needs and adequate long-term forecast confidence.
- Matching workloads can share reservation coverage across qualifying Azure resources and scopes.
- Reservation discount utilization is high enough to justify the commitment, including during off-hours.

**A hybrid approach can be superior:** retain reserved CU coverage for a reliable production baseline; keep a separate PAYG F2/F4 for experimental, intermittent notebooks or ETL that can pause when idle. Microsoft's capacity-planning guidance discusses combining reservations with PAYG flexibility. [S11]

## 6. How to pause, resume, and automate F4 safely

### Manual Azure portal

1. Sign into [Azure Portal](https://portal.azure.com/).
2. Find **Microsoft Fabric**, then select the **F4 capacity resource** (not a workspace).
3. Select **Pause**, confirm, and verify its state.
4. When work is needed, select **Resume** and wait for service availability. [S1]

### Scheduled automation

Microsoft documents **Azure Automation runbooks** and the **Fabric capacity Suspend/Resume ARM REST APIs** as supported control mechanisms. Implement independent, authenticated resume and pause schedules with least-privilege permissions, using the documented actions `Microsoft.Fabric/capacities/suspend/action` and `.../resume/action`. [S1] [S12] [S13]

**Recommended operational pattern for daytime ETL:**

1. **Resume early** to allow startup and run readiness tests.
2. Start pipelines/notebooks after the capacity is confirmed active.
3. Validate ETL status, downstream refresh, and outstanding background work.
4. **Pause only after** scheduled processing finishes, no business-critical users are connected, and a stop policy approves the action.
5. Log state transitions and reconcile the **Azure Cost Management** bill and **Fabric Capacity Metrics** against expected active hours. [S1] [S8] [S9]

For automation, treat pausing during an active job as a **failure scenario**, not as a normal way to end work. Include alerts for failed resume, jobs running past the pause window, unexpected capacity overage, and dashboards unavailable after resumption.

## 7. Risks and controls

| Risk | Why it matters | Recommended control |
|---|---|---|
| Reservation purchased for a capacity that sits idle | Discount hours may be forfeited | Check hourly CU-reservation utilization before purchase; match other workloads where permitted [S3] |
| Pausing a shared F4 | Every assigned workspace may be affected | Dedicated capacity for interruptible workloads; inventory assignments first [S1] [S8] |
| Background CU carryforward at pause | Cost spike can appear upon reconciliation | Review the Capacity Metrics app; do not treat pause as free debt cancellation [S9] |
| Capacity overage billing | Eligible excess usage may be billed separately at higher rates | Verify overage enablement and policy, monitor CU thresholds [S10] |
| Job interruption / failed reporting | Pipelines and queries may be interrupted, dashboards unavailable | Resume early, wait for completion, test dashboards, have incident override [S1] [S5] |
| Cold-cache performance after resume | First warehouse queries can be slower | Account for warm-up and run representative checks [S5] |
| Unexpected storage/network/user licensing bills | Pausing does not stop all service charges | Track compute, OneLake, licensing, and networking separately [S2] [S4] |
| Prices differ from assumptions | Regional list prices, discounts and contracts vary | Recalculate using actual Azure Cost Management and Pricing Calculator [S7] |

## 8. Recommendation for the existing F4 baseline

**If F4 primarily supports internal experimentation or scheduled ETL:** start with **PAYG + managed automatic pause/resume**, not a full 4-CU reservation. A daytime-only pattern looks much cheaper under the current reference prices, and the architecture can be validated without a multiyear financial commitment. Before enabling scheduled shutdown, verify that the F4 does not also support continuously accessed Power BI reports or off-hour operations.

**If F4 supports daily operational dashboards, interactive queries or 24/7 downstream workloads:** keep it active and evaluate **reservation coverage** against actual observed utilization. Pausing the capacity could interrupt access; selecting the cheaper billing commitment does not solve compute-sizing or availability concerns.

**If a reservation is already purchased:** pausing the capacity is still supported, but is not by itself a way to reduce the sunk reservation commitment. Instead, examine whether other eligible capacities can utilize the reserved CUs during the same hours, or evaluate a refund/change only under Microsoft's applicable reservation policies. [S2] [S3]

### Decision checklist

- [ ] Identify all workspaces, reports, pipelines and notebooks assigned to the F4.
- [ ] Measure 2–4 weeks of active hours, CU utilization, throttling, and user-access windows.
- [ ] Verify the actual F4 PAYG and reservation price for the Azure subscription, region and contract.
- [ ] Confirm whether any Fabric reservation is already in place and its scope/available CUs.
- [ ] Pilot scheduled pause/resume on a nonproduction capacity.
- [ ] Reconcile estimated savings against actual bills, including storage and one-time reconciliation charges.
- [ ] Move to production automation only after meeting data freshness and availability SLAs.

---

## Sources and evidence register

All links below are primary Microsoft documentation. Price arithmetic is independently derived from the explicitly labeled **planning rates**, rather than quoted as a live Azure price. Sources verified November 2, 2026.

1. [S1 — Pause and resume your Fabric capacity](https://learn.microsoft.com/en-us/fabric/enterprise/pause-resume) — manual/automated controls, permission requirements, and billing on pause.
2. [S2 — Save costs with Fabric Capacity reservations](https://learn.microsoft.com/en-us/azure/cost-management-billing/reservations/fabric-capacity) — financial commitment, reserved CU matching, and hourly discount utilization.
3. [S3 — Determine the Azure reservation to purchase](https://learn.microsoft.com/en-us/azure/cost-management-billing/reservations/determine-reservation-purchase) — unused hourly reservations do not carry forward.
4. [S4 — OneLake compute and storage consumption](https://learn.microsoft.com/en-us/fabric/onelake/onelake-consumption) — storage and transactions when capacity is paused.
5. [S5 — Pause and resume in Fabric Data Warehouse](https://learn.microsoft.com/en-us/fabric/data-warehouse/pause-resume) — in-flight requests, cache behavior, and storage billing.
6. [S6 — Fabric licenses and capacity](https://learn.microsoft.com/en-us/fabric/enterprise/licenses) — billing granularity and licensing context.
7. [S7 — Microsoft Fabric pricing](https://azure.microsoft.com/en-us/pricing/details/microsoft-fabric/) — official region-sensitive price calculator and reservation options.
8. [S8 — Monitor capacity health](https://learn.microsoft.com/en-us/fabric/admin/monitoring-hub-capacity) — assigned workspaces, utilization and capacity operations.
9. [S9 — Monitor a paused capacity](https://learn.microsoft.com/en-us/fabric/enterprise/monitor-paused-capacity) — pause events and usage-reconciliation spikes.
10. [S10 — Fabric capacity overage](https://learn.microsoft.com/en-us/fabric/enterprise/capacity-overage-overview) — separate excess consumption billing.
11. [S11 — Fabric capacity planning](https://learn.microsoft.com/en-us/fabric/enterprise/capacity-planning-manage-capacity-growth-governance) — reserved/PAYG combination guidance.
12. [S12 — Suspend Fabric capacity API](https://learn.microsoft.com/en-us/rest/api/microsoftfabric/fabric-capacities/suspend) — programmatic pause.
13. [S13 — Resume Fabric capacity API](https://learn.microsoft.com/en-us/rest/api/microsoftfabric/fabric-capacities/resume) — programmatic resume.

[S1]: https://learn.microsoft.com/en-us/fabric/enterprise/pause-resume "Pause and resume your Fabric capacity — Microsoft Learn"
[S2]: https://learn.microsoft.com/en-us/azure/cost-management-billing/reservations/fabric-capacity "Save costs with Microsoft Fabric Capacity reservations — Microsoft Learn"
[S3]: https://learn.microsoft.com/en-us/azure/cost-management-billing/reservations/determine-reservation-purchase "Determine what Azure reservation you should purchase — Microsoft Learn"
[S4]: https://learn.microsoft.com/en-us/fabric/onelake/onelake-consumption "OneLake compute and storage consumption — Microsoft Learn"
[S5]: https://learn.microsoft.com/en-us/fabric/data-warehouse/pause-resume "Pause and Resume in Fabric Data Warehouse — Microsoft Learn"
[S6]: https://learn.microsoft.com/en-us/fabric/enterprise/licenses "Understand Microsoft Fabric licenses and capacity — Microsoft Learn"
[S7]: https://azure.microsoft.com/en-us/pricing/details/microsoft-fabric/ "Microsoft Fabric pricing — Microsoft Azure"
[S8]: https://learn.microsoft.com/en-us/fabric/admin/monitoring-hub-capacity "Monitor capacity health — Microsoft Learn"
[S9]: https://learn.microsoft.com/en-us/fabric/enterprise/monitor-paused-capacity "Monitor a paused capacity — Microsoft Learn"
[S10]: https://learn.microsoft.com/en-us/fabric/enterprise/capacity-overage-overview "Capacity overage in Microsoft Fabric — Microsoft Learn"
[S11]: https://learn.microsoft.com/en-us/fabric/enterprise/capacity-planning-manage-capacity-growth-governance "Microsoft Fabric capacity planning — Microsoft Learn"
[S12]: https://learn.microsoft.com/en-us/rest/api/microsoftfabric/fabric-capacities/suspend "Fabric capacities Suspend REST API — Microsoft Learn"
[S13]: https://learn.microsoft.com/en-us/rest/api/microsoftfabric/fabric-capacities/resume "Fabric capacities Resume REST API — Microsoft Learn"

### Calculation verification

All numeric amounts are simple multiplication or division of the assumptions stated in Section 2. For procurement, use region-specific rates from [Azure Fabric pricing][S7] or the Azure Pricing Calculator, not the example figures as a binding quote. The reservation amount is an **amortized monthly equivalent**, not a representation of the invoice payment schedule.
