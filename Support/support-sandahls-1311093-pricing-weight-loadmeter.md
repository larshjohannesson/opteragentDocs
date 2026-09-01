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

**Note for support:** the 112 the customer calls "correct" is *also* wrong. It is
11 × 1 010 kg = 11 110 kg. With the order screen's own arithmetic the answer should be
`ceil(2410 / 100)` = **25 units**. Both figures in the ticket come from the same defect.

## Reconstruction of the three observed states

Inputs: kolli row `Antal = 11`, `Vikt = 1 010,00`, `Flakmeter = 1,30`.
Derived constants: `WCA_LoadMeterFactor = 1850` (2405 / 1,30), unit = per 100 kg with
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

## Verify against the customer database

Confirms which WCA is attached where and what the two flags say. (Order number 1311093 is
assumed to be `DEL_Id`; adjust if their numbering differs.)

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

Expected if the analysis holds: one of the first four levels has
`WCA_IgnoreQuantityCalculatingPackages = 1`, and the `PCC_WCA_Id` weight calculation has it
`= 0`.

## Workaround (no code change)

Either of these makes pricing agree with the order screen, and both are settings-only:

1. **Tick "Bortse från antal vid beräkning av kollin"** on the weight calculation that the
   price calculation points to (Prislista → Prisberäkning → *Viktberäkning*). This is the
   safe one — it only affects price calculations using that WCA.
2. **Clear the *Viktberäkning* on the price calculation** if it exists only to set a factor
   that is already inherited. With `PCC_WCA_Id = 0` the engine falls back to
   `DEL_PricingWeight`, i.e. exactly the 2 410 shown on screen.

Do **not** recommend clearing the flakmeter on the kolli row — that leaves the weight leg
inflated 11× (the 112) and hides the problem behind a plausible-looking number.

Existing orders priced through this price calculation are affected the same way; a price
recalculation will be needed after the setting change.

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
* Whether Sandahls has other price calculations with a *Viktberäkning* set — the same trap
  applies to all of them.
