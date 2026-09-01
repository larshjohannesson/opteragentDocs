# A2B Finland — "Fel vid sparande av order" / POD save fails with FK violation

**Customer:** A2B Finland Oy (`a2bfi`, CustomerId 71)
**Reported orders:** 650667, 651047
**Log window analysed:** 2026-07-15 18:22 → 2026-08-14 13:24 (still ongoing)
**Data source:** `query_data (28).csv` (4 097 rows, all customers)

---

## 1. Summary

`k2_ods_save_pod` is called with a `POD_POT_Id` that does not exist **for the office the
delivery belongs to**. The FK `FK_POD_ProofOfDelivery_POT_ProofOfDeliveryType` is composite
(`POD_POT_Id`, `OFF_Id`) → (`POT_Id`, `OFF_Id`), so a POD type id that is perfectly valid in
office 1 is invalid in office 2 and the INSERT is rejected.

Every single A2B failure has the same shape:

| Parameter | Value | Count |
|---|---|---|
| `@OFF_Id` | `2` | 3369 / 3369 |
| `@POD_POT_Id` | `1` | 3369 / 3369 |
| `@POD_POT_Id_Sub` | `0` | 3369 / 3369 |

`POT_Id` is a table-wide `IDENTITY(1,1)` with PK `(POT_Id, OFF_Id)`, so `POT_Id = 1` is the
*first office's* "Delivered" type. An office created later gets ids continuing from `MAX(POT_Id)`,
never `1`. **Office 2 in `a2bfi` therefore has no `POT_Id = 1`** — every POD the app reports
for an office-2 delivery is rejected.

3369 errors originate from only **113 distinct deliveries** (top offender `DEL_Id 1551731`:
125 attempts). The mobile app never gets an id back, so it re-sends the same POD on every
sync — a permanent retry storm (see §4).

## 2. Where it breaks — code

`Mobile/Server/Server/App/Actions/ApplyChangesToK2Handler.cs:1433` `HandleShipmentPod`:

```csharp
pod.OFF_Id = del.OFF_Id;          // line 1527 — office comes from the DELIVERY
pod.POD_DEL_Id = del.DEL_Id;
pod.POD_SHI_Id = SHI_Id;

if (shp.SHP_SPT_Id != 0)
{
    pod.POD_POT_Id = Math.Abs(shp.SHP_SPT_Id);   // line 1533 — id comes from the DEVICE
}
else
{
    pod.POD_POT_Id = shp.SHP_DPT_Id;             // line 1537
}
```

The office is taken from the delivery, the POD-type id from the device, and **the two are never
reconciled**. There is no validation that `POD_POT_Id` exists in `del.OFF_Id`, and no fallback
to the delivery office's default type. The neighbouring `HandleShipmentScanPod`
(`ApplyChangesToK2Handler.cs:1842-1868`) at least guards `POT_Id == 0`; `HandleShipmentPod`
has no guard at all.

Why the device sends an office-1 id — `Mobile/Server/Server/App/Actions/DispatchDataToSyncDataLoader.cs:593-689`:

* The device is primarily loaded with the POD types of **its own** office
  (`_cache.PodTypes.GetAll(activeDeviceInformation.OFF_Id)`, line 596).
* Other offices' types are only appended when the client is Xamarin **and** app version ≥ 2.4.50
  **and** `offices.Count > 1` (line 628). Anything older/native only ever sees its own office's types.
* Even when other offices are included, the list is flat and deduplicated by `POT_Id` alone
  (line 637) — nothing forces the app to pick the type belonging to the shipment's office.
  The default flag (`DPT_Default`) on office 1's `POT_Id = 1` makes it the natural pick.

Unchanged in `master`, `releases/2025.12` and `releases/2025.06` — this is not a regression in a
specific build, which matches the log data spanning `20250600.000412` → `20251200.000268`.

## 3. Same defect class in the desktop client

`ViewModels/Pod/PodViewModel.cs:1387` `CreatePOD` — the POD window loads deliveries from **all**
offices (`drOFF_Id` per row, line 1267) but resolves the POD type against the **window's** office:

```csharp
pod.OFF_Id = del.OFF_Id;                              // line 1390 — delivery office
...
var defaultPODType = _cache.PodTypes.GetDefault(OFF_Id);  // line 1412 — window/user office
pod.POD_POT_Id = defaultPODType?.POT_Id ?? 0;
```

The selectable POD-type list is loaded the same way (`PodViewModel.cs:152`). So a dispatcher
sitting in office 1 who registers a POD on an office-2 delivery produces exactly the same FK
violation, surfacing as **"Fel vid sparande av order. Kontakta din systemadministratör"**
(`ORD_ERR_SavingOrder`, raised from `ViewModels/Order/Order/OrderWindowViewModel.cs:3148` when
`SaveOrder()` returns `Failure`).

Note that `Order/Data/POD.cs:699` does it correctly — `_cache.PodTypes.GetDefaultOrFirstId(Delivery.OFF_Id)`.

> **Not yet confirmed for this case:** the CSV contains no `Opter.Main.Client` rows for `a2bfi`,
> so the dispatcher's exact client-side exception on orders 650667/651047 is not in this data set.
> The office/POD-type mismatch above is the only mechanism that reproduces that dialog for these
> deliveries, but a client log pull for `a2bfi` (Assembly = `Opter.Main.Client`, same time window)
> should be done to nail it down.

## 4. Secondary damage — retry storm and orphan attachments

On failure `HandleShipmentPod` catches, logs and `return false` (line 1649-1661), leaving
`shp.SHP_Id = 0`. The app therefore re-sends the same `ShipmentPod` forever (~30 retries per
delivery in this window; 125 for the worst one).

