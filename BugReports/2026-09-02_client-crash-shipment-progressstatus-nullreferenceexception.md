# Bug report: Client crashes with unhandled `NullReferenceException` in `Shipment.get_ProgressStatus()`

- **Reported:** 2026-09-02
- **Reported by:** Lars Johannesson (found while triaging `Opter.Main.Client` crash logs; confirmed independently against a wider 30-day export)
- **Assembly:** `Opter.Main.Client`
- **Area:** Order/Delivery data model (`Opter.Main.Order.Data.Shipment`), Delivery window commanding (`Opter.Main.ViewModels.Order.Order.WPF.DeliveryViewModel`)
- **Data source:** Azure Log Analytics `Logs_CL`, export `query_data (36).csv` (866 rows total, 52 matching `Shipment.get_ProgressStatus`, 2026-08-04 → 2026-09-02)

## Summary

`Shipment.get_ProgressStatus()` throws an unhandled `NullReferenceException` that crashes the
whole client. It is not tied to a specific user action — it fires as background noise via WPF's
`CommandManager.RequerySuggested`, which re-evaluates every bound command's `CanExecute` on
almost any UI input (mouse move, keypress, focus change) while a Delivery/Order window is open.
That makes it effectively unreproducible by deliberate testing and explains why it has
accumulated steadily rather than in a single burst.

Over the last 30 days it hit **52 times across 13 customers**, on **17 of 30 days**, spanning
**16 different client build versions** (`20251200.000252` through `20260101.000524`) — i.e. both
the 2025.12 and the newer 2026.01 release lines. This is a long-standing logic bug, not a recent
regression.

## Impact (evidence)

All 52 rows are `Level = Error`, `Exception` containing `Opter.Main.Order.Data.Shipment.get_ProgressStatus()`.

- **Customers (13):** Easyavfall AS (12), T Grupp Oü (8), Dölnord Åkeri AB (7), TNR Spedisjon AS (5), Transportsentralen Tromsø SA (4), Toten Transport AS (4), A2B Finland Oy (3), Hillblom Transport AB (2), ACX Logistics AB (2), BudXpress (2), Apollo bud (1), Stockholm Lyft & Transport AB (1), Wiréns Åkeri AB (1)
- **Date range:** 2026-08-04 → 2026-09-02, hits on 17 separate days (not a single spike)
- **Versions:** 16 distinct builds across both `releases/2025.12` (`20251200.*`) and a newer `20260101.*` line — confirms this predates the observation window by a wide margin and isn't scoped to one release

Every occurrence shares the same call chain:

```
Shipment.get_ProgressStatus()
Shipment.get_IsCredited()                                                        (Order/Data/Shipment.cs:6656)
DeliveryViewModel.<CanRouteOptimizeAddresses>b__..._0(Shipment shi)             (ViewModels/Order/Order/WPF/DeliveryViewModel.cs:4619)
  -- del.Shipments.ActiveItems.Any(shi => shi.IsCredited)
DeliveryViewModel.CanRouteOptimizeAddresses / CanRouteOptimizeAddressesCommand  (DeliveryViewModel.cs:4578-4633)
Telerik.Windows.Controls.RadSplitButton.CanExecuteApply()
System.Windows.Input.CommandManager.RaiseRequerySuggested
[System.Windows.Threading.ExceptionWrapper.TryCatchWhen -> WPF dispatcher pump]
```

The compiler-generated lambda suffix varies across builds (`b__605_0` vs `b__613_0`, and 10 rows
where JIT inlining collapses the lambda frame straight into `get_IsCredited()`) — this is the
same call site every time; the numeric suffix just shifts as unrelated code earlier in the file
changes between builds. 18 of the 52 crashes unwind all the way to `App.Main()`; the other 34
happen inside a nested dispatcher pump (e.g. a modal dialog's own `Window.ShowDialog()` message
loop). Both are the same UI-thread crash, just different window contexts — either way, the whole
client goes down.

