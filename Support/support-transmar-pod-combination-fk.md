# Transmar AB — "Fel vid sparande av order" on POD delete (self-referencing FK)

**Customer:** Transmar AB
**Reported by:** Mats Nyström ("har börjat få dylika meddelanden nu och då" — getting this dialog now and then)
**Dialog:** "Fel vid sparande av order. Kontakta din systemadministratör" (`ORD_ERR_SavingOrder`)
**Log window analysed:** 120 days, all customers, `Assembly == "Opter.Main.Client"`, `Exception contains "FK_POD_ProofOfDelivery"`
**Data source:** `query_data (29).csv` (156 rows) — this is the client-side follow-up query recommended in `support-a2bfi-pod-fk-office-mismatch.md` §3

---

## 1. Summary

Two unrelated defects are mixed together in this data set:

| FK constraint | Rows | Customers | Status |
|---|---|---|---|
| `FK_POD_ProofOfDelivery_POD_ProofOfDelivery` (self-reference) | 28 | **Transmar only** | **Not fixed anywhere** — new finding |
| `FK_POD_ProofOfDelivery_DEA_DeliveryAttachment` | 126 | Espeland Transport, M-Leverans, Mantum Holding, BDX Företagen, GDL, Sigurd & Ola Grimstad, Stokkebekk Transport, Transportören, Trygve Bengtsons Åkeri, Utengen, Bø Varetransport, Ntex, Gävle Fjärrfrakt, Transportsenteret, Transportsentralen Nord, Moss Transportforum, Collicare, Ekdahl Miljö, Bendiks Birkeland, Turlid | Fixed on **master only** (AB#23812) — **not backported** |
| `FK_POD_ProofOfDelivery_ADR_Address` | 2 | Best | Same defect class as `support-a2bfi...md` §5; low volume, not investigated further here |

Transmar's ticket is specifically about the first row — a distinct, still-open bug.

## 2. Transmar's bug — `POD_POD_Id_Combination` self-reference

All 28 Transmar errors have the identical shape. Client stack trace:

```
Opter.Main.Client.Order.Order.frmOrder.SaveOrder()
 → Order.Data.Delivery.Save(...)
 → Server.Orders.OrderDataStructure.SaveOrder(...)
 → Server.Orders.OrderDataStructure.InnerDeletePod(OFF_Id, POD_Id, transaction)
 → exec k2_ods_delete_pod @OFF_Id=1, @POD_Id=386392
 → SqlException 547: DELETE conflicted with SAME TABLE REFERENCE constraint
   "FK_POD_ProofOfDelivery_POD_ProofOfDelivery"
```

`OrderDataStructure.cs:455-465` iterates `delivery.ProofOfDeliveries` and calls `InnerDeletePod` (defined at line 4325) for any POD marked `SaveActions.Delete` — which happens generically whenever a POD's item ViewModel is removed from its bound collection (`Common/ViewModel/ViewModelBaseWithModel.cs`), no dedicated "delete POD" code path exists.

### Root cause

`POD_ProofOfDelivery` has a self-referencing FK (`Database/RedGateScripts/Tables/dbo.POD_ProofOfDelivery.sql:44`):

```sql
ALTER TABLE [dbo].[POD_ProofOfDelivery] ADD CONSTRAINT [FK_POD_ProofOfDelivery_POD_ProofOfDelivery]
FOREIGN KEY ([POD_POD_Id_Combination], [OFF_Id]) REFERENCES [dbo].[POD_ProofOfDelivery] ([POD_Id], [OFF_Id])
```

`POD_POD_Id_Combination` is stamped when **order consolidation** copies a POD from a sub-order into the consolidated order (`Order/Data/Delivery.cs:17134-17143`, `AddOrdersToConsolidation`, gated on `OCT_CopyPods`):

```csharp
if (oct.OCT_CopyPods)
{
    foreach (var pod in subOrder.PODs.ActiveItems)
    {
        var podCopy = pod.Copy(this, CopyMode.NewWithPrice);
        podCopy.POD_POD_Id_Combination = pod.POD_Id;   // points back at the sub-order's original POD
        podCopy.POD_Descr = pod.POD_DEL_Id.ToString() + " " + podCopy.POD_Descr;
        PODs.Add(podCopy);
    }
}
```

So: sub-order A has POD 386392. When A is consolidated into order B (with `OCT_CopyPods` on), B gets a *copy* of that POD whose `POD_POD_Id_Combination = 386392`. If POD 386392 is later deleted from sub-order A — e.g. the dispatcher redoes/removes the POD on the original order — `k2_ods_delete_pod` (`Database/RedGateScripts/Stored Procedures/dbo.k2_ods_delete_pod.sql`) does:

```sql
DELETE FROM POD_ProofOfDelivery WHERE POD_Id = @POD_Id AND OFF_Id = @OFF_Id
```

with no step that clears or cascades any row whose `POD_POD_Id_Combination` still points at `@POD_Id` — unlike `DEL_POD_Id_Lookup`, which the same SP does null out for the delivery, and unlike the PDA/DEA attachment cleanup it already performs. The consolidated copy's dangling reference blocks the delete → SQL error 547 → surfaces to the desktop user as the generic "Fel vid sparande av order" dialog.

This is **not** the AB#23812 defect (that fix only touches `PDA_ProofOfDeliveryAttachment`/`DEA_DeliveryAttachment` cleanup in `k2_ods_delete_attachment`, called from inside `k2_ods_delete_pod`, and never addresses `POD_POD_Id_Combination`) — Transmar's version `20250600.000506` predates AB#23812 anyway, but even current `master` still has this gap.

### Fix

In `k2_ods_delete_pod`, before the final `DELETE FROM POD_ProofOfDelivery`, clear the dangling reference on any combination-copy rows, e.g.:

```sql
UPDATE  POD_ProofOfDelivery
SET     POD_POD_Id_Combination = NULL
WHERE   POD_POD_Id_Combination = @POD_Id
AND     OFF_Id = @OFF_Id
```

(Deleting the combination-copy rows outright would remove PODs that are legitimately visible on other, unrelated consolidated orders — nulling the now-dangling link is the safer choice, mirroring how `DEL_POD_Id_Lookup` is already nulled instead of blocking the delete.)

### Reproduction

1. Create two orders, consolidate order A into order B with `OCT_CopyPods` enabled on the order-consolidation type (`OrderConsolidationType.OCT_CopyPods`).
2. Confirm B now has a POD with `POD_POD_Id_Combination` = A's POD id.
3. Delete/replace the POD on order A and save.
4. Expect: SQL error 547 on `FK_POD_ProofOfDelivery_POD_ProofOfDelivery`, "Fel vid sparande av order" dialog.

## 3. Secondary finding — AB#23812 fix not backported (126/156 rows, many customers)

The much larger `FK_POD_ProofOfDelivery_DEA_DeliveryAttachment` bucket (126 rows across ~20 customers, spanning at least 2026-05 to 2026-08) is a different, **already-fixed** defect: `k2_ods_delete_attachment` didn't delete the `PDA_ProofOfDeliveryAttachment` row before deleting `DEA_DeliveryAttachment`, so a leftover PDA row blocked the delete via the FK.

- Fixed on **master** by commit `ac45328eb3` ("Deleting POD now also deletes attachment", 2026-08-05, AB#23812, version 2026.01.487).
- **Not merged** into `releases/2025.12` — a cherry-pick branch `features/23812_ErrorDeletingDeaPodConnectionDea_patch_2025.12` (tip `b7efc4f288`) exists but has not been merged.
- **No backport branch exists at all for `releases/2025.06`.** Verified directly: `k2_ods_delete_attachment` on both `origin/releases/2025.12` and `origin/releases/2025.06` is missing the `DELETE FROM PDA_ProofOfDeliveryAttachment` step that `master` has.

Recommend merging the existing `2025.12` patch branch and cutting an equivalent one for `2025.06` — this is actively causing the same error dialog for a long list of customers on both release branches, independent of Transmar's issue.

## 4. Minor — `FK_POD_ProofOfDelivery_ADR_Address` (2 rows, customer "Best", 2026-05-29)

Same failure signature as `support-a2bfi-pod-fk-office-mismatch.md` §5 (`POD_ADR_Id_Lookup` resolves to 0 because the delivery lookup fails), but here it's a client-side `DELETE`, not the mobile server `INSERT` path described there. Only 2 occurrences 2.5 months ago and none since — not investigated further; flagging for awareness only.