Worse, the POD images are saved **before** the POD:

```csharp
del.Attachments.Add(att);
att.Save();                 // lines 1579 / 1599 — committed
...
pod.Save(...);              // line 1614 — throws
```

Every retry writes a new `DEA_DeliveryAttachment` row and a new FileStorage file that is never
linked to a POD. This is very likely the source of the `k2_ods_delete_attachment` /
`FK_POD_ProofOfDelivery_DEA_DeliveryAttachment` errors seen from `Opter.Main.Client` for other
customers in the same data set (cf. branch `features/23812_ErrorDeletingDeaPodConnectionDea_patch_2025.12`).

## 5. A separate defect in the same method (other customers)

631 rows in the same data set fail on `FK_POD_ProofOfDelivery_ADR_Address` instead — all with
`@POD_ADR_Id_Lookup = 0`, and with nonsense keys such as `@OFF_Id = 0, @POD_DEL_Id = 0` or
`@OFF_Id = 1, @POD_DEL_Id = 1..6`. Affected: Fraktlogistik AB (352), AS Moss Transportforum (249),
Collicare AS (23), Ressel Rederi, A3 Transport, Leverera.

Root cause: `del.LoadByShipmentId(OFF_Id, SHI_Id)` (line 1506) did not find a delivery, but the
code carries on and saves a POD anyway. `k2_ods_save_pod` then tries to substitute the address:

```sql
IF @POD_ADR_Id_Lookup IS NULL OR @POD_ADR_Id_Lookup = 0 BEGIN
    SELECT  @POD_ADR_Id_Lookup = DEL_ADR_Id_End
    FROM    DEL_Delivery
    WHERE   DEL_Id = @POD_DEL_Id AND OFF_Id = @OFF_Id
END
```

No row matches, so the variable stays `0` and the FK on `ADR_Address` fires. Same underlying
theme: unvalidated input is pushed to the database and the constraint becomes the error handler.

## 6. Verification SQL (run against `a2bfi`)

```sql
-- Which POD types exist per office? Expect no POT_Id = 1 for OFF_Id = 2.
SELECT OFF_Id, POT_Id, POT_Name, POT_Default, POT_Active, POT_AvailableInMobileDevice
FROM   POT_ProofOfDeliveryType
ORDER  BY OFF_Id, POT_Order;

-- When was office 2 set up / which offices exist?
SELECT OFF_Id, OFF_Name FROM OFF_Office ORDER BY OFF_Id;

-- Which office are the affected deliveries in, and which office are the drivers' devices in?
SELECT d.OFF_Id, d.DEL_Id, d.DEL_ExternalId
FROM   DEL_Delivery d
WHERE  d.DEL_Id IN (1551731, 1553070, 1553064, 1558632, 1560963);

SELECT ACD_Id, OFF_Id, ACD_EMP_Id, ACD_AppVersion FROM ACD_ActiveDevice ORDER BY OFF_Id;

-- Orphan POD attachments produced by the retry storm
SELECT dea.OFF_Id, dea.DEA_Id, dea.DEA_DEL_Id, dea.DEA_FSF_Id
FROM   DEA_DeliveryAttachment dea
LEFT   JOIN PDA_ProofOfDeliveryAttachment pda
       ON pda.PDA_DEA_Id = dea.DEA_Id AND pda.OFF_Id = dea.OFF_Id
WHERE  dea.DEA_FFT_Id = 5 /* POD */ AND pda.PDA_Id IS NULL
       AND dea.DEA_DEL_Id IN (1551731, 1553070, 1553064, 1558632);
```

## 7. Proposed fix

**Immediate (customer):** create the missing POD types in office 2 (or register the affected
devices/resources in the office that owns the deliveries). Once a valid type exists the queued
PODs will sync on the next attempt — but the duplicate attachments created meanwhile need cleaning up.

**Code — mobile server** (`ApplyChangesToK2Handler.HandleShipmentPod`):

1. Resolve the POD type **in the delivery's office** before assigning:
   look up the incoming id in `del.OFF_Id`; if it is not found, map it (by
   `POT_Name` / main type) or fall back to `_cache.PodTypes.GetDefaultOrFirstId(del.OFF_Id)`,
   and log a warning with `SHI_Id`, device office and the rejected id. Never write an id from
   another office.
2. Bail out early with a clear log entry when `del.DEL_Id == 0` / `del.OFF_Id == 0`
   (fixes §5) and when the resolved `POT_Id == 0`, mirroring `HandleShipmentScanPod:1856`.
3. Move `att.Save()` after a successful `pod.Save()`, or delete the attachments in the `catch`,
   so failed syncs stop accumulating files (§4).

**Code — desktop client** (`ViewModels/Pod/PodViewModel.cs`):
use `del.OFF_Id` instead of the window `OFF_Id` for the default POD type (line 1412) and for the
selectable POD-type list, consistent with `Order/Data/POD.cs:699`.

**Code — sync** (`DispatchDataToSyncDataLoader.cs:628`): the multi-office POD-type list is
version-gated and flat. Either always send other offices' types (they already carry
`OFF_Id`/`DPT_ExternalId`) and have the app filter by the shipment's office, or accept that the
server must map — item 1 above is the safety net either way.

**Optional — SP hardening** (`k2_ods_save_pod`): when the `DEL_Delivery` lookup finds nothing,
`@POD_ADR_Id_Lookup` should be set to `NULL` rather than left at `0`, so the failure is a clean
NULL instead of an FK violation.