`CanRouteOptimizeAddresses` decides whether the "Optimize route" split-button on the Delivery
window is enabled, by checking `del.Shipments.ActiveItems.Any(shi => shi.IsCredited)`. Since
`CommandManager` re-evaluates this on nearly any UI input while the window is open, the crash
requires no deliberate action from the user beyond having a Delivery window open with shipments.

## Root cause

Two unguarded null-derefs inside the same getter, [`Shipment.cs:294-367`](../../Main/Order/Data/Shipment.cs):

**1. `Delivery.OFF_Id` — [Shipment.cs:326](../../Main/Order/Data/Shipment.cs)**
```csharp
if (Delivery.OFF_Id != 0 && Delivery.DEL_PST_Id != 0)
```
`Delivery` is `Parent as Delivery` ([Shipment.cs:462](../../Main/Order/Data/Shipment.cs)) — a
plain `as`-cast, `null` whenever `Parent` is null or isn't a `Delivery`. No guard here.

The sibling property immediately above it in the same file, `IsAtEndingStatus`
([Shipment.cs:255](../../Main/Order/Data/Shipment.cs)), has the correct pattern:
```csharp
if (Delivery != null && Delivery.OFF_Id != 0 && Delivery.DEL_PST_Id != 0)
```
`ProgressStatus` is simply missing the `Delivery != null` check that its neighbor already has.

**2. `pst.DeliveryLifeCycle.DLC_STS_Id_Start` — [Shipment.cs:332-335](../../Main/Order/Data/Shipment.cs)**
```csharp
var pst = _cache.ServiceTypes.GetSingle(Delivery.OFF_Id, Delivery.DEL_PST_Id);
...
if (STS_Id == 0)
{
    STS_Id = pst.DeliveryLifeCycle.DLC_STS_Id_Start;   // <-- pst used here, unguarded
}
```
`pst` can be `null` on a cache miss — proven by the method's own next use of the same variable
three lines later ([Shipment.cs:339](../../Main/Order/Data/Shipment.cs)):
```csharp
if (pst != null && pst.DeliveryLifeCycle.STS_Id_Assignment != null)
```
The method's author clearly knew `pst` could be null and guarded the second use, but not the
first.

Either gap alone reproduces exactly the observed crash shape: a bare `NullReferenceException`
with no inner exception and no further Opter frames below `get_ProgressStatus()`.

## Suggested fix

Both gaps are simple, low-risk guards, consistent with patterns already used elsewhere in the
same file:

- [Shipment.cs:326](../../Main/Order/Data/Shipment.cs): add `Delivery != null &&` to the
  condition, matching `IsAtEndingStatus`.
- [Shipment.cs:332-335](../../Main/Order/Data/Shipment.cs): guard the first use of `pst` the same
  way the second use already is (move the `pst != null` check up, or wrap the `STS_Id == 0`
  block in it).

No behavior change for the working case — `ProgressStatus` already falls through to
`ProgressStatuses.Free`/`Assigned` when the relevant data isn't resolvable; these guards just
make the same fallback apply when `Delivery` or `pst` is unexpectedly null instead of crashing.

## Recommendations

1. Implement both guards above.
2. Add regression coverage for `ProgressStatus`/`IsCredited` across all five `ProgressStatuses`
   in the happy path, plus a case with `Delivery == null` (detached shipment) and a case where
   the service-type cache lookup misses.
3. Worth a follow-up investigation (separate from this fix) into *why* `Delivery`/`pst` end up
   null in practice in production — e.g. a shipment transiently detached from its Delivery during
   an in-progress edit/delete racing with `CommandManager`'s requery pass, or a `DEL_PST_Id`
   pointing at a service type no longer present in cache. Root-causing that could reveal a second,
   deeper bug worth its own ticket.
4. Given the bug spans both `releases/2025.12` and the `2026.01` line already, confirm the fix
   target(s) and whether a backport is needed once it lands on `master`.
