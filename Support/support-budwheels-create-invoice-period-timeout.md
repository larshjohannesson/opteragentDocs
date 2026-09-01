# BudWheels (CustomerID 208) — "Create invoice period" hangs and times out

> ## 📄 Looking for the fix? It is **not** in this file.
> **→ `budwheels-pit-cleanup-runbook.md`** — short, self-contained, current: the six criteria, the
> five scripts, measured row counts, and the consumer-safety analysis.
>
> This document is a **working log** of the investigation. It is long and contains claims that were
> later superseded (they are marked, but they are still in here). Read it for *how* the conclusion was
> reached, or for the secondary findings in §7 items 3–6. Do not read it as a runbook.

**Version:** `20251200.000256` (releases/2025.12.256 — the log says .256, the case says .250)
**Reported by:** support experiments 2026-08-11 12:08–12:22 UTC, user `Cecilia`, machine `avd-rdm-1`
**Symptom:** creating an invoice period with **[Alla]** selected hangs the client and eventually times out. Selecting individual orders works.

---

## ✅ RESOLVED 2026-08-13 — reproduced and root-caused on a local restore

Reproduced on a local copy of `budwheelsse` (SQL 2022 Developer, compat level **110**) by running the real SP inside a rolled-back transaction:

```
EXEC k2_inv2_create_invoice  @CUS_Id=5539, @CPR_Ids=NULL, @PST_Ids=NULL, @DEL_Ids=NULL   -- [Alla]
→ TOTAL 339 491 ms      (client command timeout is 300 000 ms)
```

Per-procedure cost of that single run:

| procedure | total ms | CPU ms | logical reads |
|---|---|---|---|
| `k2_inv2_create_invoice` | 339 416 | 2 957 451 | 27 806 998 |
| `k2_inv2_create_rows` | 337 065 | 2 955 165 | 27 744 080 |
| **`k2_inv2_create_rows_price_item`** | **334 783** | **2 952 936** | **27 686 497** |
| `k2_inv2_complete_invoice` | 450 | 442 | 21 878 |
| `k2_inv2_create_header` | 201 | 186 | 3 977 |
| all others | < 10 ms each | | |

**98.6 % of the runtime is a single statement** — the `SELECT DISTINCT … INTO #inr_tmp` in `k2_inv2_create_rows_price_item` (§4b).

### The causal chain, end to end

1. Auto-created `AddService` price items that never receive a price are stamped `PIT_DIS_Id = 2` by [`CustomerPrice.cs:822`](Order/Price/CustomerPrice.cs:822) and never retired — leaking ~2 000/month since at least 2013 (§4b).
2. **200 313** such rows have accumulated: **92.5 %** of the 216 563-row pending-price-item table.
3. `k2_inv2_create_rows_price_item` drives from that whole office-wide set with a forced `INNER LOOP JOIN`. Cost is proportional to **how many of those pending rows belong to the customer being invoiced** (see the correction below — this is *not* office-wide-independent).
4. CUS 5539 owns **14 369** pending price items, **14 354 of them stale (99.9 %)**. At that size the statement takes ~335 s, blowing the 300 s timeout in [`InvoiceCreator.cs:192`](Server/Economy/Invoices/InvoiceCreator.cs:192).
5. The timeout is swallowed by the bare `catch` at [`InvoiceCreator.cs:227`](Server/Economy/Invoices/InvoiceCreator.cs:227), so the client shows a generic "could not create invoice".
6. The user then tries to delete the half-built period; that has only a 120 s timeout and blocks behind the still-running create, leaving an undeletable orphan period (§2).

### ✅ Verified end to end through the UI, 2026-08-13

The five-step cleanup sequence (§7 item 1) was run against the local restore, then invoice-period creation was exercised **through the Opter client UI** for **`CUS 9174 — Westers Catering & Arrangemang AB`**:

* **Before the cleanup:** could not create the period/invoice.
* **After the cleanup:** creates successfully.

That closes the loop. The fix is confirmed not just by SQL timings but by the real client path that the customer actually uses.

**It also confirms the blind prediction.** CUS 9174 was never mentioned anywhere in the support case — it was identified purely by ranking customers on pending price items, where it sits top with 21 835 rows. It was then found to be genuinely broken. Three independent confirmations of the predictive model now:

| CUS_Id | pending PIT | how identified | confirmed |
|---|---|---|---|
| 9174 | 21 835 | **predicted from backlog ranking alone** | ✅ broken before fix, works after |
| 5539 | 14 369 | reported by support | ✅ reproduced at ~339 s |
| 14508 | 6 134 | seen in a support screenshot | ✅ reproduced at ~101 s |

So *"pending price items per customer"* is a reliable predictor of which customers are affected, and can be used to triage the rest without waiting for complaints.

### The cleanup is confirmed to be the fix

Retiring the stale rows inside the same rolled-back transaction and re-running:

```
stale rows retired: 200 313      driving set 216 563 → 16 250
EXEC k2_inv2_create_invoice  →  22 154 ms      (was 339 491 ms)
```

**15.3× faster, and comfortably inside the timeout.** Cost is roughly linear in the driving-set size (15.3× time for a 13.3× row reduction). This retracts the earlier caveat under fix 1 that the cleanup could not be promised to fix the symptom — **it does**.

Extrapolating, the 300 s threshold sits near **~191 000** driving rows, so at the observed ~2 000/month leak rate the cleanup buys roughly **7 years** of headroom. It is still a mitigation, not a fix: item 2 makes the backlog size irrelevant, item 3 stops the regrowth.

### What this rules out

* **Not blocking.** `blocking_session_id = 0`; the only wait was `CXCONSUMER` (intra-query parallelism), on a local copy with no competing sessions. The `SYS_System` lock (§4c) is a real defect but **not this bug**.
* **Not stale statistics.** All relevant statistics were FULLSCAN-fresh with `modification_counter = 0`; `IX_PIT_DIS_Id_Filt` correctly reported 216 563 rows. The optimizer had accurate numbers and still produced this plan — consistent with the forced `INNER LOOP JOIN` removing its ability to do anything else.
* **Not parameter sniffing.** `sp_recompile` had already been tried against production with no effect. See `support-budwheels-execution-plan-hypothesis.md`.
* **Not the construction-invoice path.** `CUS_ConstructionCompany = 0` for CUS 5539 (only 11 of 15 486 customers have it set).

### ⚠️ Correction — support was right, it really is only *some* customers

An earlier version of this section claimed the 335 s was a customer-independent office-wide scan and therefore `[Alla]` must be failing for the whole customer base. **That was wrong.** Two measurements settle it:

```
CUS 5539  (14 369 pending price items), [Alla]  → 339 491 ms
CUS 18094 (     0 pending price items), [Alla]  →   1 015 ms
CUS 18094                             , orders  →      99 ms
```

Both are Project-mode, both `[Alla]`, both with zero orders inside the period's date range, and the **office-wide backlog was the full 216 563 rows in both runs**. So the office-wide driving scan costs about **one second** — it is real waste, but it is not the timeout. The 335 s is driven by **how many pending price items the invoiced customer owns**.

**Pending price items per customer** — this is the predictor of who fails:

| CUS_Id | name | pending PIT | of which stale |
|---|---|---|---|
| 9174 | Westers Catering & Arrangemang AB | **21 835** | 21 789 |
| **5539** | **Ljud & Bild Media AB** | **14 369** | **14 354** |
| 14508 | STHLM Lights AB | 6 134 | 6 114 |
| 2382 | Bw När Chaufförer Delar Transp | 5 829 | 2 441 |
| 2798 | Storyline Studios STO AB | 4 998 | 4 995 |

Distribution across all customers:

| pending PIT rows | customers | total rows |
|---|---|---|
| 10 000+ | **2** | 36 204 |
| 1 000–9 999 | **27** | 57 680 |
| 100–999 | 272 | 70 735 |
| 10–99 | 1 302 | 40 761 |
| 1–9 | 4 365 | 11 183 |

So roughly **29 customers** carry a backlog in the range that produces a timeout, out of ~15 500. That is exactly the "some of their customers" support reported, and it is predictable rather than mysterious.

**Independently corroborated:** `CUS 14508 — STHLM Lights AB`, third on this list with 6 134 pending price items, appears in an early screenshot in the support case as one of the customers they were having trouble with. That is a customer identified by support from their own observations, matched by a list derived purely from backlog size, with no knowledge of the case. Two of the top three predicted customers (5539 and 14508) are confirmed problem customers.

**CUS 9174 — Westers Catering & Arrangemang AB** carries *more* backlog than 5539 (21 835 rows) and should therefore be the worst affected. It has not been mentioned in the case; worth asking support whether that customer has complained, as it is the strongest remaining prediction.

### Who times out and who is merely slow — model fitted to three measurements

Measured, all `[Alla]`, all on the same period, office-wide backlog full in every run:

| CUS_Id | pending PIT | measured |
|---|---|---|
| 18094 | 0 | **1 015 ms** |
| 14508 | 6 134 | **101 471 ms** |
| 5539 | 14 369 | **339 491 ms** |

The relationship is **superlinear**: 16.4 ms per row at 6 134 rows, rising to 23.6 ms per row at 14 369. A 2.34× increase in rows produces a 3.35× increase in time, i.e. cost ∝ rows^1.42 — worse than linear, better than quadratic, consistent with a sort or hash aggregate over the surviving row set. Fitting:

```
elapsed_ms  ≈  0.42 × (pending_PIT_rows) ^ 1.42
```

which reproduces both measured points within 1 % (6 134 → 101 s, 14 369 → 339 s). That puts the **300 s timeout at about 13 100 pending price items.**

| CUS_Id | name | pending PIT | expected | symptom |
|---|---|---|---|---|
| 9174 | Westers Catering & Arrangemang AB | 21 835 | **~615 s** | times out badly |
| 5539 | Ljud & Bild Media AB | 14 369 | 339 s *(measured)* | times out |
| — | *timeout threshold* | *~13 100* | *300 s* | |
| 14508 | STHLM Lights AB | 6 134 | 101 s *(measured)* | very slow, completes |
| 2382 | Bw När Chaufförer Delar Transp | 5 829 | ~95 s | very slow, completes |
| 2798 | Storyline Studios STO AB | 4 998 | ~76 s | very slow, completes |

**Only the two customers above ~13 100 rows actually hit the timeout.** The other ~27 in the 1 000–9 999 bucket are slow enough to look broken but do complete — which is exactly why `CUS 14508` was reported as a *problem* customer rather than a *timeout*, and it is a good match for a support picture that mixes hard failures with hangs that eventually resolve.

An earlier linear fit over-predicted 14508 at ~145 s against the measured 101 s. The superlinear fit above replaces it. Still an extrapolation from three points, but the two interior points are measured and the shape is consistent, so it is fit for triage: **rank the remaining customers by pending price items and work down from the top.**

This also explains why support could not reproduce it on a newly created customer: a new customer has **zero** accumulated backlog, so it takes the 1-second path regardless of `[Alla]`. Nothing to do with the Order-mode exemption (§3b) — the two exempt customers, `13335` and `14175`, are long-standing and unrelated.

