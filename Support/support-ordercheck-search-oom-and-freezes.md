# Incident: OrderCheck search — slow responses, UI freezes, and OutOfMemoryExceptions

**Status:** Under investigation — root cause not fully confirmed, contributing factors identified
**Scope:** Fleet-wide (43+ customers affected by OOM crashes in 30 days); UI freeze pattern confirmed in detail for one customer
**Date of write-up:** 2026-09-03
**Related:** [PR #13088 "Move OrderCheck search off the UI thread"](https://opter.ghe.com/code/Main/pull/13088) (open since 2026-06-17, unreviewed)

## Summary

Customers searching in Order Control (OrderCheck) intermittently experience either a frozen application (up to 21+ minutes observed) or a hard failure dialog ("Fel vid hämtande av data — Kontakt din systemadministratör"). Both symptoms trace back to the same server-side code path: `OrderCheck.GetAllSimpleResult` → `OrderCheck.LoadSearchResultModelsFromDataSet`. The failure dialog case has been confirmed to be a genuine `System.OutOfMemoryException` server-side, occurring for 43 different customers in the last 30 days. The freeze case has been confirmed via DeadlockWatchdog telemetry to be a real multi-minute block on the UI thread, not a false positive.

Investigation found a long-standing structural weakness in how OrderCheck materializes search results, a client-side design flaw that turns any slow server response into a full application freeze, and a correlation with the .NET 9 → .NET 10 runtime upgrade that shipped between the `2025.06` and `2025.12` release branches. No single root cause has been conclusively isolated; several contributing factors have been identified and are ranked below.

## Customer impact observed

| Customer | Symptom | Evidence |
|---|---|---|
| Markis City Service AB (`markiscityse`) | UI freezes, no crashes | 263 DeadlockWatchdog freeze episodes in 30 days; several 200-500+ seconds; one 1305-second (21.75 min) outlier on 2026-09-01; affects at least 8 different workstations |
| itide.no | Hard crash, "Fel vid hämtande av data" | Freshdesk ticket #149308, 2026-08-31; log shows `System.OutOfMemoryException` in `OrderCheck.LoadSearchResultModelsFromDataSet` |
| 41 other customers | Same OOM exception | Confirmed via Azure Log Analytics (`Logs_CL`), same exact exception/stack, 100+ occurrences in 30 days |

Notably, Markis City Service — the customer with by far the most severe and frequent *freezes* — has **zero** OOM exceptions in the same 30-day window. Freezing and crashing are related symptoms of the same code path but are not the same event, and don't always co-occur for the same customer.

## Evidence: OOM trend and version migration (90 days, weekly)

**Methodology note**: customer counts below are scoped to `Assembly == "Opter.Main.ServerApi"` only, with one version picked per customer per week (`arg_max` by time). An earlier pass counted a customer as "active on version X" if *any* of their services (across `Client`, `ServerApi`, `Web.App`, `MessageServer`, unrelated components like `FAQ.Routing`, etc.) reported that version prefix — this let a single customer be counted into multiple, non-mutually-exclusive branch buckets in the same week, and pulled in an unrelated component (`Opter.FAQ.Routing`, static version `1.0.0.0`, unrelated to release branches) that inflated an "other" bucket with no real meaning. That first pass is not used below; the charts and numbers here use the corrected, single-assembly methodology.

![Weekly OrderCheck OutOfMemoryException count, split by ServerApi version](images/ordercheck-oom-weekly-by-version.png)

![Active ServerApi customers by version, unstacked — each line is its own value, not a cumulative sum](images/ordercheck-serverapi-customers-by-version.png)

![OOM incidence rate per 100 active customers per week, by version](images/ordercheck-oom-incidence-rate.png)

What these show:
- **The `2025.06` → `2025.12` migration was sharp, not gradual**: `2025.06`'s active customer count holds steady around 270-280 for the first 11 weeks, then collapses to 187 → 1 → 0 across the final three weeks (2026-08-17 through 2026-08-31), while `2025.12` correspondingly jumps from ~290-295 to 380 → 564 → 562 in the same window. Essentially the whole remaining `2025.06` fleet moved in a two-week span.
- **Total ServerApi-active customer count is flat (~575-597) across the entire 90 days** — there is no visible dip in *customer presence* through late June/July. An earlier informal read of a (since-discarded, methodologically flawed) chart suggested a summer-vacation dip in this data; that claim is retracted as originally stated (customer *presence* doesn't dip) but the underlying intuition is confirmed by a better metric below.

![Logins per active customer per week, showing a dip during the Swedish summer vacation period](images/ordercheck-usage-volume-logins.png)

- **Usage *intensity* does dip for the Swedish vacation period, even though customer *presence* doesn't.** Raw `Logs_CL` row count turned out to be a poor proxy (dominated by background/infrastructure noise — it's actually *higher*, not lower, across the presumed vacation weeks, which is itself a sign it isn't measuring real usage). Login events ("User logged in to Opter client") are a cleaner signal: logins per active customer per week peaks at 24.5 in late June, drops to 17.2-18.5 across mid-to-late July (weeks of Jul 13, 20, 27 — roughly the concentrated Swedish "semester" period, weeks 28-31), then recovers to the low-20s by August. That's a genuine ~25-30% dip in usage intensity, distinct from and not visible in the customer-*presence* data. The first data point (Jun 1) is excluded as an artifact of the query's 90-day boundary (partial week).

![Weekly login volume vs. total OOM exceptions, showing the vacation dip and the migration window as two distinct effects](images/ordercheck-login-vs-oom-crosscheck.png)

- **The vacation dip and the migration wave are two separate, additive effects on the OOM count, not one confounding the other.** Total weekly OOM count (summed across all versions) correlates with login volume at **r = 0.63** during the stable pre-migration window (2026-06-08 through 2026-08-10) — the OOM curve visibly tracks the login curve's shape, bottoming out in the same week (Jul 20, the lowest point in both series). That correlation weakens to r = 0.49 once the migration weeks are folded in, because the migration window (Aug 17 onward) decouples the two: login volume stays roughly flat (~22/week) while OOM count jumps from 22 to 41, driven by branch composition rather than usage. Practically: the vacation dip modestly suppressed incident counts in July; the migration then dominates and drives counts well above anything usage volume alone would explain.
- **A third cohort exists**: a stable ~23-32 customers throughout the period run a separate `20260101.*` build family, distinct from both `2025.06` and `2025.12`. This is a real, minor branch, not noise.
- **The incidence-rate gap is not just a 90-day aggregate artifact**: normalizing OOM count by active customers per branch per week, `2025.12` runs at roughly 2.4-7.9 exceptions per 100 customers most weeks, versus `2025.06` at roughly 0.2-5.5 (weeks with fewer than 10 remaining `2025.06` customers are omitted from that line as statistically unreliable — this excludes the final two weeks, where the migration itself was still completing mid-week and produces a weekly-binning artifact: a customer who migrated mid-week is attributed wholly to `2025.12` for that week's customer count, while any `2025.06`-version exceptions they threw earlier that same week still show up in the `2025.06` numerator). The rate gap holds consistently from the very first week of the observation window, before the migration wave even accelerated — i.e. it isn't an effect of the migration itself, it predates it.

## Facts (directly observed/confirmed)

1. Multiple customers have reported slow OrderCheck searches and hard error dialogs.
2. At least one confirmed case (itide.no, ticket #149308, 2026-08-31) traced the error dialog directly to a server-side `System.OutOfMemoryException` thrown in `Opter.Main.Server.Orders.OrderCheck.OrderCheck.LoadSearchResultModelsFromDataSet`, called from `GetAllSimpleResult`.
3. The identical exception, in the identical code path, has occurred for **43 distinct customers, 100+ times in the last 30 days** — this is a fleet-wide issue, not isolated to one customer.
4. DeadlockWatchdog logs confirm genuine multi-minute UI-thread freezes for Markis City Service: 263 episodes over 30 days, several 200-500+ seconds, one 21-minute outlier, all tied specifically to `OrderCheckViewModel.get_SearchCommand` → `GetAllSimpleResult`.
5. The client makes this call **synchronously on the UI thread**: `SearchCommand` is a plain (non-async) `RelayCommand` whose handler calls `CallerBase.RestCall`, which blocks via `.GetAwaiter().GetResult()`. Confirmed by direct code inspection ([OrderCheckViewModel.cs](../../Main/Order/ViewModels/Order/OrderCheck/WPF/OrderCheckViewModel.cs), [CallerBase.cs](../../Main/ServerProxy/CallerBase.cs)), not inferred.
6. Freezes cluster across multiple users/machines within the same time windows (observed in Markis City Service's data — e.g. MC040 and ANDREAS-LAPTOP both froze for 295-499s within minutes of each other on 2026-09-02).
7. **Not all freezes correlate with OOM crashes.** Markis City Service had 263 freeze episodes but zero OOM exceptions in the same period.
8. `releases/2025.06` targets **.NET 9.0**; `releases/2025.12` (and `master`) target **.NET 10.0** — confirmed via `.csproj` (`ServerApi/Opter.Main.ServerApi.csproj`).
9. Customers on `2025.12` show a higher OOM incidence than customers on `2025.06`, both in the 90-day aggregate (11.5% of that branch's customers vs. 7.7%) and, more granularly, in a corrected week-by-week incidence rate (~2.4-7.9 vs. ~0.2-5.5 OOM exceptions per 100 active customers per week — see "Evidence: OOM trend and version migration" above). `2025.12` also shows higher peak ServerApi pod memory (91.9% max vs 54.4% max of the pod's 1000 MB container limit, in the settled 6-24h-uptime window). The weekly rate gap is present from the first week of the 90-day window, before the `2025.06`→`2025.12` migration wave accelerated — it is not an artifact of the migration itself.
10. No explicit GC tuning exists anywhere in the stack — not in any `.csproj`, not in `ServerApi/Dockerfile`, not in the shared base image (`VersionBuilder/BaseImage/Dockerfile`, which is just `mcr.microsoft.com/dotnet/aspnet:10.0` + fonts). Both .NET 9 and .NET 10 builds run on stock GC defaults (`System.GC.Server: true`, the SDK default for ASP.NET Core apps).
11. **No substantive functional diff exists between `2025.06` and `2025.12`** for `OrderCheck.cs`, `ModelBase.cs` (incl. `Clone<T>()`), `OrderCheckSearchResult.cs`, or `k2_orc_get_all_by_all.sql`, beyond CSharpier reformatting and two trivial (2-10 line) additions (an access-attribute annotation, and loading one extra string field `SHI_SubcontractorResource`). This is a direct branch-to-branch comparison (`git diff`/`git log releases/2025.06..releases/2025.12`), which lists every commit present in `2025.12` but not `2025.06` regardless of how old it is — it is **not** time-boxed, and already surfaced commits from November 2025 and February 2026. So this rules out a code-level explanation across the full lifetime of both branches, not just recent history. (An earlier, separate "any changes in the last 3 months" check was run first and is superseded by this — that check was time-boxed and would have missed older changes on its own; it is not independent evidence and shouldn't be read as such.)
12. For Markis City Service's shared SQL elastic pool (`opterproductionsweden1c`), CPU %, Data IO %, and Workers % all sit comfortably within headroom over the last 7 days (CPU peaks ~40-50%, Data IO ~20-25%, Workers near 0%). Aggregate resource exhaustion is ruled out for that pool, on those three metrics.

## Strong inferences (well-supported, not directly measured)

- **The server was genuinely slow to respond during freezes, not the client hanging on its own.** The UI thread was observed blocked *inside* the outbound REST call for up to 21 minutes (via DeadlockWatchdog stack traces showing the wait inside `CallerBase.RestCall`/`Task.InternalWait`). Nothing else explains a call staying outstanding that long.
- **`LoadSearchResultModelsFromDataSet`'s reflection-based `Clone<T>()` fan-out is the mechanical cause of the OOM crashes**, and plausibly a contributor to the intermittent slowness too. Mechanism, from code inspection ([OrderCheck.cs:221-306](../../Main/Server/Orders/OrderCheck/OrderCheck.cs)):
  - `GetAllSimpleResult` loads the **entire** multi-table result into an in-memory `DataSet` via `_databaseAccess.ExecuteDataSet(...)` — no streaming (`SqlDataReader`), no row cap, no `TOP`/pagination anywhere in the SQL layer either (`k2_orc_get_all` → `k2_orc_get_all_by_all` → the 2816-line `k2_search_deliveries_common_output`, ~200+ output columns).
  - `LoadSearchResultModelsFromDataSet` materializes one `OrderCheckSearchResult` object per row — a wide DTO with ~350 properties.
  - For Consignment/Shipment/PriceItemCost search modes, every shipment/price-item beyond the first calls `ModelBase.Clone<T>()` ([ModelBase.cs:76-100](../../Main/Model/ModelBase.cs)), which reflects over **all ~350 properties** to produce a full duplicate object — not a lean child row. A delivery with 10 shipments costs 11 full-size objects instead of 1 large + 10 small. This is an O(rows × fanout) memory (and CPU) multiplier with no ceiling.
  - No safeguard exists anywhere: no max-row check, no warning, no TODO acknowledging the risk.

## Open questions / not yet ruled out

- **Database vs. pod as the source of slow response — unresolved.** We've ruled out *aggregate* SQL resource exhaustion for one customer's elastic pool on three metrics (CPU/IO/Workers), but:
  - Lock/blocking contention is invisible to those metrics and has not been checked (would need Query Store or a live blocking-chain capture during an incident).
  - No other customer's database has been examined.
  - No actual execution plan for `k2_orc_get_all_by_all` has been pulled.
- **The .NET 9→10 GC behavior change is a hypothesis, not a confirmed mechanism.** We have a real, consistent correlation (higher peak memory + higher OOM rate on the .NET 10 branch, with no other explanatory code change found) but have not verified against .NET's actual GC release notes what, if anything, changed in Server GC's container-memory-limit heuristics between .NET 9 and .NET 10.
- **Sample size caveat, updated**: with the corrected weekly counts, the `2025.06` (.NET 9) group ranges from ~270-280 customers down to single digits as the migration completes, vs. ~290-570 on `2025.12` (.NET 10). The rate gap is visible in most individual weeks, not just the 90-day aggregate, which strengthens confidence — but the final two weeks of `2025.06` data are excluded from the rate comparison (fewer than 10 remaining customers, and a weekly-binning artifact around the mid-week migration cutover — see "Evidence" section above).
- Whether the OOM-triggering searches and the freeze-triggering searches share a common size/selectivity threshold, or represent different severities of the same problem, has not been established.
- **Migration velocity from `2025.06` to `2025.12` has now been measured directly** (see "Evidence" section above) — it was not a slow drift but a sharp two-week transition (2026-08-17 to 2026-08-31). This confirms the mechanism (more customers moving onto the higher-incidence branch increases fleet-wide absolute counts) but the *rate* gap itself predates and is independent of the migration timing.
- ~~Not yet built: a usage-volume view~~ — built (see "Evidence" section above, login-based chart). Confirms a genuine ~25-30% dip in usage intensity during the Swedish vacation period, distinct from customer presence (which stays flat).
- ~~Cross-check the vacation dip against the OOM weekly series~~ — done, see "Evidence" section above. **Two distinct, additive effects on the same metric, confirmed quantitatively**: usage volume (login rate) correlates with total weekly OOM count at **r = 0.63** during the stable pre-migration window (2026-06-08 through 2026-08-10), dropping to r = 0.49 once the migration weeks are included — because the migration window briefly *decouples* the two (logins stay roughly flat at ~22/week while OOM count jumps from 22 to 41, driven by branch composition, not usage). Both the vacation effect and the migration effect are real; they just dominate on different weeks.

## Hypotheses (plausible, unconfirmed)

- .NET 10's Server GC commits more memory relative to the container limit than .NET 9 did, for the same workload, as a performance/throughput tradeoff — the leading candidate explanation for the fleet-wide ServerApi memory-plateau shift (every `serverapi` pod, across ~40 sampled customers, climbs from ~150-230 MB at cold start to 35-90% of its 1000 MB limit within 6-24 hours and stays there, with no restarts/crash-loops — i.e. not a leak in the classic sense, a sustained plateau).
- The `2025.06` → `2025.12` customer migration (now confirmed as a sharp two-week transition around 2026-08-17 to 2026-08-31, see "Evidence" section above) is why fleet-wide OOM/slowness incident *counts* have trended upward (14-30/week in June-July → 31-41/week by mid-to-late August) — this mechanism is now directly supported by data, though the underlying *rate* difference between branches predates and is independent of the migration itself.
- Slow SQL response (query-plan degradation on broad/unselective searches, from `k2_orc_get_all_by_all`'s forced `INNER LOOP JOIN` hints across `DEL_Delivery`/`SHI_Shipment`/`ADR_Address`/`RCU_ResourceCreditingUnit`) is a contributing factor to freezes independent of the memory story — plausible from the SP's structure, unconfirmed without an execution plan.

## Timeline of relevant code changes

- **2026-01-21** — `f52905ab7c`, "Load SHI_SubcontractorResource on search" (#12064): +2 lines to `OrderCheckSearchResult.cs`, loads one additional string field. Negligible impact.
- **2026-02-04** — `6523365d63`, "Server access attributes" (#12144): 334 files, adds `[MethodAccessAttribute(...)]`-based access-control checks across nearly every server method, including the OrderCheck path (10-line change to `OrderCheck.cs`). Largest behavioral diff found between `2025.06` and `2025.12`; plausible source of added per-call overhead, though not confirmed to affect memory specifically.
- **2026-06-17** — [PR #13088](https://opter.ghe.com/code/Main/pull/13088) opened: "Move OrderCheck search off the UI thread" — wraps `GetAllSimpleResult` in `Task.Run`, makes `SearchCommand` async, adds a busy-indicator overlay. CI green, Copilot review comments addressed, **still awaiting human review/merge** as of this write-up (2.5+ months open).
- **2026-06-23** — `a651ce02df` / `2ebbe93594` / `b3cfc63f45`, "Include schedule search data for all schedule columns" (#13117): one-line change to `k2_search_deliveries_common_output.sql`, broadens which customers' searches include an extra block of "schedule" columns (6 more `OCO_OCC_Id` values trigger the branch). Widens result-set columns for affected customers; contributes to per-row memory footprint but does not by itself explain the fleet-wide pattern.
- **No changes found** to `OrderCheck.cs`, `ModelBase.cs`, `OrderCheckSearchResult.cs`, or `k2_orc_get_all_by_all.sql` in the 3 months preceding this investigation (2026-06 through 2026-09), beyond the items above and pure CSharpier reformatting.

## Recommended next steps, roughly in order of effort

1. **Get [PR #13088](https://opter.ghe.com/code/Main/pull/13088) reviewed and merged.** It doesn't fix the underlying slowness or crash, but it converts "frozen, unresponsive application for up to 21 minutes" into "visible busy spinner while a slow operation completes" — a major UX improvement for the exact same underlying condition, ready to ship today.
2. **Add a row-count/size guard in `GetAllSimpleResult`** (e.g. `COUNT(*)`/`TOP N+1` pre-check) to fail fast with a friendly "narrow your search" message instead of materializing and crashing. Stops the hard-crash case; does not address the more common intermittent slowness.
3. **Replace `Clone<T>()`'s reflection fan-out in `LoadSearchResultModelsFromDataSet`** with a lean child-row DTO for the shipment/price-item fan-out, instead of duplicating the full 350-property object per child row. Addresses both the OOM crashes and the everyday intermittent slowness at the mechanical root; the biggest-impact code fix identified.
4. **Test an explicit GC heap hard limit** (`System.GC.HeapHardLimitPercent` via `runtimeconfig.json`, or `DOTNET_GCHeapHardLimitPercent` env var) on `ServerApi` pods, to force more conservative memory behavior under container constraints. Cheap, code-free, fully reversible; worth testing on a small customer cohort before broader rollout.
5. **Confirm or rule out SQL-side contribution**: get blocking-chain/Query Store access for one affected customer during a live incident, and pull an actual execution plan for `k2_orc_get_all_by_all` under a broad/unselective search.
6. **Verify the .NET 9→10 GC hypothesis** against Microsoft's actual GC release notes/changelog for those versions, to confirm or rule out the specific mechanism before relying on it as a explanation in any customer-facing communication.
7. ~~Build a usage-volume view~~ — done (see "Evidence" section above). Remaining: cross-check the login-volume dip against the OOM weekly series to see whether reduced vacation-period usage measurably suppressed incident counts in those specific weeks.

## Supporting evidence / data sources used

- Azure Log Analytics workspace `opter` (`Logs_CL`, `EdiLogs_CL`, `PodResourceUsage_CL`) — DeadlockWatchdog telemetry, OOM exception logs, ServerApi pod memory/CPU trends, fleet-wide version distribution.
- Freshdesk ticket #149308 (itide.no), 2026-08-31.
- Azure SQL Elastic Pool metrics (AdminTool Cloud Status page) for `opterproductionsweden1c`, last 7 days: CPU %, Data IO %, Workers %.
- Git history: `releases/2025.06`, `releases/2025.12`, `master` — diffs and commit logs for `OrderCheck.cs`, `ModelBase.cs`, `OrderCheckSearchResult.cs`, relevant stored procedures, `.csproj` files, Dockerfiles.
- [PR #13088](https://opter.ghe.com/code/Main/pull/13088) and its review comments.
- Weekly OOM-count, active-customer-count, and incidence-rate charts (`images/ordercheck-oom-weekly-by-version.png`, `images/ordercheck-serverapi-customers-by-version.png`, `images/ordercheck-oom-incidence-rate.png`), generated from `Logs_CL` scoped to `Assembly == "Opter.Main.ServerApi"`, 2026-06-01 through 2026-09-03. Raw weekly series available on request — not checked into this doc to keep it readable.
- Usage-volume chart (`images/ordercheck-usage-volume-logins.png`), from `Logs_CL` messages matching "User logged in to Opter client", same 90-day window.
- Login-vs-OOM cross-check chart (`images/ordercheck-login-vs-oom-crosscheck.png`) and correlation calculation, same underlying weekly series.
