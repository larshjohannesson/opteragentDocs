# Sandahls – order 1311093 prices on an 11× inflated pricing weight

| | |
|---|---|
| Customer | Sandahls Logistik AB |
| Version | 2025.06.447 (commit `940cb2ed51`; `releases/2025.06` tip is .538) |
| Order | 1311093, Logent JKPG Utrikes → ITAB Germany GmbH |
| Price list | "Logent Transport Utrikes fr 2026-06-01", service "Partigods", price calculation "[Standard]" |
| Analysed on | `releases/2025.06` |

## TL;DR

The order screen and the price engine run **two different weight calculations** on the same
order, and they disagree about whether the values on a kolli row are *per package* or
*row totals*.

* The order screen treats them as row totals → 1 010 kg, 1,30 flm, prissättningsvikt 2 410 kg.
* The price engine multiplies both by `Antal` (11) → 14,3 flm → 26 455 kg → **265 units à 100 kg**.

The trigger is `WeightCalculation.OverrideSettings()`, which copies
`WCA_IgnoreQuantityCalculatingPackages` ("Bortse från antal vid beräkning av kollin")
from the child weight calculation **unconditionally**, unlike every inheritable setting
around it. As soon as the price calculation has a *Viktberäkning* selected, the flag that
is set higher up in the hierarchy is silently switched off — but only for pricing.

