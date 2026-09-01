# Two open invoice periods with the same period number (552761), duplicate invoice 583468

**Customer version:** `releases/2025.06`
**Reported:** invoice period created ~07:45; result was **two open periods both numbered 552761**, invoice **583468** present in both with identical customer and amount, subsequent invoice numbers **alternating** between the two periods, and the remaining invoices in one period **zeroed**. Nothing in Opter Logs at that time. User noticed nothing unusual during creation.

**Verdict: a concurrency race — two invoice-creation runs executing at the same time.** This is precisely the failure mode PR 12513 was written to fix, and that PR is **not** on `releases/2025.06`. However, PR 12513 would only have prevented *part* of what this customer is seeing; the duplicate period number is unfixed on **every** branch, including `master`.

---

## 1. Why we know it was genuine concurrency, not a sequential double-run

The **alternating invoice numbers** are the giveaway.

A sequential double-run (first run finishes, second starts) would produce period A with invoices 1…N and period B with N+1…M — contiguous blocks, not alternation. Alternation can only happen if two loops were **interleaved in time**, each taking the next number as it went.

And invoice **583468 appearing in both periods** with the same customer and amount means both loops computed the same next invoice number before either committed.

---

## 2. Three separate races, one per symptom

### 2a. Two periods with the same `INP_PeriodNo`

[`k2_inv_create_invoice_period.sql:82`](Database/RedGateScripts/Stored%20Procedures/dbo.k2_inv_create_invoice_period.sql:82):

```sql
SELECT @INP_PeriodNo = ISNULL(MAX(INP_PeriodNo) + 1, @INS_StartNo)
FROM   INP_InvoicePeriod
WHERE  INP_IPT_Id = @IPT_Id AND OFF_Id = @OFF_Id AND (...)

INSERT INTO INP_InvoicePeriod (... INP_PeriodNo ...) VALUES (... @INP_PeriodNo ...)
```

Read-then-insert with **no transaction, no lock, and no unique constraint**. `INP_InvoicePeriod` has only `PK_INP_InvoicePeriod` on `(INP_Id, OFF_Id)` — nothing enforces uniqueness of `INP_PeriodNo`. Two concurrent calls both read `MAX = 552760`, both compute `552761`, both insert.

The "only one open period" guard at line 44 is raced the same way: both sessions evaluate `@OpenCount = 0` before either has inserted, so neither returns `-1`. Hence **two open** periods.

### 2b. Invoice 583468 in both periods, then alternating numbers

[`k2_inv2_get_next_invoice_number.sql:49`](Database/RedGateScripts/Stored%20Procedures/dbo.k2_inv2_get_next_invoice_number.sql:49):

```sql
SELECT @INH_InvoiceNo = ISNULL(MAX(INH_InvoiceNo)+1, @INS_StartNo)
FROM   INP_InvoicePeriod INP
       JOIN INH_InvoiceHeader INH ON INH_INP_Id = INP_Id AND INH.OFF_Id = INP.OFF_Id
WHERE  INP_INS_Id = @INS_Id AND INP.OFF_Id = @OFF_Id
```

Same read-then-insert race. Note the `MAX` spans the whole **number series**, not the period — so both periods draw from one sequence. Two interleaved loops therefore alternate: A takes 583469, B takes 583470, A takes 583471… and at the start both took 583468 before either committed.

### 2c. The zeroed invoices

Whichever run reached a given customer first marked its deliveries and price items as invoiced (`DEL_DIS_Id = 3`, `PIT_DIS_Id = 3`) in [`k2_inv2_create_rows.sql:109`](Database/RedGateScripts/Stored%20Procedures/dbo.k2_inv2_create_rows.sql:109). Every row-creation SP filters on status `2`, so the second run found nothing left to add and produced **invoice headers with no rows** — zero amount. That is the "nollade" invoices, and it is why one period looks hollowed out.

---

## 3. Would PR 12513 have prevented this?