**The cleanup fix is unaffected by this correction, and now better explained:** 99.9 % of the affected customers' backlog is stale, so retiring it removes almost exactly the rows that make their invoices slow.

For reference, the invoicing-mode split (relevant to §3b's fast-path exemption, not to the timeout):

| `COALESCE(CUS_INT_Id, OFF_INT_Id)` | mode | customers |
|---|---|---|
| 2 | Project | 15 481 |
| 1 | Collection | 3 |
| 3 | Order — always sends `DEL_Ids` | 2 |

---

### Code-version verification

The analysis below was originally read from the `master` working tree. Every file it relies on has since been diffed against the customer's exact build, commit `b2ee9c7831` — *"Version: 2025.12.256 (20251200.000256)"*. Result:

| | vs. `master` |
|---|---|
| All 15 stored procedures in the `k2_inv2_create_invoice` chain | **byte-identical** |
| `CreateInvoicesViewModel.cs` (the `[Alla]` logic, §3 / §3b) | **byte-identical** |
| `InvoiceCreator.cs` (300 s timeout, transaction, bare `catch`) | **byte-identical** |
| `EconomyInvoicesInvoiceCreatorController.cs` | **byte-identical** |
| `IX_PIT_DIS_Id_Filt`, `IX_DEL_ToBeInvoiced`, `IX_INH_INP_Id_incl` | **identical definitions** |
| `DIS_DeliveryInvoiceStatus`, `PIY_PriceItemType`, `INT_InvoiceType` enums | **identical values** |
| `Invoice.cs` — `DeleteInvoicePeriod` 120 s timeout | **identical** (file differs elsewhere; diff touches no timeout) |
| `DatabaseAccess.cs` — `ExecuteStatement` retry logic | **identical** (file differs elsewhere; diff touches no retry code) |

The `PIT_PriceItem` / `DEL_Delivery` / `INH_InvoiceHeader` table scripts and `Model/Tables.cs` differ between the two branches, but **not** in any index or enum cited here. Every line number, quoted snippet and conclusion in this document is therefore valid for what BudWheels is actually running.

One consequence worth stating: because the SPs are identical, **every fix in §7 applies unchanged to `master` as well** — this is not a release-branch-only defect, and `master` carries it too.

---

## 1. Conclusion first

`k2_inv_create_invoice_period` is **not** the problem. That SP is four `SELECT`s and one `INSERT` — it cannot hang.

The period *is* created successfully. What hangs is the **next** step: `CreateInvoicesViewModel` loops over every invoice candidate and calls `EconomyInvoicesInvoiceCreator/CreateInvoice` → **`k2_inv2_create_invoice`**, once per customer. For BudWheels' customer **CUS_Id 5539** that single call exceeds its **300 second** command timeout.

Because the client shows one progress dialog spanning "create period → create invoices", the user perceives it as "creating the invoice period hangs".

**Measured 2026-08-11:** CUS 5539 has only **9** pending deliveries, and the delivery × price-item fan-out is also **9** (no fan-out at all). So the 300 s is *not* proportional to this customer's data — a nine-delivery invoice is taking over five minutes.

That rules out the row-volume explanation entirely and leaves exactly two candidates, both of which are cost centres that **do not scale with the customer**:

* **§4b — office-wide backlog scan.** `k2_inv2_create_rows_price_item` / `_add_service` / `_direct_expense` / `_expense` drive from the *pending-items* table filtered by **office and status only, never by customer**. This is the prime suspect: it explains a 300 s call for a 9-delivery customer, and it is exactly the part that `[Alla]` disarms the optimizer against (§3).
* **§4c — lock wait on `SYS_System`.** If the 300 s was spent waiting rather than working, this is the mechanism.

Distinguishing these two needs one measurement — see §6. The `[Alla]` correlation itself is exact and comes from one line of client code (§3).

---

## 2. Evidence from the log dump

67 log rows: 61 client UI-watchdog warnings, 5 ServerApi errors, 1 empty Web.App error.

The 5 ServerApi errors are all `Execution Timeout Expired` (`Error Number:-2`), from two SPs:

| Time (UTC) | SP | Params | Timeout |
|---|---|---|---|
| 12:12:52.100 | `k2_inv_delete_invoice_period` | `@INP_Id=21472` | 120 s |
| 12:13:55.504 | `k2_inv2_create_invoice` | `@CUS_Id=5539, @INP_Id=21472` | 300 s |
| 12:19:31.462 | `k2_inv_delete_invoice_period` | `@INP_Id=21474` | 120 s |
| 12:21:01.547 | `k2_inv_delete_invoice_period` | `@INP_Id=21474` | 120 s |
| 12:21:33.576 | `k2_inv2_create_invoice` | `@CUS_Id=5539, @INP_Id=21474` | 300 s |

Subtracting the configured timeouts ([`Invoice.cs:435`](Server/Economy/Invoices/Invoice.cs:435) = 120 s, [`InvoiceCreator.cs:192`](Server/Economy/Invoices/InvoiceCreator.cs:192) = 300 s) reconstructs the timeline exactly:

```
12:08:55  create invoice for CUS 5539 in period 21472 starts
12:10:52  user gives up, clicks "delete invoice period"  → blocks behind the running create
12:10:55  UI-thread watchdog episode 1 begins (currentCloudCall = /DeleteInvoicePeriod/)
12:12:52  delete times out (120 s)          ← blocked, not slow
12:12:48  watchdog episode 1 ends
12:13:55  create times out (300 s)          ← the real problem
12:14:52  Web.App error (empty message)

12:16:33  second attempt, period 21474
12:17:31  delete attempt #1 → times out 12:19:31   (watchdog episode 2)
12:19:01  delete attempt #2 → times out 12:21:01   (watchdog episode 4,
                                                    InvoiceWindowViewModel.InvoicePeriodDeleteCommand)
12:21:33  create times out (300 s)
```

Two independent facts worth noting:

* Every watchdog warning has `currentCloudCall = /EconomyInvoicesInvoice/DeleteInvoicePeriod/` and `executingCommand` empty or `InvoicePeriodDeleteCommand`. **The client-side hang the watchdog caught is the *delete*, not the create** — the create runs on a worker thread behind a progress dialog, so the UI thread stays alive. `heapMb` stays flat at 153–155 MB and `gcGen2` never moves, so this is pure I/O wait, not a client-side leak or GC storm.
* Both runs failed on the **same customer, 5539**, and 5539 is the *only* customer appearing in the log. [`CreateInvoicesViewModel.cs:1339`](ViewModels/Economy/Invoice/CreateInvoices/CreateInvoicesViewModel.cs:1339) aborts the loop on the first failure, so 5539 is at minimum the first customer that fails. Combined with the measured 9 pending deliveries, the most likely reading of the case is that support **selected customer 5539** and left the **order** list at `[Alla]` — i.e. `[Alla]` refers to the order/delivery filter, not the customer filter. That matches "trouble for some of their customers" and "no problem when individual orders are selected" exactly, and means we are looking at **one** invoice for a nine-delivery customer taking over 300 s.

---

## 3. Why only `[Alla]` — the exact line

[`CreateInvoicesViewModel.cs:802-808`](ViewModels/Economy/Invoice/CreateInvoices/CreateInvoicesViewModel.cs:802):

```csharp
if (!_deliveries.AllChecked || ic.Customer.CUS_INT_Id == (int)Tables.INT_InvoiceType.Order)
{
    if (!invoiceCandidate.DEL_Ids.Contains(ic.DEL_Id))
    {
        invoiceCandidate.DEL_Ids.Add(ic.DEL_Id);
    }
}
```

`DEL_Ids` is populated **only when the order filter is not "all checked"**. Same shape for `CPR_Ids` (line 784) and `PST_Ids` (line 794).

With `[Alla]`, all three lists stay empty → `NullIfZeroOrEmpty(CreateString(...))` turns them into `NULL` → matching the log exactly:

```
@CPR_Ids=NULL, @PST_Ids=NULL, @DEL_Ids=NULL
```

Downstream, [`k2_inv2_create_invoice_data_selection`](Database/RedGateScripts/Stored%20Procedures/dbo.k2_inv2_create_invoice_data_selection.sql:84) then takes the `@c = 0` branch and writes **one `IDS_InvoiceDataSelection` row with all three filter columns NULL** — i.e. "no restriction".

Every row-creation SP filters on that table like this ([`k2_inv2_create_rows_delivery.sql:79`](Database/RedGateScripts/Stored%20Procedures/dbo.k2_inv2_create_rows_delivery.sql:79)):

```sql
AND EXISTS (SELECT IDS_Id FROM IDS_InvoiceDataSelection IDS
            WHERE IDS.OFF_Id = DEL.OFF_Id
            AND   IDS_INH_Id = @INH_Id
            AND   COALESCE(IDS_CPR_Id, DEL_CPR_Id, 0) = ISNULL(DEL_CPR_Id, 0)
            AND   ISNULL(IDS_PST_Id, DEL_PST_Id)      = DEL_PST_Id
            AND   ISNULL(IDS_DEL_Id, DEL_Id)          = DEL_Id)
```

With individual orders selected, `IDS_DEL_Id` holds real ids and this becomes a selective semi-join the optimizer can drive the query from. With `[Alla]`, all three `ISNULL(…)` collapse to tautologies and the predicate is worthless — the full unrestricted candidate set for the customer must be built.

**That is the whole `[Alla]` mechanism.** "Some of their customers" = the customers whose unrestricted candidate set is large enough to blow past 300 s.

---

## 3b. Why some `CUS_Id` and not others

The §4b scan cost is the same for every customer, so something else decides who tips over 300 s. Three factors, in decreasing order of certainty.

### Certain — `CUS_INT_Id = 3` (Order) customers are structurally exempt

Look again at the `[Alla]` guard, [`CreateInvoicesViewModel.cs:802`](ViewModels/Economy/Invoice/CreateInvoices/CreateInvoicesViewModel.cs:802):

```csharp
if (!_deliveries.AllChecked || ic.Customer.CUS_INT_Id == (int)Tables.INT_InvoiceType.Order)
```

That `||` means an **Order-invoiced customer always sends `DEL_Ids`, even under `[Alla]`**. They get real ids in `IDS`, the optimizer gets its selective driver, and they take the fast path unconditionally. These customers can never exhibit the bug.

The project equivalent, [line 784](ViewModels/Economy/Invoice/CreateInvoices/CreateInvoicesViewModel.cs:784), has the same exemption **but with an extra condition**:

```csharp
if ((!_customerProjects.AllChecked || ic.Customer.CUS_INT_Id == (int)Tables.INT_InvoiceType.Project)
    && ic.CPR_Id != 0)
```

`CPR_Ids` is only populated when the delivery actually **has** a project. And a `CPR_Id` filter is weaker anyway — it makes `IDS_CPR_Id = ISNULL(DEL_CPR_Id,0)`, which narrows rows but gives no `DEL_Id` equality to drive from. Only a non-null `IDS_DEL_Id` produces `IDS_DEL_Id = DEL_Id`, the joinable predicate that lets the optimizer start from the small set.

So under `[Alla]`:

| `CUS_INT_Id` | `DEL_Ids` sent? | Path |
|---|---|---|
| 3 — Order | **always** (the `\|\|`) | fast, always |
| 2 — Project, deliveries **have** a project | no; `CPR_Ids` only | slow scan, partly narrowed |
| 2 — Project, deliveries have **no** project (`CPR_Id = 0`) | no; nothing at all | **slowest — no filter whatsoever** |
| 1 — Collection | no; nothing at all | **slowest — no filter whatsoever** |

**CUS 5539 is in the worst row.** The log shows `@INT_Id=2` with `@CPR_Id=NULL` and `@CPR_Ids=NULL` — a customer configured for **project invoicing whose pending orders carry no project**. Line 784's `ic.CPR_Id != 0` therefore fails, so not even the weak `CPR_Ids` filter is sent. All three lists are empty and the scan runs completely unfiltered.

### Certain — `CUS_INT_Id` also multiplies how many scans one customer costs

`InvoiceCandidate.GetKey` ([line 157](ViewModels/Economy/Invoice/CreateInvoices/CreateInvoicesViewModel.cs:157)) decides how many separate `CreateInvoice` calls a customer generates, and each call pays the full 216 435-row scan:

| `CUS_INT_Id` | key | invoices per customer | scans |
|---|---|---|---|
| 1 — Collection | `CUS_Id` | 1 | 1 × 216 435 |
| 2 — Project | `CUS_Id-CPR_Id` | one per project/littera | **N × 216 435** |
| 3 — Order | `CUS_Id-DEL_Id` | one per order | N, but each is fast |

A Project-invoiced customer with 40 littera pays forty full backlog scans inside one user action. That alone can separate a "works" customer from a "hangs" customer at identical data volumes.

### Unresolved — and the contradiction that proves my model is incomplete

**Straight answer: I cannot currently explain why it is lightning fast for most `CUS_Id` and 300 s for a few. The explanation I have been building does not account for it, and I should not pretend otherwise.**

The logic is worth stating plainly, because it points at the gap:

1. §4b's cost is **office-wide** — it does not depend on `@CUS_Id` at all.
2. An office-wide cost predicts that **every** non-Order customer is equally slow.
3. Observation: most customers are lightning fast.
4. Therefore §4b is **not** the dominant cost, and the dominant cost is something **customer-specific that I have not found**.

The corrected arithmetic in §4b agrees: that scan is roughly one second warm, not 300. It is a real defect worth fixing, and it is real that `[Alla]` disarms the optimizer — but it cannot be the thing that produces a five-minute call.

So the two factors above (Order-type exemption, invoice-count multiplier) are solid, and they explain part of the spread. They do **not** explain a 300 s single invoice for a nine-delivery customer. Something else dominates, and the honest position is that further static code reading is now low-yield — I have read every SP in the chain and none of the customer-scoped ones has an obvious 300 s shape.

**One gap I did find while re-checking:** there is a **fifth** office-wide-driven SP I never counted. [`k2_inv2_create_rows_correction.sql:43`](Database/RedGateScripts/Stored%20Procedures/dbo.k2_inv2_create_rows_correction.sql:43) has the identical anti-pattern — driving table `DCO_DeliveryCorrection` filtered by `DCO_DIS_Id = 2 AND DCO.OFF_Id = @OFF_Id` (office-wide), with `DEL_CUS_Id = @CUS_Id` on the inner side of an `INNER LOOP JOIN`. **`DCO_DIS_Id = 2` was never in my §6 measurement list**, so this backlog is entirely unmeasured:

```sql
SELECT COUNT(*) AS pending_dco FROM DCO_DeliveryCorrection WHERE OFF_Id = 1 AND DCO_DIS_Id = 2;
```

Note it also joins `INR_InvoiceRow` on `INR_DEL_Id = DEL_Id` with no invoice restriction, pulling in *every invoice row ever written for that delivery* — which unlike the others **is** customer-shaped, and would punish a customer whose orders have been re-invoiced or corrected many times.

### Stop guessing — the answer is probably already recorded

Query Store is **enabled by default on Azure SQL Database**, and this is `opterproductionsweden1.database.windows.net`. That means the two 300 s executions from 2026-08-11 are most likely **already captured, per statement, with actual durations and wait breakdowns** — no reproduction needed, no production impact, one query.

This names the culprit statement directly:

```sql
SELECT   q.query_id,
         OBJECT_NAME(q.object_id)             AS proc_name,
         SUBSTRING(qt.query_sql_text, 1, 300) AS stmt,
         rs.count_executions,
         rs.avg_duration / 1000000.0          AS avg_sec,
         rs.max_duration / 1000000.0          AS max_sec,
         rs.avg_cpu_time / 1000000.0          AS avg_cpu_sec,
         rs.avg_logical_io_reads,
         rs.avg_physical_io_reads,
         rs.avg_rowcount,
         rsi.start_time, rsi.end_time
FROM     sys.query_store_runtime_stats rs
         JOIN sys.query_store_runtime_stats_interval rsi
           ON rsi.runtime_stats_interval_id = rs.runtime_stats_interval_id
         JOIN sys.query_store_plan  p  ON p.plan_id      = rs.plan_id
         JOIN sys.query_store_query q  ON q.query_id     = p.query_id
         JOIN sys.query_store_query_text qt ON qt.query_text_id = q.query_text_id
WHERE    rsi.start_time >= '2026-08-11T11:30:00+00:00'
AND      rsi.start_time <= '2026-08-11T13:00:00+00:00'
AND      (OBJECT_NAME(q.object_id) LIKE 'k2_inv%' OR q.object_id IS NULL)
ORDER BY rs.max_duration DESC;
```

And this settles "working vs waiting" — the question I have asked twice and still do not have an answer to:

```sql
SELECT   ws.wait_category_desc,
         OBJECT_NAME(q.object_id) AS proc_name,
         SUM(ws.total_query_wait_time_ms) / 1000.0 AS total_wait_sec,
         MAX(ws.max_query_wait_time_ms)   / 1000.0 AS max_single_wait_sec
FROM     sys.query_store_wait_stats ws
         JOIN sys.query_store_runtime_stats_interval rsi
           ON rsi.runtime_stats_interval_id = ws.runtime_stats_interval_id
         JOIN sys.query_store_plan  p ON p.plan_id  = ws.plan_id
         JOIN sys.query_store_query q ON q.query_id = p.query_id
WHERE    rsi.start_time >= '2026-08-11T11:30:00+00:00'
AND      rsi.start_time <= '2026-08-11T13:00:00+00:00'
GROUP BY ws.wait_category_desc, OBJECT_NAME(q.object_id)
ORDER BY total_wait_sec DESC;
```

How to read it:

| result | meaning | fix |
|---|---|---|
| `max_sec` ≈ 300 on a statement inside one `k2_inv2_*` SP, high `avg_cpu_sec`, `wait_category` = `Buffer IO` / none | it really is a scan — and `stmt` + `avg_rowcount` tells us *which* scan, ending the guesswork | §7 fix 2, scoped to whatever it names |
| `wait_category` = **`Lock`** with large `total_wait_sec` | §4c — the `SYS_System` exclusive lock, or another blocker. **Not customer-specific at all**, which would mean "some customers" is really "some *moments*" and my whole §3b framing is the wrong lens | §7 fix 5, promoted to first |
| huge `max_sec` vs small `avg_sec` on **one** `plan_id` | parameter sniffing — the plan was compiled for a different customer's parameters | `OPTION (RECOMPILE)` on the affected statement |

If Query Store has aged out that window, the same answers come from `SET STATISTICS PROFILE, IO, TIME ON` around a single `k2_inv2_create_invoice` call for CUS 5539 on a **restored copy** — that prints actual rows and reads per statement, which is what I have been trying to infer and getting wrong.

## 3c. Bad execution plans — the best remaining explanation

**Short answer: yes, and these particular SPs are close to a worst-case design for it.** This fits every observation that §4b cannot, including the one that keeps coming up: lightning fast for most `CUS_Id`, 300 s for a few, reproducibly, then fine again later.

### Why these SPs are unusually exposed

Two structural properties, both measurable:

| SP | `OR @param` predicates | forced hints |
|---|---|---|
| `_rows_delivery` | 5 | 3 |
| `_rows_price_item` | 6 | 1 |
| `_rows_add_service` | 5 | 2 |
| `_rows_direct_expense` | 5 | 2 |
| `_rows_expense` | 5 | 0 |
| `_rows_correction` | 1 | 2 |

And across the **whole** database — 3 750 stored procedures — exactly **2** use `OPTION (RECOMPILE)` or `OPTIMIZE FOR`. So there is essentially no plan-stability defence anywhere in this codebase.

**1. The `OR @param` predicates make these "catch-all queries."** This is the classic parameter-sniffing trap: one cached plan has to serve every parameter combination, and it is built from the cardinality estimates of whichever values happened to compile it. The specific killer is in `_rows_price_item` and `_rows_delivery`:

```sql
AND (DEL_Id = @DEL_Id OR @INT_Id <> 3)
```

* Compiled with `@INT_Id = 3, @DEL_Id = 12345` (an **Order**-invoiced customer): the optimizer sees an equality on the clustered key and estimates **~1 row** out of `DEL_Delivery`. It builds a plan sized for one row — nested loops, minimal memory grant, no parallelism.
* Reused with `@INT_Id = 1, @DEL_Id = NULL` (a **Collection** customer): the predicate is now `OR TRUE`, the true cardinality is five or six orders of magnitude higher, but the plan is unchanged.

Same for `(ISNULL(DEL_CPR_Id,0) = ISNULL(@CPR_Id,0) OR @INT_Id <> 2)` across Project vs non-Project customers, and `(COALESCE(DEL_REG_Id,-1) = @REG_Id OR COALESCE(@REG_Id,0) = 0)` across region-scoped vs not.

**This is exactly a per-`CUS_Id` explanation** — the relevant parameters (`@INT_Id`, `@CPR_Id`, `@DEL_Id`, `@REG_Id`) are all derived from customer configuration. Whether a given customer is fast depends on whether the cached plan was compiled for a customer whose *shape* resembles theirs.

**2. The forced hints remove the optimizer's safety net — this is the amplifier.** A 100 000× underestimate is normally survivable: the optimizer would have chosen a hash join, which degrades gracefully, and modern SQL Server can adapt. But `INNER LOOP JOIN`, `FORCESEEK` and `INDEX=` are *hard* directives. Under a forced loop join a bad estimate becomes 100 000+ nested iterations with **no possible escape**, and an undersized memory grant becomes a **tempdb spill** on every sort and hash. That is how a query that should take one second takes three hundred.

The hint comments say they exist *"to avoid deadlock"*. That may well be necessary — but the cost is that these queries cannot recover from a bad estimate, and nobody appears to have weighed that trade-off.

**3. Filtered-index statistics are a concrete, likely culprit.** `IX_PIT_DIS_Id_Filt` is defined `WHERE PIT_DIS_Id = 2`, and `IX_PAS_DIS_Id` similarly. Auto-update of statistics is triggered by modifications to the **whole table**, not to the filtered subset — so filtered-index statistics on a small subset of a large table are notoriously, silently stale. If SQL Server thinks there are 3 000 pending price items when there are 216 435, every plan built on that estimate is wrong by ~70×.

### Why this explains what §4b could not

| observation | bad-plan explanation |
|---|---|
| fast for most `CUS_Id`, 300 s for a few | the cached plan suits some customer *shapes* (`INT_Id`/`CPR_Id`/`REG_Id` combinations) and not others |
| same customer fails twice, 8 min apart | same plan still in cache |
| works again later / "sometimes" | plan evicted and recompiled with different sniffed values — Azure SQL failovers, index rebuilds, statistics updates and memory pressure all do this |
| a 9-delivery customer taking 300 s | duration is driven by estimate error and spills, **not** by the customer's row count — which is exactly the paradox §4b choked on |
| `[Alla]` slow, individual orders fast | `IDS_InvoiceDataSelection` holds 1 all-NULL row vs N real ids, against the **same cached plan** — actual cardinality diverges wildly from the estimate |

### A 30-second test that settles it

**Capture the bad plan from Query Store first** (§3b) — recompiling destroys the evidence. Then:

```sql
EXEC sp_recompile 'dbo.k2_inv2_create_rows_price_item';
EXEC sp_recompile 'dbo.k2_inv2_create_rows_delivery';
EXEC sp_recompile 'dbo.k2_inv2_create_rows_add_service';
EXEC sp_recompile 'dbo.k2_inv2_create_rows_direct_expense';
EXEC sp_recompile 'dbo.k2_inv2_create_rows_correction';
```

Then have support retry `[Alla]` for CUS 5539. `sp_recompile` only marks the procedure for recompilation — it takes a brief schema-modification lock, changes no data, and is safe on production between invoice runs.

* **Becomes fast** → it was a bad cached plan. Near-proof, and fix 6 below resolves it properly.
* **Still 300 s** → plans are not the issue; go back to the Query Store output for the real statement.

Either result is decisive, which is more than any of my static reading has managed.

### Supporting checks

```sql
-- Plan instability: same query_id with several plan_ids and wildly different durations.
SELECT   q.query_id, OBJECT_NAME(q.object_id) AS proc_name,
         COUNT(DISTINCT p.plan_id)          AS distinct_plans,
         MIN(rs.avg_duration)/1000000.0     AS best_avg_sec,
         MAX(rs.avg_duration)/1000000.0     AS worst_avg_sec,
         MAX(rs.max_duration)/1000000.0     AS worst_max_sec
FROM     sys.query_store_query q
         JOIN sys.query_store_plan p          ON p.query_id = q.query_id
         JOIN sys.query_store_runtime_stats rs ON rs.plan_id = p.plan_id
WHERE    OBJECT_NAME(q.object_id) LIKE 'k2_inv%'
GROUP BY q.query_id, OBJECT_NAME(q.object_id)
HAVING   MAX(rs.max_duration) > 5000000        -- worse than 5 s at least once
ORDER BY worst_max_sec DESC;

-- Stale statistics, especially the filtered ones.
SELECT   OBJECT_NAME(s.object_id) AS table_name, s.name AS stats_name,
         s.has_filter, s.filter_definition,
         sp.last_updated, sp.rows, sp.rows_sampled, sp.modification_counter
FROM     sys.stats s
         CROSS APPLY sys.dm_db_stats_properties(s.object_id, s.stats_id) sp
WHERE    OBJECT_NAME(s.object_id) IN
         ('PIT_PriceItem','DEL_Delivery','PAS_PriceAddService','DCO_DeliveryCorrection','PRI_Price','INR_InvoiceRow')
ORDER BY s.has_filter DESC, sp.last_updated;

-- Which optimizer protections are even available on this database.
SELECT name, compatibility_level FROM sys.databases WHERE name = DB_NAME();
SELECT name, value, is_value_default FROM sys.database_scoped_configurations
WHERE  name IN ('LEGACY_CARDINALITY_ESTIMATION','PARAMETER_SNIFFING',
                'QUERY_OPTIMIZER_HOTFIXES','LAST_QUERY_PLAN_STATS');
```

A low `compatibility_level` matters: Memory Grant Feedback and Adaptive Joins — the two features that would otherwise blunt exactly this failure mode — require 140/150+. If this database is running an older compat level for legacy-app safety, these SPs have no protection at all.

### Earlier hypotheses, now demoted

These remain possible but I want to be clear they are speculation, not findings, and both were offered before the arithmetic above was corrected:

* **Cold vs warm buffer pool.** 216 435 loop seeks touch a large share of `DEL_Delivery`. The *first* invoice in a run pays physical IO; later ones read the same pages from cache and may be 10–50× faster. Whichever customer is processed first would time out and the loop then aborts ([line 1339](ViewModels/Economy/Invoice/CreateInvoices/CreateInvoicesViewModel.cs:1339)) — and because the loop order is deterministic (`OFF_INU_Id` sorts by customer code, name or amount), **the same customer fails every retry**, which reads as "this customer is broken".
* **Parameter sniffing.** `k2_inv2_create_rows_price_item` is riddled with `OR @param` predicates, including `AND (DEL_Id = @DEL_Id OR @INT_Id <> 3)`. The cached plan is compiled from whichever customer's parameters happened to be first after a cache eviction. A plan compiled for an `INT_Id = 3` customer estimates ~1 row out of `DEL` and makes the forced loop join look cheap; reused for an `INT_Id = 1` or `2` customer it performs 216 435 iterations. Which customers are slow then depends on **who compiled the plan**, which changes on every failover, index rebuild or memory-pressure eviction — matching "sometimes it works".

Note the cold-cache hypothesis is now weaker than when I first raised it: if the scan is ~1 s warm and tens of seconds cold, "first caller pays" does not reach 300 s either. Parameter sniffing survives better, because a badly-sniffed plan can be *orders of magnitude* worse than a good one rather than merely cold — and the Query Store query above tests it directly.

Both are cheap to check:

```sql
-- Plan reuse and per-execution variance. Huge max_elapsed_time vs a small
-- min_elapsed_time on ONE cached plan = parameter sniffing.
SELECT  OBJECT_NAME(st.objectid, st.dbid) AS proc_name,
        qs.execution_count, qs.plan_generation_num,
        qs.min_elapsed_time/1000  AS min_ms,
        qs.max_elapsed_time/1000  AS max_ms,
        qs.total_elapsed_time/qs.execution_count/1000 AS avg_ms,
        qs.min_logical_reads, qs.max_logical_reads,
        qs.creation_time, qs.last_execution_time
FROM    sys.dm_exec_query_stats qs
        CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
WHERE   OBJECT_NAME(st.objectid, st.dbid) LIKE 'k2_inv2_create_rows%'
ORDER BY qs.max_elapsed_time DESC;

-- How the office's customers are distributed across the three modes.
-- Tells us how many customers are exposed at all, and which pay the N× multiplier.
SELECT  COALESCE(CUS.CUS_INT_Id, OFI.OFF_INT_Id) AS effective_INT_Id,
        COUNT(*)                                 AS customers
FROM    CUS_Customer CUS
        JOIN OFF_Office OFI ON OFI.OFF_Id = CUS.OFF_Id
WHERE   CUS.OFF_Id = 1
GROUP BY COALESCE(CUS.CUS_INT_Id, OFI.OFF_INT_Id);

-- Project-invoiced customers whose pending orders have NO project
-- = the CUS 5539 pattern. These are the ones to expect more reports about.
SELECT  DEL.DEL_CUS_Id, COUNT(*) AS pending_deliveries_without_project
FROM    DEL_Delivery DEL
        JOIN CUS_Customer CUS ON CUS.OFF_Id = DEL.OFF_Id AND CUS.CUS_Id = DEL.DEL_CUS_Id
WHERE   DEL.OFF_Id = 1 AND DEL.DEL_DIS_Id = 2 AND DEL.DEL_ReadyForInvoicing = 1
AND     DEL.DEL_ScheduledTemplate = 0
AND     CUS.CUS_INT_Id = 2 AND ISNULL(DEL.DEL_CPR_Id, 0) = 0
GROUP BY DEL.DEL_CUS_Id
ORDER BY pending_deliveries_without_project DESC;
```

Note that the two hypotheses point at different follow-ups: cold-cache means "it is always marginal and the first caller loses", sniffing means "add `OPTION (RECOMPILE)` or `OPTIMIZE FOR UNKNOWN` and the variance collapses". Fix 1 in §7 makes both moot by shrinking the scan 13×, which is why it is still the right first move either way.

## 4. Why the unrestricted query is slow

I audited every SP in the `k2_inv2_create_invoice` call chain. Only three pieces of work are **not** scoped to the customer, and they are the only possible explanations for a nine-delivery invoice taking 300 s:

| | Cost scales with | Status after measurement |
|---|---|---|
| §4a fan-out in `_rows_delivery` | customer's deliveries × price items | ❌ **ruled out** — measured 9 × 9 |
| §4b office-wide pending-item backlog | whole office, per invoice | ✅ **prime suspect** |
| §4c `SYS_System` lock | whatever else holds the lock | ✅ possible, if the time was spent waiting |

Everything else is customer- or invoice-scoped and cannot account for the duration: `k2_inv2_create_header`, `k2_inv2_get_last_invoice_date`, `k2_inv2_create_interests_and_reminder_fees`, `k2_inv2_filter_construction_items`, `k2_inv2_copy_prices_for_invoice` and `k2_inv2_complete_invoice` are all keyed on `@INH_Id` or `@CUS_Id`. `k2_inv2_get_next_invoice_number` takes `MAX(INH_InvoiceNo)` across the entire number series rather than the customer, but `IX_INH_INP_Id_incl (OFF_Id, INH_INP_Id) INCLUDE (INH_InvoiceNo)` covers it, so it should be an index-only scan of seconds at worst — worth a glance in the plan, not a suspect.

### 4a. `k2_inv2_create_rows_delivery` — row fan-out under `DISTINCT` (RULED OUT)

> **Measured 2026-08-11: `pending_deliveries = 9`, `del_x_pit_rows = 9`.** No fan-out exists for this customer and the candidate set is trivially small. This section is retained because the anti-pattern is real and will bite a genuinely large customer, but it is **not** the cause of BudWheels' timeout. Skip to §4b.

[`dbo.k2_inv2_create_rows_delivery.sql:35-85`](Database/RedGateScripts/Stored%20Procedures/dbo.k2_inv2_create_rows_delivery.sql:35)

```sql
SELECT DISTINCT ...
FROM   DEL_Delivery DEL WITH (FORCESEEK, INDEX=IX_DEL_ToBeInvoiced)
       ...
       LEFT OUTER JOIN PIT_PriceItem PIT
           ON PIT.OFF_Id = DEL.OFF_Id AND PIT_DEL_Id = DEL_Id AND PIT_PIY_Id <> 2
WHERE  ...
AND    EXISTS (SELECT PRI_Id FROM PRI_Price PRI ...
               AND (PRI.PRI_PIT_Id IS NULL OR PRI.PRI_PIT_Id = PIT_Id))
AND    NOT (DEL_CNT_Id IS NOT NULL AND EXISTS (... SHI_Shipment INNER LOOP JOIN DEL_Delivery ...))
```

The `LEFT JOIN` to `PIT_PriceItem` multiplies each delivery by its number of non-cost price items. The correlated `EXISTS` into `PRI_Price` references `PIT_Id` from that outer join, so it **cannot** be evaluated once per delivery — it runs once per (delivery × price item) pair, as does the consignment `NOT EXISTS`. The `DISTINCT` then collapses the fan-out afterwards. A customer with 5 000 pending deliveries averaging 20 price items means ~100 000 rows, each doing two or three index seeks, feeding a distinct sort.

The index hints make it worse rather than better. `IX_DEL_ToBeInvoiced` is
`(DEL_DIS_Id DESC, DEL_OrderDate, DEL_REG_Id, DEL_ScheduledTemplate, DEL_ReadyForInvoicing, DEL_ValidOrder, DEL_STS_Id, DEL_CUS_Id, OFF_Id)`.
The second key column's predicate is

```sql
AND (DEL_OrderDate <= @UntilDate OR @DirectInvoiceOnly = 1)
```

— non-sargable because of the `OR` on a parameter. So the seek can only use `DEL_DIS_Id = 2` and everything else, **including `DEL_CUS_Id = @CUS_Id` and `OFF_Id`**, degrades to a residual over the entire "to be invoiced" range. `FORCESEEK` plus a hard `INDEX=` hint means the optimizer cannot pick anything better even when statistics say it should. This pattern (`OR @param`) repeats on five predicates in this SP and in all four sibling SPs.

> Verify against the customer's plan before acting on this paragraph — the fan-out in the paragraph above it is certain from the SQL text, this one is the expected plan shape rather than an observed one.

### 4b. `k2_inv2_create_rows_price_item` scans the whole office backlog — real waste, **but NOT the root cause**

> ## ⚠️ MEASURED 2026-08-13 — ELIMINATED AS THE CAUSE
>
> The query was extracted and run standalone against production with CUS 5539's real parameters (see the hypothesis document §11a). Result:
>
> ```
> Table 'DEL_Delivery'.  Scan count 433093, logical reads 1384050, physical reads 0
> Table 'PIT_PriceItem'. Scan count 5,      logical reads 1711
> CPU time = 1296 ms,  elapsed time = 493 ms
> (0 rows affected)
> ```
>
> **Two conclusions, pointing opposite ways.**
>
> **The office-wide loop is empirically confirmed to exist.** 433 093 seeks into `DEL_Delivery` ≈ **2 × the 216 435 pending price items** (one index seek plus one key lookup per row), to produce **zero** rows for this customer. 1 384 050 logical reads to return nothing. The structural defect described below is real and measured.
>
> **But it is not the timeout.** 493 ms. My earlier "CONFIRMED ROOT CAUSE" heading was wrong — it rested on arithmetic I later corrected to "~1 second warm", and the measurement now agrees with the correction, not with the original claim. `physical reads 0` confirms a warm buffer pool, and `CPU 1296 ms > elapsed 493 ms` shows it even went parallel.
>
> So this stays on the fix list on **efficiency** grounds — 1.4 million logical reads per invoice to return nothing is worth removing, and it is the mechanism `[Alla]` disarms — but it does not explain a 300 s call. See the hypothesis document §11 for where the search moved.

> **Measured 2026-08-11 on `opterproductionsweden1` / `budwheelsse`:**
>
> | driving table | pending rows (`OFF_Id = 1`, `DIS_Id = 2`) |
> |---|---|
> | `PIT_PriceItem` | **216 435** |
> | `PAS_PriceAddService` | 0 |
> | `DEX_DirectExpense` | 0 |
> | `PEX_PriceExpense` | 39 |
> | of which `PIT` rows on **already-invoiced** orders (`DEL_DIS_Id = 3`) | **200 289 — 92.5 %** |
>
> Every invoice creation drives from **216 435 rows** and loop-seeks into `DEL_Delivery` once per row — to find CUS 5539's **9** deliveries. That is a 24 000× overshoot.
>
> ⚠️ **Correction to an earlier version of this document.** I originally wrote "216 435 seeks at ~1 ms is ~216 s, so the 300 s timeout is within reach." **That arithmetic was wrong and the conclusion it supported does not hold.** ~1 ms per seek assumes a physical random read every time, with no buffer-pool caching and no read-ahead. In reality a seek into `DEL_Delivery`'s clustered PK costs ~3–4 *logical* reads; 216 435 × 4 ≈ 870 000 logical reads, which a single core does in roughly **one second** once the pages are warm — and they warm up immediately, since the same scan repeats for every customer. Even fully cold, read-ahead over a clustered index makes this tens of seconds, not 300.
>
> So §4b is a genuine and serious inefficiency, and it is genuinely what `[Alla]` disarms — but **it is not, on its own, 300 seconds.** See §3b for what this means: the dominant cost is still unidentified.
>
> Two consequences worth noting:
> * **Only `PIT` matters.** `PAS` and `DEX` are empty and `PEX` has 39 rows. The fix has a single target: `k2_inv2_create_rows_price_item`.
> * **92.5 % of the scanned backlog is garbage** — price items still marked "to be invoiced" hanging off orders that were invoiced long ago. They can never produce a row for any future invoice, yet they are re-scanned on every invoice, forever.

#### Breakdown of the backlog (measured 2026-08-11)

| `PIT_PIY_Id` | rows total | on invoiced orders | oldest order | newest order |
|---|---|---|---|---|
| **3 — AddService** | **213 013** | **200 191** | 2013-10-16 | 2026-08-11 |
| 1 — Service | 3 423 | 98 | 2013-10-16 | 2026-11-20 |

`stale_with_no_active_price` = **200 289**, and 200 191 + 98 = **200 289** exactly. So **every single stale row has no active price** (`PRI_PTY_Id = 1`).

That makes the cleanup provably safe rather than merely plausible. [`k2_inv2_create_rows_price_item.sql:58`](Database/RedGateScripts/Stored%20Procedures/dbo.k2_inv2_create_rows_price_item.sql:58) joins prices with an **inner** join:

```sql
JOIN PRI_Price PRI
    ON  PRI.OFF_Id = PIT.OFF_Id
    AND PRI.PRI_PIT_Id = PIT_Id
    AND PRI.PRI_PTY_Id = 1
```

No active price → eliminated by the join → the row cannot appear on any invoice, now or ever. All 200 289 are dead weight being dragged through every invoice creation.

Three further observations:

* **It is AddService, not Cost.** I expected `PIY_Id = 2` (contractor costs, never invoiced to the customer). It is overwhelmingly `PIY_Id = 3` — *tillbehör* / add-on services. That changes the diagnosis of the leak: these are items that were meant to be invoiced and mostly were.
* **The leak is ACTIVE, not historical.** I initially guessed this was a legacy gap that had since been fixed. **That was wrong.** The per-month measurement shows a steady ~2 000 new stale rows every month for the last two years:

  | | | | | |
  |---|---|---|---|---|
  | 2026-06: 2 707 | 2026-05: 2 461 | 2026-04: 1 941 | 2026-03: 1 995 | 2026-02: 1 531 |
  | 2026-01: 1 318 | 2025-12: 2 014 | 2025-11: 2 233 | 2025-10: 2 330 | 2025-09: 2 076 |
  | 2025-06: 2 364 | 2025-05: 2 374 | 2024-11: 2 140 | 2024-10: 2 149 | 2024-09: 2 168 |

  (The two most recent buckets — 2026-08: 177 and 2026-07: 822 — are **undercounts, not a decline**. A row only becomes "stale" once its order reaches `DEL_DIS_Id = 3`, so recent months are still filling in as their orders get invoiced. Do not read them as the leak slowing down.)

  ~2 000/month ≈ 24 000/year, sustained since at least 2024, against `oldest_order = 2013-10-16`. That is entirely consistent with the 200 289 total. **This is a live product defect, and every Opter installation is accumulating it.** BudWheels is simply the first to cross the volume where it turns into a 300 s timeout.
* **The remaining ~16 147 rows are genuinely live.** 213 013 − 200 191 + 3 423 − 98 = 16 147, and note `newest_order = 2026-11-20` for Service — future-dated pre-booked orders. Cleanup must not touch these.

[`k2_inv2_create_rows_price_item.sql:33-68`](Database/RedGateScripts/Stored%20Procedures/dbo.k2_inv2_create_rows_price_item.sql:33):

```sql
FROM   PIT_PriceItem PIT
       INNER LOOP JOIN DEL_Delivery DEL
           ON  DEL.OFF_Id = PIT.OFF_Id
           AND DEL_Id     = PIT_DEL_Id
           AND DEL_CUS_Id = @CUS_Id          -- ← the only customer restriction, on the inner side
WHERE  PIT_DIS_Id  = 2
AND    PIT.OFF_Id  = @OFF_Id                 -- ← driving table filtered by office only
```

The driving table is filtered by **office and pending-status only, never by customer**. The customer restriction sits on the inner side of an `INNER LOOP JOIN` — a hard hint, so the optimizer may not reverse the order or switch to a hash join. Cost is therefore proportional to the **office-wide backlog of pending price items**, per invoice.

Same shape in `k2_inv2_create_rows_add_service` (`PAS`, forced loop + `INDEX=IX_PAS_DIS_Id`), `k2_inv2_create_rows_direct_expense` (`DEX`, forced loop), and `k2_inv2_create_rows_expense` (`PEX`, plain join — this one the optimizer can still fix).

Two things make this fit the measurements exactly:

**It explains a 300 s call for a 9-delivery customer.** The cost has nothing to do with `@CUS_Id`. A customer with nine pending deliveries pays exactly the same 216 435-row scan as a customer with nine thousand.

**The backlog is not self-limiting — and the numbers prove it.** Note line 66: `AND DEL_DIS_Id in (2,3)` — status 3 means *already invoiced*. So pending price items sitting on long-since-invoiced orders stay in the driving set forever. 200 289 of the 216 435 rows are in exactly that state. `k2_inv2_create_rows.sql:135` only promotes a price item to `PIT_DIS_Id = 3` if it actually landed on an invoice row (`INR_PIT_Id = PIT_Id`); anything filtered out earlier — by the `HAVING SUM(PRI_Price_Standard) >= 0` clause, by the negative-items-as-corrections office setting, or by never having a matching `PRI_PTY_Id = 1` price — keeps status 2 while its delivery moves to 3, and is then re-scanned on every invoice for the rest of the installation's life.

And the `[Alla]` link is precisely here. In all four SPs the `IDS_InvoiceDataSelection` `EXISTS` is the *only* selective thing available:

* **Individual orders:** `IDS_DEL_Id` holds real ids, so the optimizer can transform the `EXISTS` into a semi-join and drive the whole query from that tiny set. The `INNER LOOP JOIN` hint constrains only the `PIT`↔`DEL` pair; it does not stop `IDS` being placed first. Nine deliveries in, nine deliveries scanned. Fast.
* **`[Alla]`:** all three `ISNULL(…)` terms collapse to tautologies (§3), the semi-join is worthless, and the only remaining access path is the office-wide `PIT_DIS_Id = 2` / `PAS_DIS_Id = 2` / `DEX_DIS_Id = 2` scan with a per-row loop seek into `DEL_Delivery`. Whole backlog scanned to find nine deliveries.

That is a complete, self-consistent account of every observation: same duration regardless of customer size, fast with individual orders, slow with `[Alla]`, and worse for customers that happen to be tried when the backlog is large.

`k2_inv2_copy_prices_for_invoice` and `k2_inv2_complete_invoice` are scoped to `@INH_Id` and scale with invoice size only — they are not suspects.

### 4c. A global invoice lock held for the full 300 s — new in 2025.12

[`k2_inv2_create_invoice.sql:36-49`](Database/RedGateScripts/Stored%20Procedures/dbo.k2_inv2_create_invoice.sql:36), the **first statement** in the SP:

```sql
UPDATE SYS_System
    SET SYS_CreatingInvoiceLock = (SELECT MAX(INH_Id) FROM INH_InvoiceHeader)
```

`SYS_System` is a single-row table. [`InvoiceCreator.cs:187`](Server/Economy/Invoices/InvoiceCreator.cs:187) wraps the call in an explicit transaction:

```csharp
var transaction = _databaseAccess.OpenNewConnectionAndBeginTransaction();
```

So this takes an **exclusive lock on the single `SYS_System` row and holds it for the entire duration of invoice creation** — here, the full 300 seconds, plus however long the rollback takes afterwards. 31 stored procedures reference `SYS_System`; any of them running concurrently in another session will block for the duration.

This was introduced by **PR 12513** — commit `cc173884b8`, **2025-07-30**, Johan Hallberg, *"Fix able to create duplicate invoices, if invoice creation is run of different clients at the same time"*, AB#27099.

**Branch matrix**, verified by file content rather than commit containment (a cherry-pick would carry a different SHA, so `--contains` alone is not conclusive):

| branch | lock in `k2_inv2_create_invoice` | `SYS_CreatingInvoiceLock` column |
|---|---|---|
| `releases/2023.12` | ❌ | ❌ |
| `releases/2024.06` | ❌ | ❌ |
| **`releases/2025.06`** | ❌ | ❌ |
| `releases/2025.12` | ✅ | ✅ |
| `master` | ✅ | ✅ |

* **First shipped in ≈ `2025.07-01.798`.** The commit predates the `releases/2025.12` branch point (merge-base `c2fd8f8d18`, *"Version: 2025.07.1132+"*), so it was inherited rather than merged in — **every 2025.12 build contains it**, including BudWheels' `.256`.
* **Never revised since.** `k2_inv2_create_invoice.sql` has had no commits on `master` after `cc173884b8` — no follow-up, no refinement, no revert.
* **Not patched to 2025.06**, and no commit on that branch references PR 12513, AB#27099 or the column.

Two consequences, pulling in opposite directions:

1. **The duplicate-invoice bug PR 12513 fixed is still open on `releases/2025.06`.** Customers there can still get duplicate invoices when two clients invoice simultaneously.
2. **2025.06 does not carry the global serialization side effect.** Porting the fix as written would *introduce* the system-wide stall described above into that branch.

If it is ported, port a better implementation rather than this one — `sp_getapplock`, or a short dedicated transaction — for the reasons in fix 5. Note also that the guard itself is weak: the `IF @CreatingInvoiceLock < @MaxINH_Id` comparison re-reads `MAX(INH_Id)` inside the *same* transaction that just wrote it, so both reads see the same snapshot and the condition can only fire if a *different* code path commits an `INH_InvoiceHeader` row in between. And `RAISERROR(..., 16, 1)` does not abort the batch, so the SP continues executing after raising — the duplicate is actually prevented by the caller's `catch { Rollback }` in [`InvoiceCreator.cs:227`](Server/Economy/Invoices/InvoiceCreator.cs:227), not by the check. The real serialization comes from the exclusive lock alone, which is the expensive part.

It is not the root cause of the 300 s — but it converts one slow invoice into a system-wide stall, and it is the reason this case looks worse in 2025.12 than it would have before.

---

## 5. Two secondary defects that made diagnosis harder

**The timeout is swallowed.** [`InvoiceCreator.cs:227-230`](Server/Economy/Invoices/InvoiceCreator.cs:227):

```csharp
catch
{
    _databaseAccess.RollbackTransactionAndCloseConnection(transaction);
}
```

Bare `catch`, no logging, no rethrow — `result` stays `0`. The client sees `INH_Id == 0` and shows the generic `CIN_ERR_CreateInvoice` message. The user is never told it was a timeout, and there is no correlation id linking the client message to the ServerApi log entry. This is why support had to guess.

**The half-built period cannot be cleaned up.** After the timeout the period exists with partially created invoices. `DeleteInvoicePeriod` has a 120 s timeout against a create that has 300 s, so the delete is guaranteed to lose the race — visible three times in this log. The user is left with an orphan period and a hung UI.

Note also that `CreateCancelled` is only checked *between* customers ([`CreateInvoicesViewModel.cs:1308`](ViewModels/Economy/Invoice/CreateInvoices/CreateInvoicesViewModel.cs:1308)), never during a single `CreateInvoice` call. A 300 s call is therefore uncancellable — the progress dialog is genuinely frozen for five minutes.

---

## 6. What to tell BudWheels now

**Workaround:** invoice CUS 5539 (and any other customer that stalls) by selecting orders in the order list rather than leaving it at `[Alla]`. Selecting *any* subset populates `DEL_Ids` and takes the fast path. With nine deliveries this is not even burdensome for this customer.

### Still needed — one measurement, and one question

**(1) What are the 200 289 stale rows?** This determines whether the cleanup in fix 1 is safe, and it is the only thing blocking the cheapest available fix:

```sql
-- Breakdown of the stale backlog by price-item type.
-- PIY: Service=1, Cost=2, AddService=3, Option=4
SELECT PIT.PIT_PIY_Id,
       COUNT(*)                                                    AS rows_total,
       SUM(CASE WHEN DEL.DEL_DIS_Id = 3 THEN 1 ELSE 0 END)         AS on_invoiced_orders,
       MIN(DEL.DEL_OrderDate)                                      AS oldest_order,
       MAX(DEL.DEL_OrderDate)                                      AS newest_order
FROM   PIT_PriceItem PIT
       JOIN DEL_Delivery DEL ON DEL.OFF_Id = PIT.OFF_Id AND DEL.DEL_Id = PIT.PIT_DEL_Id
WHERE  PIT.OFF_Id = 1 AND PIT.PIT_DIS_Id = 2
GROUP BY PIT.PIT_PIY_Id
ORDER BY rows_total DESC;

-- Do the stale ones even have an invoiceable price? If not, they are provably dead weight.
SELECT COUNT(*) AS stale_with_no_active_price
FROM   PIT_PriceItem PIT
       JOIN DEL_Delivery DEL ON DEL.OFF_Id = PIT.OFF_Id AND DEL.DEL_Id = PIT.PIT_DEL_Id
WHERE  PIT.OFF_Id = 1 AND PIT.PIT_DIS_Id = 2 AND DEL.DEL_DIS_Id = 3
AND    NOT EXISTS (SELECT 1 FROM PRI_Price PRI
                   WHERE PRI.OFF_Id = PIT.OFF_Id AND PRI.PRI_PIT_Id = PIT.PIT_Id
                   AND   PRI.PRI_PTY_Id = 1);
```

If the bulk is `PIT_PIY_Id = 2` (Cost — contractor costs, never invoiced to the customer) or has no active price, those rows can never appear on any invoice and the cleanup is provably safe.

**(2) Optional confirmation — working or waiting?** §4b is confirmed by arithmetic, so this is now only needed if fix 1 does *not* resolve it (which would re-open §4c). During a live reproduction, while the create is running:

```sql
SELECT r.session_id, r.status, r.command, r.wait_type, r.wait_time, r.blocking_session_id,
       r.cpu_time, r.reads, r.logical_reads, r.total_elapsed_time,
       SUBSTRING(t.text, (r.statement_start_offset/2)+1,
                 ((CASE r.statement_end_offset WHEN -1 THEN DATALENGTH(t.text)
                   ELSE r.statement_end_offset END - r.statement_start_offset)/2)+1) AS current_stmt
FROM   sys.dm_exec_requests r
       CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) t
WHERE  r.session_id <> @@SPID AND r.session_id > 50;

SELECT * FROM sys.dm_os_waiting_tasks WHERE session_id > 50;
```

Expected, given the measurements: `wait_type` empty or `PAGEIOLATCH_*`, `logical_reads` in the millions, `cpu_time` climbing, `current_stmt` inside `k2_inv2_create_rows_price_item`. If instead it shows `LCK_M_*` with a non-zero `blocking_session_id` and near-zero `cpu_time`, §4c is also in play and needs the blocker identified.

Capture the plan too, but on a **restored copy** — this writes rows:

```sql
SET STATISTICS IO, TIME ON;
-- then the create with the same parameters as the log line, inside a transaction you roll back
```

---

## 7. Suggested fixes, in order of value

Re-ordered twice: once when the 9-delivery measurement killed §4a, and again now that §4b is confirmed and scoped to a single SP.

0. **Run the Query Store queries in §3b first.** They are read-only, need no reproduction, and will most likely name the actual 300 s statement and say whether it was working or lock-waiting. Everything below is ordered on the assumption that §4b is the dominant cost — and §3b now shows that assumption **does not survive its own arithmetic**. Do not spend effort on 1–3 before this, because if the answer is `wait_category = Lock` then item 5 is the real fix and items 1–3 will change nothing.

1. **Clear the 200 289 stale `PIT_DIS_Id = 2` rows** — no deploy needed, and worth doing on its own merits: it removes a 13.4× (216 435 → 16 147) waste multiplier from every invoice this installation will ever create. The §6 breakdown confirms these rows have no active price and cannot reach any invoice, so it is safe by construction.

   ⚠️ **But do not promise it fixes the timeout.** I previously wrote that this "should bring the call well under the timeout on its own." Given the corrected arithmetic in §4b, that claim is unsupported: if the 300 s is not actually this scan, removing 92.5 % of it will make the call faster without making it *pass*. Treat this as a real efficiency win with an **unknown** effect on the reported symptom until item 0 confirms the scan is the culprit.

   **The `NOT EXISTS` clause is load-bearing — do not drop it.** My first draft filtered on `DEL_DIS_Id = 3` alone. That is wrong: adding extra parts to an *already-invoiced* order is a supported flow (see the `DELETE` at [`k2_inv2_create_rows_price_item.sql:83`](Database/RedGateScripts/Stored%20Procedures/dbo.k2_inv2_create_rows_price_item.sql:83) and its comment "Order must be invoiced (or directly paid) before extra parts are added"). Such items are legitimately pending on a `DEL_DIS_Id = 3` order and **must not** be cleared. Today the two sets happen to coincide exactly — there are currently zero late-added extras awaiting invoicing — but relying on that coincidence would silently destroy revenue the moment someone adds one.

   **Step 1 — record exactly which rows will change, so the operation is precisely reversible.** Without this, "undo" is guesswork.

   ```sql
   CREATE TABLE dbo.PIT_CleanupLog_20260813 (
       PIT_Id         int          NOT NULL PRIMARY KEY,
       OFF_Id         int          NOT NULL,
       PIT_DEL_Id     int          NULL,
       PIT_PIY_Id     int          NOT NULL,
       old_PIT_DIS_Id int          NOT NULL,
       new_PIT_DIS_Id int          NOT NULL,
       logged_at      datetime2(0) NOT NULL CONSTRAINT DF_PITCleanup_at DEFAULT SYSUTCDATETIME()
   );

   INSERT dbo.PIT_CleanupLog_20260813 (PIT_Id, OFF_Id, PIT_DEL_Id, PIT_PIY_Id, old_PIT_DIS_Id, new_PIT_DIS_Id)
   SELECT PIT.PIT_Id, PIT.OFF_Id, PIT.PIT_DEL_Id, PIT.PIT_PIY_Id, PIT.PIT_DIS_Id, 1
   FROM   PIT_PriceItem PIT
          JOIN DEL_Delivery DEL ON DEL.OFF_Id = PIT.OFF_Id AND DEL.DEL_Id = PIT.PIT_DEL_Id
   WHERE  PIT.OFF_Id = @OFF_Id
   AND    PIT.PIT_DIS_Id = 2
   AND    DEL.DEL_DIS_Id = 3
   AND    PIT.PIT_PIY_Id IN (1, 3)
   AND    NOT EXISTS (SELECT 1 FROM PRI_Price PRI
                      WHERE PRI.OFF_Id = PIT.OFF_Id AND PRI.PRI_PIT_Id = PIT.PIT_Id
                      AND   PRI.PRI_PTY_Id = 1);
   ```

   **Step 2 — apply, driven off the log so the two can never diverge.** Batched; each batch autocommits, so it is interruptible and resumable.

   ```sql
   SET NOCOUNT ON;
   DECLARE @batch int = 1;
   WHILE @batch > 0
   BEGIN
       UPDATE TOP (5000) PIT
       SET    PIT_DIS_Id = L.new_PIT_DIS_Id        -- 2 (ToBeInvoiced) -> 1 (DoNotInvoice)
       FROM   PIT_PriceItem PIT
              JOIN dbo.PIT_CleanupLog_20260813 L
                ON L.PIT_Id = PIT.PIT_Id AND L.OFF_Id = PIT.OFF_Id
       WHERE  PIT.PIT_DIS_Id = 2;                  -- idempotent: already-done rows are skipped
       SET @batch = @@ROWCOUNT;
       RAISERROR('retired %d', 0, 1, @batch) WITH NOWAIT;
   END
   ```

   **Undo, if ever needed:**

   ```sql
   UPDATE PIT SET PIT_DIS_Id = L.old_PIT_DIS_Id
   FROM   PIT_PriceItem PIT
          JOIN dbo.PIT_CleanupLog_20260813 L ON L.PIT_Id = PIT.PIT_Id AND L.OFF_Id = PIT.OFF_Id
   WHERE  PIT.PIT_DIS_Id = L.new_PIT_DIS_Id;
   ```

   **Measured scope on the 2026-08-13 copy (`OFF_Id = 1`)** — what each guard actually does:

   | | rows |
   |---|---|
   | **changed by the cleanup** | **200 313** |
   |  — of which `PIY 3` AddService | 200 215 |
   |  — of which `PIY 1` Service | 98 |
   | excluded: order not yet invoiced (`DEL_DIS_Id <> 3`) | **16 250** ← the live backlog, left alone |
   | excluded: has an active price (`PRI_PTY_Id = 1` exists) | 0 |
   | excluded: `PIY 2` Cost | 0 |
   | excluded: not attached to an order (`PIT_DEL_Id IS NULL`) | 0 |

   216 563 pending = 200 313 retired + 16 250 untouched. Reconciles exactly.

   The last three guards currently exclude nothing at BudWheels, but keep them: they are what make the statement safe to hand to another installation without re-deriving the analysis.

   **One interaction checked and cleared.** [`k2_ods_update_crediting_consignment_children`](Database/RedGateScripts/Stored%20Procedures/dbo.k2_ods_update_crediting_consignment_children.sql:169) folds child price-item statuses into a parent consignment's `DEL_DIS_Id`, and its `@ConsignedOrders` set is **not** filtered by `DEL_DIS_Id` — so in principle changing price items from 2 to 1 could cause an already-invoiced consignment child to be recomputed. **Measured exposure at BudWheels: zero** — none of the 200 313 rows sit on a consignment or a consignment child. For other installations, add `AND DEL.DEL_CNT_Id IS NULL AND NOT EXISTS (SELECT 1 FROM SHI_Shipment SHI WHERE SHI.OFF_Id = DEL.OFF_Id AND SHI.SHI_DEL_Id = DEL.DEL_Id AND (SHI.SHI_DEL_Id_Consignment IS NOT NULL OR SHI.SHI_DEL_Id_StatisticsConsignment IS NOT NULL))` and measure before dropping it.

   **Status: `DoNotInvoice = 1`, not `Invoiced = 3`.** Settled, with precedent in the product. [`k2_sw_sca_set_handled.sql:69`](Database/RedGateScripts/Stored%20Procedures/dbo.k2_sw_sca_set_handled.sql:69) already does exactly this, for exactly this reason:

   ```sql
   -- Make rejected price items "not to be invoiced" to avoid affecting performance when creating invoices
   UPDATE PIT_PriceItem
   SET PIT_DIS_Id = 1 -- Do not invoice
   ```

   So retiring an unusable price item to status 1 to keep it out of the invoice-creation scan is an established in-product pattern — this cleanup is the bulk application of a remedy Opter already uses one row at a time. Supporting reasons:

   * **Semantically true.** These rows have no active price and can never reach an invoice. `Invoiced = 3` would assert an invoice that does not exist.
   * **Much smaller blast radius.** `PIT_DIS_Id = 3` is read in 22 stored procedures and 7 C# comparisons, several of which join through to invoice rows and would find orphans. `PIT_DIS_Id = 1` is read in **two** SPs, neither of which is harmed: `k2_ods_update_crediting_consignment_children` only folds child statuses into a parent, and `k2_sw_sca_set_handled` is the precedent above.
   * **No reversion path.** `k2_inv2_reset_rows` only touches rows at `PIT_DIS_Id = 3`, so status 1 is immune to being flipped back to 2 by a later period deletion. Status 3 would not be.

   This is a mitigation, not a fix. The leak is **active at ~2 000 rows/month** (§4b), so the backlog regrows — but slowly enough to be useful: from 16 147 back to 216 435 would take roughly 8 years, and back to ~100 000 roughly 3.5 years. The exact pain threshold between 16 147 (works) and 216 435 (300 s) is unknown, so treat "buys years, not months" as the claim, and re-run the cleanup periodically until fix 3 ships.
2. ⚠️ **Downgraded — customer-scoping the driving table will NOT fix this.** An earlier version of this list called it "the actual fix", on the belief that the office-wide scan was the cost. The measurements disprove that: a customer with **zero** pending price items completes in about a second *with the office-wide backlog at full size*. The office-wide driving table is real waste worth perhaps a second per invoice, and it is what the `[Alla]`/individual-orders difference turns on — but the timeout comes from the **customer's own** surviving rows, which customer-scoping does not reduce. Still worth doing for tidiness; not the fix.

   **The genuine open question is the superlinearity.** Cost scales as roughly rows^1.42, and at 14 369 surviving rows the statement spends ~23 ms and ~1 800 logical reads *per row*. That is wildly disproportionate for a row that only needs a `DEL` seek, a `PRI` seek, four small dimension joins and an `IDS` probe. Something in the plan is scanning where it should seek, or a spool is being rewound. **If that one operator can be fixed, every customer becomes immune permanently — which is a better outcome than managing the data forever.** Getting it needs the actual execution plan with estimated-vs-actual row counts, which is now easy on the local copy:

   ```sql
   ALTER DATABASE SCOPED CONFIGURATION SET LAST_QUERY_PLAN_STATS = ON;
   -- re-run the SP for CUS 5539, then:
   SELECT ps.query_plan
   FROM   sys.dm_exec_query_stats qs
          CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
          CROSS APPLY sys.dm_exec_query_plan_stats(qs.plan_handle) ps
   WHERE  OBJECT_NAME(st.objectid, st.dbid) = 'k2_inv2_create_rows_price_item';
   ```

   Save as `.sqlplan` and look for the operator whose actual rows dwarf its estimate. Note compatibility level is **110** (SQL 2012 CE, no adaptive joins, no memory grant feedback), so raising it may itself change the plan — worth testing on the copy.

   > **⚠️ Plan capture was attempted five times on 2026-08-13 and abandoned. Read this before trying again** — each obstacle is real and costs a ~7-minute run to rediscover:
   >
   > | approach | why it failed |
   > |---|---|
   > | `dm_exec_query_plan_stats` after the run | the statement's `DROP TABLE #inr_tmp` invalidates its cached plan, so it is gone from `dm_exec_query_stats` by the time you look — even with `LAST_QUERY_PLAN_STATS = ON` |
   > | `SET STATISTICS XML ON` on the isolated statement | forces **DOP 1**. The statement runs at DOP 10 in the SP; serialised it exceeded a 900 s timeout without finishing |
   > | Extended Events → `ring_buffer` | `sys.dm_xe_sessions` only lists **running** sessions and stopping one discards the buffer — read the target *before* `STATE = STOP` |
   > | XE, extracting with `.value('…showplan_xml/value','nvarchar(max)')` | `.value()` strips XML markup and returns an empty string. Use `.query('(data[@name="showplan_xml"]/value/*)[1]')` |
   > | `SET SHOWPLAN_XML ON` via `sp_executesql` | returns only the trivial plan for the `sp_executesql` call, not the inner statement |
   >
   > **What would probably work:** an XE session with an **`event_file`** target (not ring_buffer) written to a path the SQL service account can write — `SELECT SERVERPROPERTY('InstanceDefaultLogPath')` — reading it afterwards with `sys.fn_xe_file_target_read_file` and extracting via `.query()`. Or simply open the `.sqlplan` from SSMS with *Include Actual Execution Plan* while running the SP interactively, which sidesteps all of the above.
   >
   > **This is optional work.** The root cause, the fix, and the affected-customer list are all established without it. The plan would only tell us whether the superlinearity can be engineered away, i.e. whether item 1 stops being a recurring chore.
3. **Stop the leak — confirmed live at ~2 000 rows/month, and creation-side** (§4b). 96.2 % of the backlog is auto-created `AddService` price items stamped `PIT_DIS_Id = 2` by [`CustomerPrice.cs:822`](Order/Price/CustomerPrice.cs:822) *before* pricing is attempted, which then never get a price and never get retired. Not BudWheels-specific — needs a product fix plus a cleanup script for all customers.

   **Two places this could be fixed. Prefer the second.**

   *Upstream (riskier):* don't stamp `ToBeInvoiced` until a price exists, and hard-remove auto price items that fail to price. This is the honest fix but it sits in the middle of the pricing engine, interacts with `PCC_AcceptFailure` and the deleted-item-revival logic at [line 788](Order/Price/CustomerPrice.cs:788), and risks changing prices on live orders. Not a support-driven change.

   *At invoicing (safer, and where I would start):* next to the existing status promotion at [`k2_inv2_create_rows.sql:135`](Database/RedGateScripts/Stored%20Procedures/dbo.k2_inv2_create_rows.sql:135), retire any price item on this invoice's orders that provably cannot be invoiced. Bounded to the current invoice, so it cannot run away:

   ```sql
   UPDATE  PIT
   SET     PIT_DIS_Id = 1                      -- DoNotInvoice
   FROM    PIT_PriceItem PIT
           JOIN INR_InvoiceRow INR WITH (INDEX=IX_INR_INH_Id_incl)
             ON  INR.OFF_Id     = PIT.OFF_Id
             AND INR.INR_DEL_Id = PIT.PIT_DEL_Id
   WHERE   INR.OFF_Id     = @OFF_Id
   AND     INR.INR_INH_Id = @INH_Id
   AND     PIT.PIT_DIS_Id = 2
   AND     PIT.PIT_PIY_Id IN (1, 3)             -- ← see warning below
   AND     NOT EXISTS (SELECT 1 FROM PRI_Price PRI
                       WHERE PRI.OFF_Id     = PIT.OFF_Id
                       AND   PRI.PRI_PIT_Id = PIT.PIT_Id
                       AND   PRI.PRI_PTY_Id = 1)
   ```

   **`PIT_PIY_Id IN (1, 3)` is a required guard.** Do **not** let this touch `PIY_Id = 2` (Cost) items — contractor cost items legitimately have no customer-facing `PTY = 1` price, and forcing them to `DoNotInvoice` could break contractor invoicing and resource crediting. The measured backlog contains only `PIY` 1 and 3, so restricting to those is both sufficient and safe.

   One open question for whoever owns pricing: this assumes a delivery reaching invoicing has finished pricing. If price calculation can still be pending on an order that is being invoiced, an item could be retired just before it would have been priced. Worth confirming before merge.

   **One query still needed to pick the right fix.** It separates the two possible mechanisms. Columns verified to exist in the customer's build `b2ee9c7831`: `PIT_PIY_Id`, `PIT_Invoice`, `PIT_Credit`.

   ```sql
   -- The CASE must be computed in a CTE and grouped by name; SQL Server rejects a
   -- subquery inside a GROUP BY expression (Msg 144).
   WITH stale AS (
       SELECT  PIT.PIT_Id,
               PIT.PIT_PIY_Id,
               PIT.PIT_Invoice,
               PIT.PIT_Credit,
               CASE WHEN EXISTS (SELECT 1 FROM PRI_Price PRI
                                 WHERE PRI.OFF_Id = PIT.OFF_Id
                                 AND   PRI.PRI_PIT_Id = PIT.PIT_Id)
                    THEN 'priced once (has PRI at another PTY)'
                    ELSE 'never priced (no PRI at all)'
               END AS bucket
       FROM    PIT_PriceItem PIT
               JOIN DEL_Delivery DEL
                 ON DEL.OFF_Id = PIT.OFF_Id AND DEL.DEL_Id = PIT.PIT_DEL_Id
       WHERE   PIT.OFF_Id = 1
       AND     PIT.PIT_DIS_Id = 2
       AND     DEL.DEL_DIS_Id = 3
       AND     NOT EXISTS (SELECT 1 FROM PRI_Price PRI
                           WHERE PRI.OFF_Id = PIT.OFF_Id
                           AND   PRI.PRI_PIT_Id = PIT.PIT_Id
                           AND   PRI.PRI_PTY_Id = 1)
   )
   SELECT   bucket, PIT_PIY_Id, PIT_Invoice, PIT_Credit, COUNT(*) AS row_count
   FROM     stale
   GROUP BY bucket, PIT_PIY_Id, PIT_Invoice, PIT_Credit
   ORDER BY row_count DESC;
   ```

   And, for whatever lands in the "priced once" bucket, which price types they actually carry — `PTY_PriceType`: `Price=1, Invoice=2, Storage=3, ResourcePrel=4, ResourceBill=5`. A large `PTY_Id = 2` count with `linked_to_invoice_row > 0` is direct proof these were invoiced and only the status update was missed:

   ```sql
   SELECT   PRI.PRI_PTY_Id,
            COUNT(DISTINCT PIT.PIT_Id) AS distinct_price_items,
            COUNT(*)                   AS pri_rows,
            SUM(CASE WHEN PRI.PRI_INR_Id IS NOT NULL THEN 1 ELSE 0 END) AS linked_to_invoice_row
   FROM     PIT_PriceItem PIT
            JOIN DEL_Delivery DEL ON DEL.OFF_Id = PIT.OFF_Id AND DEL.DEL_Id = PIT.PIT_DEL_Id
            JOIN PRI_Price PRI    ON PRI.OFF_Id = PIT.OFF_Id AND PRI.PRI_PIT_Id = PIT.PIT_Id
   WHERE    PIT.OFF_Id = 1 AND PIT.PIT_DIS_Id = 2 AND DEL.DEL_DIS_Id = 3
   AND      NOT EXISTS (SELECT 1 FROM PRI_Price P2
                        WHERE P2.OFF_Id = PIT.OFF_Id AND P2.PRI_PIT_Id = PIT.PIT_Id
                        AND   P2.PRI_PTY_Id = 1)
   GROUP BY PRI.PRI_PTY_Id
   ORDER BY pri_rows DESC;
   ```

#### Result (measured 2026-08-11) — it is a creation-side leak

| bucket | `PIY_Id` | `PIT_Invoice` | `PIT_Credit` | rows |
|---|---|---|---|---|
| **never priced (no PRI at all)** | **3 AddService** | **1** | **1** | **192 632** |
| priced once (PRI at another PTY) | 3 AddService | 1 | 1 | 7 559 |
| never priced (no PRI at all) | 1 Service | 1 | 1 | 86 |
| priced once (PRI at another PTY) | 1 Service | 1 | 1 | 10 |
| priced once (PRI at another PTY) | 1 Service | 1 | 0 | 2 |

Totals to 200 289 exactly. **96.2 % are `AddService` price items that never had a price row of any kind**, and they are all flagged `PIT_Invoice = 1` — "should be invoiced".

**This corrects an earlier suggestion in this document.** I proposed reusing the [`k2_inv2_reset_rows.sql:56`](Database/RedGateScripts/Stored%20Procedures/dbo.k2_inv2_reset_rows.sql:56) `CASE` as the target state, on the guess that the leaked rows would be `PIT_Invoice = 0`. They are not — they are `PIT_Invoice = 1`, and that `CASE` maps `PIT_Invoice = 1` to status **2**, i.e. it would leave them exactly where they are. That `CASE` is not the fix.

The origin is the **auto-price-item path**, not the invoicing path. [`CustomerPrice.cs:822`](Order/Price/CustomerPrice.cs:822) stamps the status *before* pricing is attempted:

```csharp
pit.PIT_DIS_Id = Tables.DIS_DeliveryInvoiceStatus.ToBeInvoiced;   // line 822 — unconditional
...
if (priceItemPrice.CalculatePrices(0, true, testPriceObjects, 0)  // line 863
    && priceItemPrice.GetPrices().Length > 0)
{ ...  del.PriceItems.Add(pit);  ... }                            // line 883 — success only
else
{ pit.Delete = true; }                                            // line 892 — already carries DIS_Id = 2
```

Supporting evidence that failed auto items are *retained* rather than removed: [line 788](Order/Price/CustomerPrice.cs:788) deliberately searches **outside** `ActiveItems` for auto price items with `p.Delete` already set — the comment says *"Search not only in ActiveItems to be able to find deleted auto service price item"* — and [line 715](Order/Price/CustomerPrice.cs:715) has an explicit `PCC_AcceptFailure` escape that tolerates a price item which failed to price.

So `OFF_CreatePriceItemsAutoPricing` creates one auto price item per active PCC per order; those whose PCC yields no price for that order are stamped `ToBeInvoiced`, marked deleted, and evidently persist. At one or two unused `AddService` PCCs per order this matches the observed ~2 000/month precisely.

> **Confidence boundary:** the *shape* above is established from the code. I have **not** traced the save layer to prove exactly how a `Delete = true` auto price item ends up as a committed row, so treat the precise persistence path as unproven. It does not affect the recommended fix, which is deliberately independent of it.
4. **Log the timeout.** Replace the bare `catch` in `InvoiceCreator.CreateInvoice` with a logged catch that preserves the `SqlException` and surfaces "timeout" distinctly from "failed" to the user. Cheapest change in the list, and it makes the next case diagnosable without a log-analytics dig.
5. **Move the `SYS_System` update out of the long transaction**, or replace it with `sp_getapplock` / a short dedicated transaction (§4c). Not the root cause here, but it is why one slow invoice stalls every other invoice operation in the installation, and why the delete attempts had no chance.
6. **Raise `DeleteInvoicePeriod`'s timeout above `CreateInvoice`'s**, or block deletion while a create is in flight with a clear message, so support is not left with an undeletable period.
6. **Add plan stability to the catch-all queries** (§3c) — cheap, targeted, and currently the most likely thing to actually fix the reported symptom. The five `_rows_*` SPs carry 5–6 `OR @param` predicates each and, across all 3 750 SPs in the database, only 2 use any plan-stability hint at all.

   ```sql
   -- On the INSERT/SELECT in each of the five _rows_* SPs:
   OPTION (RECOMPILE)
   ```

   `OPTION (RECOMPILE)` is the right tool here rather than `OPTIMIZE FOR UNKNOWN`: these statements run at most a few times per invoice, so the compile cost is negligible against a 300 s worst case, and recompiling per execution lets the optimizer see the *actual* `@INT_Id` / `@CPR_Id` / `@DEL_Id` / `@REG_Id` for this customer plus the real `IDS_InvoiceDataSelection` contents. It also makes the `[Alla]` and individual-order paths get appropriately different plans instead of sharing one.

   Two things to do alongside it:
   * **Update statistics on the filtered indexes** (`IX_PIT_DIS_Id_Filt`, `IX_PAS_DIS_Id`) with `FULLSCAN`, and consider a maintenance job — filtered-index statistics do not auto-update reliably, since the trigger is whole-table modifications.
   * **Re-examine the forced hints.** They were added to avoid deadlocks, but `INNER LOOP JOIN` / `FORCESEEK` also remove the optimizer's ability to recover from a bad estimate (§3c). If `OPTION (RECOMPILE)` gives good estimates, some hints may no longer be needed — but do not remove them without re-testing the deadlock scenarios they were added for.

7. **Restructure `k2_inv2_create_rows_delivery`** so the `PIT` existence test is a correlated `EXISTS` rather than a `LEFT JOIN` under `DISTINCT` (§4a). Not BudWheels' problem, but it will be someone's. Needs care around the "orders with only cost price items should not be invoiced" rule from TFS 15239.
8. **~~Make `[Alla]` pass the delivery ids anyway~~** — dropped. With fix 2 in place this is unnecessary, and on its own it would only move the cliff to whatever count the threshold is set at.

Items 4, 5 and 6 are independent of the query work. Item 1 unblocks the customer today; items 2 and 3 are the ones that belong in the product.