Confirmed in the customer database: office, price list and customer all have the flag **on**
(WCA 1 and WCA 27), while price calculations **10542 / 10543 "Pris/100kg…"** point at
**WCA 11 "Övriga Europa"**, which has it **off**. See
[Confirmed against the customer database](#confirmed-against-the-customer-database).

**Note for support:** the 112 the customer calls "correct" is *also* wrong. It is
11 × 1 010 kg = 11 110 kg. With the order screen's own arithmetic the answer should be
`ceil(2410 / 100)` = **25 units**. Both figures in the ticket come from the same defect, so
fixing this will move prices on that price list a long way down — agree it with Sandahls
first.

## Reconstruction of the three observed states

Inputs: kolli row `Antal = 11`, `Vikt = 1 010,00`, `Flakmeter = 1,30`.
Constants: `WCA_LoadMeterFactor = 1850` (confirmed on WCA 27), unit = per 100 kg with
rounding **Up** (`PRR_PriceRoundingType.Up` → `Math.Ceiling`; 111,1 → 112 is only
reachable with ceiling, not with `Math.Round`).

| State | Total flm | Total weight | Loadmeter weight | Pricing weight used | Units | Matches ticket |
|---|---|---|---|---|---|---|
| Order screen (all three states) | 1,30 | 1 010 | 2 405 | 2 410 | *(would be 25)* | prissättningsvikt 2 410 ✔ |
| **1. As reported** | 1,30 × 11 = 14,30 | 1 010 × 11 = 11 110 | 26 455 | 26 455 | `ceil(264,55)` = **265** | ✔ 265,00 |
| **2. Flakmeter typed in Dimensioner** | 1,30 (manual replaces) | 11 110 | 2 405 | 11 110 | `ceil(111,10)` = **112** | ✔ 112,00 |
| **3. Flakmeter cleared on kolli row** | 0 | 11 110 | 0 | 11 110 | `ceil(111,10)` = **112** | ✔ same result |

State 2 works because manual order-level dimensions **replace** rather than max-merge the
package-derived ones — `ConsiderDimensions(dd, GetManualDimensions(del), replace: true)`
([WeightFactorCalculatorImplementation.cs:332](Order/CalculateWeights/WeightFactorCalculatorImplementation.cs:332)).
`DEL_Weight` is still empty, so the weight leg stays at the inflated 11 110 and simply wins
the `Max()`.

## Root cause

### 1. Two different weight calculations for the same order

The order screen computes `DEL_PricingWeight` with the hierarchy **office → price list →
service type → customer**:

```csharp
// WeightFactorCalculatorImplementation.cs:1303
var weightCalculation = GetWeightCalculation(del.OFF_Id, del.DEL_PLI_Id, del.DEL_PST_Id, del.DEL_CUS_Id, 0, 0);
```

The price engine ignores `DEL_PricingWeight` whenever the price calculation has its own
weight calculation, and recomputes from scratch with one extra level —
**+ price calculation (`PCC_WCA_Id`)**:

```csharp
// PriceUtilHelperImplementation.cs:163
case Tables.PRD_PriceDimension.PricingWeight:
    if (WCA_Id_ResourcePriceList != 0 || WCA_Id_PriceCollectionCalculation != 0)
    {
        var wca = _weightFactorCalculator.GetWeightCalculation(..., WCA_Id_PriceCollectionCalculation);
        ...
        quantity = _weightFactorCalculator.CalculatePricingWeight(w, del, wca);
    }
    else
    {
        quantity = del.DEL_PricingWeight;   // <- the value shown on screen
    }
```

`PCC_WCA_Id` is passed in from
[Generic.cs:1178-1184](Order/Price/PriceMode/Generic.cs:1178). This is why the customer sees
**"prissättningsvikten är fortsatt densamma"** while the price changes — the displayed
figure is never the one the price is built on.

### 2. The extra level silently resets the package-quantity flag

[`WeightCalculation.OverrideSettings()`](Common/CacheItems/SelectableItems/WeightCalculation.cs:708)
carefully inherits everything that is nullable:

```csharp
if (child.WCA_LoadMeterFactor.HasValue)          { clone.WCA_LoadMeterFactor = child.WCA_LoadMeterFactor; }
if (child.WCA_IndividualCalculationExtra1.HasValue) { clone.WCA_IndividualCalculationExtra1 = ...; }
```

…and then ends with two unconditional assignments:

```csharp
// Common/CacheItems/SelectableItems/WeightCalculation.cs:864-865
clone.WCA_PricingWeightPerPackage          = child.WCA_PricingWeightPerPackage;
clone.WCA_IgnoreQuantityCalculatingPackages = child.WCA_IgnoreQuantityCalculatingPackages;
```

Both are `bit NOT NULL DEFAULT 0` (`dbo.WCA_WeightCalculation.sql:34-35`) and are plain
two-state checkboxes in the editor
([WeightCalculationEditor.xaml:86](Client/Settings/WeightCalculation/WeightCalculationEditor.xaml:86)),
so a child weight calculation has **no way to express "inherit"**. Any WCA attached further
down the chain forces the flag to its own value — in practice `false`.

### 3. What the flag controls

```csharp
// WeightFactorCalculatorImplementation.cs:722
var quantity = weightCalculation.WCA_IgnoreQuantityCalculatingPackages ? 1 : Math.Max(packDim.Packages ?? 0, 0);
...
// :770-774
dd.LoadMeter += packDim.LoadMeter.GetValueOrDefault() * quantity;
dd.Weight    += packDim.Weight * quantity;
```

With the flag on, kolli-row values are row totals (Sandahls' setup — that is what the
Dimensioner box shows). With it off, they are per-package and get multiplied by `Antal`.
Same logic in `CalculatePricingWeightForPackage` ([:230](Order/CalculateWeights/WeightFactorCalculatorImplementation.cs:230)).

That the greyed Dimensioner values are `DEL_CalculatedWeight` / `DEL_CalculatedLoadMeter`
(1 010 and 1,30, not 11 110 and 14,30) is what proves the flag is **on** in the order-screen
chain — see [Dimensions.xaml:105](Client/Order/Order/WPF/Dimensions.xaml:105) and
[Dimensions.xaml:155](Client/Order/Order/WPF/Dimensions.xaml:155).

## Not fixed in a later build

No relevant changes to `Order/CalculateWeights/`, `Order/Price/` or
`Common/CacheItems/SelectableItems/WeightCalculation.cs` between `.447` and the `.538` tip,
and `origin/master` still has the identical unconditional assignment. Upgrading will not
help.

## Confirmed against the customer database

Run 2026-09-01 against `SB-SQL01\OPTER`, database `opter`. Everything matches the
reconstruction above.

**Weight-calculation hierarchy** — every level the order screen uses has the flag **on**:

| Level | WCA_Id | Name | IgnoreQuantity | PricingWeightPerPackage | LoadMeterFactor | RRS_Id |
|---|---|---|---|---|---|---|
| Office | 1 | Standard | **1** | 0 | 1950 | 1 |
| PriceList | 27 | Övriga Europa – Logent Utrikes | **1** | 0 | 1850 | NULL |
| Customer | 27 | Övriga Europa – Logent Utrikes | **1** | 0 | 1850 | NULL |

No ServiceType row — `PST_WCA_Id` is 0.

**Price calculations on the price list** — the extra level only pricing sees:

| PCC_Id | Name | PCC_WCA_Id | WCA | IgnoreQuantity | PricingWeightPerPackage |
|---|---|---|---|---|---|
| 10528 | Fraktrunt… | NULL | – | – | – |
| 10535 | Nod… | NULL | – | – | – |
| **10542** | **Pris/100kg avg Näs** | **11** | Övriga Europa | **0** | 0 |
| **10543** | **Pris/100kg …** | **11** | Övriga Europa | **0** | 0 |
| 10569 | Tidslossning – Logent | NULL | – | – | – |

The two `Pris/100kg` calculations point at **WCA 11 "Övriga Europa"**, which has the flag
off. That is the reset. (Names read off a low-resolution screenshot — the ids are reliable,
the exact spelling of the names is not.)

**Order 1311093:** `DEL_Weight` NULL, `DEL_CalculatedWeight` 1010, `DEL_LoadMeter` NULL,
`DEL_CalculatedLoadMeter` 1,30, `DEL_LoadMeterWeight` 2405, `DEL_PricingWeight` 2410.

**Package row:** `PAC_Quantity` 11, `PAC_Weight` 1010, `PAC_LoadMeter` 1,30, with
`PAC_LoadMeterManualChange` / `PAC_WeightManualChange` / `PAC_QuantityManualChange` all 1 —
so the entered values are used as-is.

Two things this settles beyond the main finding:

* `WCA_LoadMeterFactor` **1850** at price-list level correctly overrides the office's
  **1950** — the nullable settings inherit properly. Only the two non-nullable `bool` flags
  leak through. 1,30 × 1850 = 2405, then `WCA_RRS_Id` 1 rounds it to the 2410 on screen.
* WCA 11's own loadmeter factor must be NULL or 1850 — with the office's 1950 the units
  would have come out at 279, not 265.

### Queries used

(Order number 1311093 is `DEL_Id`.)

```sql
-- Which weight calculations apply to this order, and how each one sets the two flags
SELECT  lvl = 'Office',        w.WCA_Id, w.WCA_Name, w.WCA_IgnoreQuantityCalculatingPackages,
        w.WCA_PricingWeightPerPackage, w.WCA_LoadMeterFactor, w.WCA_RRS_Id
FROM    DEL_Delivery d
JOIN    OFF_Office o ON o.OFF_Id = d.OFF_Id
JOIN    WCA_WeightCalculation w ON w.WCA_Id = o.OFF_WCA_Id AND w.OFF_Id = d.OFF_Id
WHERE   d.DEL_Id = 1311093
UNION ALL
SELECT  'PriceList', w.WCA_Id, w.WCA_Name, w.WCA_IgnoreQuantityCalculatingPackages,
        w.WCA_PricingWeightPerPackage, w.WCA_LoadMeterFactor, w.WCA_RRS_Id
FROM    DEL_Delivery d
JOIN    PLI_PriceList p ON p.PLI_Id = d.DEL_PLI_Id AND p.OFF_Id = d.OFF_Id
JOIN    WCA_WeightCalculation w ON w.WCA_Id = p.PLI_WCA_Id AND w.OFF_Id = d.OFF_Id
WHERE   d.DEL_Id = 1311093
UNION ALL
SELECT  'ServiceType', w.WCA_Id, w.WCA_Name, w.WCA_IgnoreQuantityCalculatingPackages,
        w.WCA_PricingWeightPerPackage, w.WCA_LoadMeterFactor, w.WCA_RRS_Id
FROM    DEL_Delivery d
JOIN    PST_PriceServiceType s ON s.PST_Id = d.DEL_PST_Id AND s.OFF_Id = d.OFF_Id
JOIN    WCA_WeightCalculation w ON w.WCA_Id = s.PST_WCA_Id AND w.OFF_Id = d.OFF_Id
WHERE   d.DEL_Id = 1311093
UNION ALL
SELECT  'Customer', w.WCA_Id, w.WCA_Name, w.WCA_IgnoreQuantityCalculatingPackages,
        w.WCA_PricingWeightPerPackage, w.WCA_LoadMeterFactor, w.WCA_RRS_Id
FROM    DEL_Delivery d
JOIN    CUS_Customer c ON c.CUS_Id = d.DEL_CUS_Id AND c.OFF_Id = d.OFF_Id
JOIN    WCA_WeightCalculation w ON w.WCA_Id = c.CUS_WCA_Id AND w.OFF_Id = d.OFF_Id
WHERE   d.DEL_Id = 1311093;

-- The extra level that only pricing sees
SELECT  pcc.PCC_Id, pcc.PCC_Name, pcc.PCC_WCA_Id, w.WCA_Name,
        w.WCA_IgnoreQuantityCalculatingPackages, w.WCA_PricingWeightPerPackage
FROM    DEL_Delivery d
JOIN    PCC_PriceCollectionCalculation pcc ON pcc.PCC_PLI_Id = d.DEL_PLI_Id AND pcc.OFF_Id = d.OFF_Id
LEFT
JOIN    WCA_WeightCalculation w ON w.WCA_Id = pcc.PCC_WCA_Id AND w.OFF_Id = d.OFF_Id
WHERE   d.DEL_Id = 1311093;

-- The order's own numbers
SELECT  d.DEL_Weight, d.DEL_CalculatedWeight, d.DEL_LoadMeter, d.DEL_CalculatedLoadMeter,
        d.DEL_LoadMeterWeight, d.DEL_PricingWeight
FROM    DEL_Delivery d WHERE d.DEL_Id = 1311093;

SELECT  p.PAC_Quantity, p.PAC_Weight, p.PAC_LoadMeter,
        p.PAC_LoadMeterManualChange, p.PAC_WeightManualChange, p.PAC_QuantityManualChange
FROM    PAC_Package p WHERE p.PAC_DEL_Id = 1311093;
```

## Workaround (no code change)

All three options are settings-only and all three land on the same answer for this order:
pricing weight 2 410 → `ceil(24,10)` = **25 units**, not 112 and not 265.

1. **Repoint price calculations 10542 and 10543 to WCA 27** ("Övriga Europa – Logent
   Utrikes", the one the price list already uses) instead of WCA 11. *Recommended* — the
   most targeted change, and since it equals the price-list-level WCA the override becomes a
   no-op.
2. **Clear the *Viktberäkning*** on 10542/10543 (Prislista → Prisberäkning → *Viktberäkning*).
   With `PCC_WCA_Id = 0` the engine skips the recalculation entirely and uses
   `DEL_PricingWeight` — exactly the 2 410 shown on screen. Only do this if WCA 11 was not
   there to set something the price list doesn't already provide.
3. **Tick "Bortse från antal vid beräkning av kollin" on WCA 11** itself. Simplest, but the
   widest blast radius — check what else uses WCA 11 first:

   ```sql
   SELECT 'PCC' AS lvl, PCC_Id AS Id, PCC_Name AS Name FROM PCC_PriceCollectionCalculation WHERE PCC_WCA_Id = 11
   UNION ALL SELECT 'PriceList',   PLI_Id, PLI_Name FROM PLI_PriceList        WHERE PLI_WCA_Id = 11
   UNION ALL SELECT 'ServiceType', PST_Id, PST_Name FROM PST_PriceServiceType WHERE PST_WCA_Id = 11
   UNION ALL SELECT 'Customer',    CUS_Id, CUS_Name FROM CUS_Customer         WHERE CUS_WCA_Id = 11
   UNION ALL SELECT 'Office',      OFF_Id, OFF_Name FROM OFF_Office           WHERE OFF_WCA_Id = 11;
   ```

Do **not** clear the flakmeter on the kolli row — that leaves the weight leg inflated 11×
(the 112) and hides the problem behind a plausible-looking number.

Every order priced through 10542/10543 is affected the same way, so this is not a one-order
correction: expect a price recalculation across that price list, and prices will drop
sharply (this order goes 12 964 → whatever the 25-unit tier gives). Agree the change with
Sandahls before applying it.

## Proposed code fix

Make the two package flags inheritable like everything else in `OverrideSettings`. They are
`bool` today, so this needs the model, cache, DB column and editor checkbox to go nullable
(`bit NULL`, three-state / "inherit" in the UI), then:

```csharp
if (child.WCA_PricingWeightPerPackage.HasValue)
{
    clone.WCA_PricingWeightPerPackage = child.WCA_PricingWeightPerPackage;
}

if (child.WCA_IgnoreQuantityCalculatingPackages.HasValue)
{
    clone.WCA_IgnoreQuantityCalculatingPackages = child.WCA_IgnoreQuantityCalculatingPackages;
}
```

Migration must set existing rows to the value they have today so behaviour does not change
for customers who currently rely on the reset.

Worth deciding at the same time: the whole "Automatic dimensions" block at
[WeightCalculation.cs:838-862](Common/CacheItems/SelectableItems/WeightCalculation.cs:838)
is unconditional too (`WCA_CalculateVolume`, `WCA_CalculateLoadMeter`,
`WCA_LoadMeterPerSquareMeter`, the pallet settings…). That may be deliberate — they read as
one group the child either owns or doesn't — but it is the same class of trap, and
`WCA_LoadMeterPerSquareMeter` falling back to `0` (treated as `1` at
[:1006](Order/CalculateWeights/WeightFactorCalculatorImplementation.cs:1006)) can distort
automatically calculated flakmeter in the same silent way.

A cheaper, non-breaking complement: show the *actual* pricing weight used per price row in
the price details tooltip, so a mismatch against the Dimensioner figure is visible instead
of invisible.

## Open questions

* **Which figure does Sandahls actually want?** By the order screen's arithmetic it is 25
  units (2 410 kg / 100, rounded up). The customer expects 112. Before changing any setting,
  confirm with them what the agreed rate basis is — if their Logent price list was tuned
  while the ×11 inflation was in effect, correcting the weight will move every price on that
  price list.
* **`DEL_LoadMeterWeight` shows 2 405 in the first screenshot and 2 410 in the second**, from
  the same 1,30 flm. 2 405 = 1,30 × 1850 exactly; 2 410 is the value after the pricing-weight
  rounding rule set. Most likely the first screenshot shows a stored value and the second a
  freshly recalculated one, but this is unverified and worth a look if the customer reports
  drifting dimension weights. It does not affect the price defect.
* **Why is WCA 11 on those two price calculations at all?** Its loadmeter factor is NULL or
  1850, i.e. the same as what the price list already supplies, so it may be a leftover that
  can simply be cleared. Worth asking whoever set up the Logent price list.
* **How far does WCA 11 reach?** Other price calculations, price lists, service types or
  customers pointing at it hit the same trap — the query under workaround option 3 lists
  them. The same goes for any other WCA used at price-calculation level with the flag off
  while a parent has it on.