PR 12513 (`cc173884b8`, 2025-07-30, AB#27099) changed four things. Mapping them onto the three symptoms:

| symptom | prevented by PR 12513? | why |
|---|---|---|
| duplicate invoice 583468 | ✅ **yes** | the `SYS_System` exclusive lock serialises `k2_inv2_create_invoice`, so two sessions cannot both read the same `MAX(INH_InvoiceNo)` |
| alternating invoice numbers | ✅ mostly | same serialisation — numbering would run sequentially |
| zeroed invoices in one period | ⚠️ partly | serialised, the second run still finds nothing to invoice; it would still create empty invoices, just without number collisions |
| **two periods numbered 552761** | ❌ **no** | PR 12513 **did not touch `k2_inv_create_invoice_period`**. That SP is byte-identical on 2023.12, 2024.06, 2025.06, 2025.12 and `master` |

So the instinct is correct — this is the bug PR 12513 was aimed at, and on 2025.06 it is unmitigated. But **the period-number race is still open on every branch we ship**, including `master`. A customer on 2025.12 could still end up with two periods sharing a number; they would just not get duplicate invoice numbers inside them.

PR 12513's fourth change is directly relevant to cleanup — see §5.

---

## 4. What let one user action run twice

The most likely trigger is a **double invocation of the create command**. Two contributing facts:

**The command permits concurrent execution.** [`CreateInvoicesViewModel.cs:947`](ViewModels/Economy/Invoice/CreateInvoices/CreateInvoicesViewModel.cs:947):

```csharp
_createInvoicesCommand = new AsyncRelayCommand(
    () => Task.Run(() => ResultOk = CreateInvoices()),
    _ => CanCreateInvoices
);
```

`AsyncRelayCommand` derives from `AsyncCommand` in `AsyncAwaitBestPractices.MVVM` **10.0.0**. That type takes an `allowsMultipleExecutions` parameter which defaults to `true`; the wrapper in [`Common/AsyncRelayCommand.cs`](Common/AsyncRelayCommand.cs) does not pass it, and nothing anywhere in the solution sets it. So the library provides no re-entrancy guard here.
*(Worth confirming the v10 default against the package before acting — but the repo definitely never overrides it.)*

**The only guard is set too late.** `CanCreateInvoices => InvoiceCount > 0 && CreateNotStarted && INP_Id != -1`, and `CreateStatus` is not set to `InProgress` until [line 1186](ViewModels/Economy/Invoice/CreateInvoices/CreateInvoicesViewModel.cs:1186) — *after* `ValidateData()` at line 1157, which can display several modal dialogs (future accounting date, old accounting date, open-period confirmation). A modal dialog pumps messages, so throughout that window the button remains enabled and `CanCreateInvoices` still returns `true`.

Other possibilities worth ruling out with the customer: two users invoicing simultaneously, or a retried cloud call. I found no retry logic in `ServerProxy`, and `CreateInvoicePeriod` passes no `NumberOfRetriesOnTimeout` (so it defaults to 0) — but `DatabaseAccess.ExecuteStatement` does retry up to 3 times on SQL error 1204/1205, which is worth keeping in mind.

---

## 5. Helping the customer correct the data

⚠️ **This is financial data. Take a backup first, and have someone who knows the customer's accounting sign off — invoice numbers may already have been exported, reported or sent.** Do not run repair SQL blind.

The intended route is to delete the bogus period, which cascades to its invoices via `k2_inv_delete_invoice_period`. Two obstacles on 2025.06:

1. **`k2_inv_delete_invoice_period` is gated** by `k2_inv_check_delete_period` → `@StopperCount`. One of its checks is `k2_inv_check_exists_later_periods`, which blocks deletion when a later period exists.
2. **PR 12513 relaxed exactly that check — and 2025.06 does not have it.** The PR added an early exit:

   ```sql
   IF @INP_InvoiceCount = 0 BEGIN
       RETURN 0
   END
   ```

   i.e. an **empty** period became deletable even when later periods exist. That is the "det går att ta bort den avsett om det finns senare perioder" line in the PR description.

Since the duplicate period here contains invoices (the zeroed ones), that exemption would not apply even on 2025.12 — the invoices have to go first. Practical sequence, to be validated on a restored copy before production:

1. Identify both periods and their invoices, and confirm which period is the keeper (the one with non-zero amounts and correctly sequenced numbers).
2. Confirm none of the zeroed invoices have been exported or paid — check `IEL_InvoiceExportLog` and `SLT`/reskontra transactions.
3. Delete the zeroed invoices individually (`k2_inv_delete_invoice`), which also resets the affected orders' invoice status.
4. Delete the now-empty duplicate period.
5. Verify the surviving period's numbering, and check whether the gaps left in the invoice number series are acceptable to the customer's accounting.

If the customer needs the empty-period deletion path, **backporting only PR 12513's `k2_inv_check_exists_later_periods` change to 2025.06 is low risk** — it is a pure relaxation of a guard and carries none of the locking side effects.

---

## 6. Recommendations

1. **Do not backport PR 12513's `SYS_System` lock to 2025.06 as written.** It serialises all invoice creation behind an exclusive lock on a single-row table held for the entire transaction (up to 300 s), which is an active problem for another customer — see `support-budwheels-create-invoice-period-timeout.md` §4c. If the duplicate-invoice protection is wanted on 2025.06, implement it with `sp_getapplock` or a short dedicated transaction instead.
2. **Fix the period-number race — this is unfixed on `master`.** The minimum viable fix is a unique constraint on `INP_InvoicePeriod (OFF_Id, INP_IPT_Id, INP_PeriodNo)` (plus `INP_REG_Id` where `OFF_SeparateInvoiceNumbersForRegions = 1`), so the second insert fails loudly instead of silently corrupting. Better: wrap the read-and-insert in a transaction with `UPDLOCK, HOLDLOCK` on the `MAX` read.
3. **Same treatment for the invoice-number race** in `k2_inv2_get_next_invoice_number`, which has the identical read-then-insert shape and is currently protected only as a side effect of the `SYS_System` lock on 2025.12+.
4. **Add a re-entrancy guard to the create command.** Either pass `allowsMultipleExecutions: false`, or set `CreateStatus = InProgress` *before* `ValidateData()` runs its modal dialogs. Cheap, client-side, and closes the most likely trigger.
5. **Investigate why nothing appeared in Opter Logs.** Both runs apparently succeeded, so nothing was logged — but a duplicate period number is a condition worth detecting and logging in its own right. Related: the bare `catch` in [`InvoiceCreator.cs:227`](Server/Economy/Invoices/InvoiceCreator.cs:227) — added by PR 12513 — swallows exceptions during invoice creation without logging them, so if anything *did* go wrong it would be invisible.

Items 2 and 3 are the real fix and belong on `master` with a cherry-pick to the supported release branches. Item 4 alone would likely have prevented this incident.
