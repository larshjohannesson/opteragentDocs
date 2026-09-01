# Hypothesis: bad cached execution plans in the invoice-creation SPs

**Customer:** BudWheels, CustomerID 208, `budwheelsse` on `opterproductionsweden1.database.windows.net` (Azure SQL Database)
**Version:** 2025.12.256 (`b2ee9c7831`)
**Incident window:** 2026-08-11 12:08–12:22 UTC — two `k2_inv2_create_invoice` executions for `CUS_Id = 5539` exceeding a 300 s command timeout

Companion document: `support-budwheels-create-invoice-period-timeout.md` (full case analysis, the `[Alla]` mechanism, and the separate — and confirmed — stale-data leak).

---

## 1. The hypothesis in one paragraph

The row-creation SPs are "catch-all queries": each carries 5–6 predicates of the form `OR @param`, so a **single cached plan must serve every parameter combination**. The plan is built from the cardinality estimates of whichever parameter values happened to compile it. Because those parameters (`@INT_Id`, `@CPR_Id`, `@DEL_Id`, `@REG_Id`) are all derived from **customer configuration**, a plan compiled for one customer's shape can be catastrophically wrong for another's — which would explain why invoice creation is lightning fast for most `CUS_Id` and times out for a few, reproducibly, then behaves differently after a plan eviction.

**Why this fits better than a volume explanation:** CUS 5539 has only **9** pending deliveries. Under this hypothesis duration is driven by estimate error and tempdb spills, not by row count — so a tiny customer taking 300 s is expected rather than paradoxical.

> ## 🛑 CLOSED 2026-08-13 — hypothesis falsified, root cause found elsewhere
>
> The case was reproduced on a local restore and root-caused: **`k2_inv2_create_rows_price_item` accounts for 98.6 % of a 339 s run**, driven by 200 313 leaked `PIT_DIS_Id = 2` rows inflating its office-wide driving set. Retiring those rows drops the whole call from 339 491 ms to 22 154 ms.
>
> **See `support-budwheels-create-invoice-period-timeout.md` → "RESOLVED 2026-08-13" for the full result.** Everything below is retained as the record of how the plan hypothesis was tested and eliminated.
>
> Specifically falsified here: parameter sniffing (recompile had no effect), stale statistics (all FULLSCAN-fresh, `modification_counter = 0`), and lock waiting (`blocking_session_id = 0`, only `CXCONSUMER`). What *did* survive from §2 is the **forced `INNER LOOP JOIN`** — the optimizer had accurate cardinality and still could not choose a better shape, which is why the plan stays bad no matter how often it is recompiled.
>
> Also note §11a's 383 ms measurement was **wrong** — my reconstruction used `@FromDate = '2026-07-01'` where the real period has `INP_InvoiceFromDate = NULL`, dropped the `INTO #inr_tmp`, and reused a settled `@INH_Id`. The real statement costs 2 280× more CPU. A reconstructed query is not the query.

> ## ⚠️ TESTED 2026-08-11 — RESULT: NEGATIVE
>
> `sp_recompile` was run on the invoice-creation SPs and `[Alla]` for CUS 5539 was **still slow**.
>
> **The stale-cached-plan variant of this hypothesis is dead.** After a recompile the plan is built fresh from CUS 5539's own parameters (`@INT_Id=2, @CPR_Id=NULL, @DEL_Id=NULL, @REG_Id=NULL`), so parameter sniffing across customer shapes — the mechanism §2 describes — cannot be what is happening.
>
> **Three things recompiling does not fix, and which therefore survive:**
>
> 1. **The forced join hints** (§2, amplifier 1). A fresh compile with perfect estimates still cannot choose a hash join when the SP says `INNER LOOP JOIN`. The plan shape is dictated by the source, not the optimizer. **This is now the strongest plan-related suspect.**
> 2. **Bad statistics.** Recompiling uses *current* statistics. If the filtered-index stats on `IX_PIT_DIS_Id_Filt` are stale, a fresh compile produces the same bad plan. Recompile fixes sniffing; it does not fix wrong numbers.
> 3. **Query Store plan forcing.** If Azure automatic tuning has forced a plan (`is_forced_plan = 1`), the forced plan is re-imposed on recompilation and `sp_recompile` is simply defeated. Automatic tuning is on by default on Azure SQL DB. Checking needs the §3 grant.
>
> Also still fully open, and untouched by any of this: **lock waiting** on `SYS_System` (companion document §4c).
>
> **Caveat on the test itself:** my recompile list covered the six `_rows_*` SPs plus `k2_inv2_create_invoice`. The chain contains ~20 more procedures (`_create_header`, `_get_next_invoice_number`, `_copy_prices_for_invoice`, `_complete_invoice*`, `_create_interests_and_reminder_fees`, `_filter_construction_items`, …). If the slow statement lives in one of those, the test did not touch it. §11 has a complete version.
>
> **Read §11 next** — it has two further tests that need no new permissions.

**Status: largely falsified.** Sections 3–10 remain the runbook that got here and are still the right way to gather plan evidence once §3's grant arrives. §11 is where the live work now is.

---

## 2. Why these SPs are unusually exposed

| SP | `OR @param` predicates | forced hints |
|---|---|---|
| `k2_inv2_create_rows_delivery` | 5 | 3 |
| `k2_inv2_create_rows_price_item` | 6 | 1 |
| `k2_inv2_create_rows_add_service` | 5 | 2 |
| `k2_inv2_create_rows_direct_expense` | 5 | 2 |
| `k2_inv2_create_rows_expense` | 5 | 0 |
| `k2_inv2_create_rows_correction` | 1 | 2 |

Across all **3 750** stored procedures in the database, exactly **2** use `OPTION (RECOMPILE)` or `OPTIMIZE FOR`. There is effectively no plan-stability defence anywhere in the codebase.

**The predicate to focus on.** In `_rows_delivery` and `_rows_price_item`:

```sql
AND (DEL_Id = @DEL_Id OR @INT_Id <> 3)
```

* Compiled with `@INT_Id = 3, @DEL_Id = 12345` — an **Order**-invoiced customer — the optimizer sees an equality on the clustered key and estimates **~1 row** from `DEL_Delivery`. It sizes the whole plan for one row: nested loops, minimal memory grant, no parallelism.
* Reused with `@INT_Id = 1, @DEL_Id = NULL` — a **Collection** customer — the predicate becomes `OR TRUE`, true cardinality is orders of magnitude higher, and the plan does not change.

The same applies to `(ISNULL(DEL_CPR_Id,0) = ISNULL(@CPR_Id,0) OR @INT_Id <> 2)` across Project vs non-Project customers, and `(COALESCE(DEL_REG_Id,-1) = @REG_Id OR COALESCE(@REG_Id,0) = 0)` across region-scoped vs not.

**Two amplifiers:**

1. **The forced hints remove the optimizer's safety net.** A large underestimate is normally survivable — a hash join degrades gracefully. But `INNER LOOP JOIN`, `FORCESEEK` and `INDEX=` are hard directives: under a forced loop join a bad estimate becomes N nested iterations with no escape, and an undersized memory grant becomes a **tempdb spill** on every sort and hash. The hint comments say they exist *"to avoid deadlock"*; the cost is that these queries cannot recover from a bad estimate.
2. **Filtered-index statistics go stale silently.** `IX_PIT_DIS_Id_Filt` is `WHERE PIT_DIS_Id = 2`. Statistics auto-update triggers on **whole-table** modifications, not the filtered subset, so filtered stats on a small subset of a large table are notoriously stale. Wrong estimate in → wrong plan out.

---

## 3. Step 0 — Permissions (do this first)

Query Store catalog views and the performance DMVs are gated behind a permission that a plain reader account does not have:

```
Msg 262, Level 14, State 1
VIEW DATABASE PERFORMANCE STATE permission denied in database 'budwheelsse'.
```

Azure SQL Database (and SQL Server 2022+) split the old `VIEW DATABASE STATE` into `VIEW DATABASE PERFORMANCE STATE` and `VIEW DATABASE SECURITY STATE`. Query Store — `sys.query_store_*`, `sys.database_query_store_options` — and the `sys.dm_exec_*` DMVs all now require the **performance** one. A login with `db_datareader` can read the customer's tables (which is why the `PIT` / `DEL` counts worked) but cannot read Query Store.

**Essentially every step in this document needs it**, so resolve this before continuing.

### First, see what you actually have

```sql
SELECT ORIGINAL_LOGIN() AS login_name, USER_NAME() AS db_user;

SELECT IS_ROLEMEMBER('db_owner')      AS is_db_owner,
       IS_ROLEMEMBER('db_ddladmin')   AS is_ddladmin,
       IS_ROLEMEMBER('db_datareader') AS is_datareader;

SELECT permission_name
FROM   fn_my_permissions(NULL, 'DATABASE')
ORDER  BY permission_name;
```

If `is_db_owner = 1` you can grant it to yourself and proceed immediately.

### Measured 2026-08-11 for `lars.johannesson@opter.com` on `budwheelsse`

`is_db_owner = 0`, **`is_ddladmin = 1`**, `is_datareader = 1`. 59 database permissions, all the `ALTER ANY …` / `CREATE …` set plus `SELECT`, `INSERT`, `UPDATE`, `DELETE`, `EXECUTE`, `REFERENCES`.

| needed for | permission | held? |
|---|---|---|
| Query Store (§4–§8) | `VIEW DATABASE PERFORMANCE STATE` | ❌ |
| any execution plan in SSMS, estimated or actual | `SHOWPLAN` | ❌ |
| `sp_query_store_force_plan` (§10) | `ALTER` on the database (bare) | ❌ |
| **`sp_recompile` (§10)** | `ALTER` on the procedures | ✅ **via `db_ddladmin` → `ALTER ANY SCHEMA`** |
| running the SP, reading tables | `EXECUTE` / `SELECT` | ✅ |

**Two things follow.**

**1. The recompile test in §10 is available right now.** It is the single highest-value action possible with the current permission set, it needs nothing new, and — per the correction at the top of §6 — it does not destroy the Query Store evidence. If you want to move today, that is the thing to do.

**2. `SHOWPLAN` is missing, which is a bigger blocker than the Query Store gate.** Without it you cannot display *any* execution plan in SSMS — not the actual plan (Ctrl+M), not even the estimated one (Ctrl+L) — on this database or a restored copy accessed with this login. Any plan-level analysis needs this grant regardless of route.

### The grant request should be an easy sell

Worth stating plainly when asking, because it inverts the usual objection: **you already hold `db_ddladmin`**, which lets you alter schema and stored procedures in the customer's production database. `VIEW DATABASE PERFORMANCE STATE` and `SHOWPLAN` are read-only and expose nothing beyond query text and plan shapes. They are **strictly less privileged than what you already have** — you can already change the database; you just cannot see why it is slow.

Note you cannot self-grant: `db_ddladmin` carries no `GRANT` authority. It has to come from a `db_owner`, `db_securityadmin`, or the Azure SQL server admin.

### Option A — get the grant (preferred; read-only, 30 seconds)

Run **in `budwheelsse`**, by a `db_owner` of that database or the Azure SQL server admin:

```sql
GRANT VIEW DATABASE PERFORMANCE STATE TO [<your-login-or-user>];
```

For the full runbook you also want:

```sql
GRANT SHOWPLAN TO [<your-login-or-user>];   -- graphical / actual execution plans
```

`VIEW DATABASE PERFORMANCE STATE` and `SHOWPLAN` are **read-only diagnostic** permissions — they expose no customer data beyond what a reader already sees, and grant no ability to modify anything. That is usually an easy approval to get.

Two later steps need more than read access, and are worth requesting separately rather than bundling:

| step | needs |
|---|---|
| `sp_recompile` (§10) | `ALTER` on the procedures, or `db_ddladmin` |
| `sp_query_store_force_plan` (§10) | `ALTER` on the database |

### Option B — server-wide, if support will need this repeatedly

This is a multi-tenant server with many customer databases, and support will hit this same wall on the next case. The Azure SQL server admin can grant it once at server level instead of per database — run in the **`master`** database of `opterproductionsweden1`:

```sql
ALTER SERVER ROLE ##MS_ServerPerformanceStateReader## ADD MEMBER [<your-login>];
```

That covers every database on the logical server. Worth proposing as a standing arrangement for the support team — the alternative is a permission request per customer per incident.

### Option C — restore a copy and own it (best technical fallback)

If the grant is slow or refused, do a **point-in-time restore** of `budwheelsse` to a new database, restoring to just after the incident (say `2026-08-11T12:25:00Z`). You are the owner of the restored database, so all permission problems disappear — and this is strictly better than Option A for two reasons:

* Query Store contents are restored with the database, so the captured plans from the incident come with it.
* You can **actually run** `k2_inv2_create_invoice` for CUS 5539 and capture a real actual-execution plan with true row counts and spill warnings — which is the one thing Query Store alone cannot give you (§6c). That is not safe on production, because it writes rows.

Restore via the Azure Portal (SQL database → Restore) or:

```bash
az sql db restore --dest-name budwheelsse-diag-20260811 --name budwheelsse \
  --resource-group <rg> --server opterproductionsweden1 --time "2026-08-11T12:25:00Z"
```

It bills as an extra database for as long as it exists, so drop it when done. Note the restored copy sits on the same server, so its compute may be shared depending on tier — check before restoring if the server is under load.

### Option D — hand the queries to someone who has the permission

Whoever owns the Azure subscription or acts as DBA can run §4–§8 and send back the `.sqlplan` files and result grids. Slower to iterate, but requires no permission change at all.

> **Note:** the SSMS Query Store GUI reports (§6d) read the same catalog views, so they fail with the identical error. The Azure Portal's **Query Performance Insight** blade is a partial exception — it surfaces top queries by duration from Query Store using Azure RBAC rather than T-SQL permissions, so `Reader` on the SQL database resource is enough. It will show you *whether* a `k2_inv2_*` statement burned 300 s and how its duration varies, but it will not give you plan XML or `ParameterCompiledValue`. Useful as a first look while a grant is pending.

---

## 4. Step 1 — Confirm Query Store is on and still holds the window

Query Store is enabled by default on Azure SQL Database and persists across failovers, so the two timed-out executions are very likely already recorded. The incident is only hours old, well inside the default 30-day retention.

```sql
SELECT actual_state_desc,
       desired_state_desc,
       query_capture_mode_desc,
       stale_query_threshold_days,
       current_storage_size_mb,
       max_storage_size_mb,
       readonly_reason
FROM   sys.database_query_store_options;
```

Expected: `actual_state_desc = READ_WRITE`. If it says `READ_ONLY`, check `readonly_reason` — storage full is the usual cause and it means capture stopped, possibly before the incident.

---

## 5. Step 2 — Find the timed-out executions precisely

A `SqlClient` command timeout is a client-initiated abort, which Query Store records as **`execution_type_desc = 'Aborted'`**. That gives an exact handle on the two failures rather than guessing from durations.

```sql
SELECT   q.query_id,
         p.plan_id,
         OBJECT_NAME(q.object_id)             AS proc_name,
         rs.execution_type_desc,
         rs.count_executions,
         rs.avg_duration  / 1000000.0         AS avg_sec,
         rs.max_duration  / 1000000.0         AS max_sec,
         rs.avg_cpu_time  / 1000000.0         AS avg_cpu_sec,
         rs.avg_logical_io_reads,
         rs.avg_physical_io_reads,
         rs.avg_rowcount,
         rs.avg_query_max_used_memory         AS avg_grant_pages,
         rs.first_execution_time,
         rs.last_execution_time,
         SUBSTRING(qt.query_sql_text, 1, 400) AS stmt
FROM     sys.query_store_runtime_stats rs
         JOIN sys.query_store_runtime_stats_interval rsi
              ON rsi.runtime_stats_interval_id = rs.runtime_stats_interval_id
         JOIN sys.query_store_plan       p  ON p.plan_id      = rs.plan_id
         JOIN sys.query_store_query      q  ON q.query_id     = p.query_id
         JOIN sys.query_store_query_text qt ON qt.query_text_id = q.query_text_id
WHERE    rsi.start_time >= '2026-08-11T11:30:00+00:00'
AND      rsi.start_time <= '2026-08-11T13:00:00+00:00'
ORDER BY rs.max_duration DESC;
```

Add `AND rs.execution_type_desc <> 'Regular'` to isolate only the aborted ones.

**Write down the `query_id` and `plan_id` of the top row — everything below keys off them.**

Reading the result:

| signal | reading |
|---|---|
| `max_sec` ≈ 300 with high `avg_cpu_sec` | genuinely executing — a plan/scan problem, continue to Step 3 |
| `max_sec` ≈ 300 with `avg_cpu_sec` near zero | it was **waiting**, not working — go to Step 5, this is a locking problem instead |
| `avg_rowcount` tiny but `avg_logical_io_reads` enormous | classic bad-plan signature: huge effort, few rows |
| same `query_id` appearing with several `plan_id`s at very different durations | plan instability — strong support for this hypothesis |

---

## 6. Step 3 — Capture the plan (do this **before** recompiling anything)

> **Correction to an earlier version of this document.** I originally wrote *"recompiling destroys the bad plan, capture it first"* and made §10 conditional on finishing this section. **That was wrong.** `sp_recompile` clears the entry in the **live plan cache** (`sys.dm_exec_cached_plans`); it does **not** touch Query Store, which persisted the plan XML and its runtime stats to disk when the query ran (default flush interval 15 minutes — the incident is hours old, so it is long since on disk). Query Store data is only removed by `ALTER DATABASE ... SET QUERY_STORE CLEAR` or by its retention policy.
>
> Practical consequence: **§10's recompile test can be run before this section**, and the evidence will still be here afterwards. That matters because §10 needs only `db_ddladmin`, which you already have, while this section needs a permission you do not.

### 6a. Save the plan XML

```sql
SELECT p.plan_id,
       p.query_id,
       p.is_forced_plan,
       p.compatibility_level,
       CAST(p.query_plan AS XML) AS plan_xml
FROM   sys.query_store_plan p
WHERE  p.plan_id IN (/* plan_id values from Step 2 */);
```

In SSMS, click the `plan_xml` cell — it opens in a new tab. **Save that tab as a file with a `.sqlplan` extension**, then reopen it; SSMS renders it as a graphical plan. Attach the `.sqlplan` files to the case.

Do this for **every** `plan_id` on the affected `query_id`, not just the slow one — the comparison between a fast plan and a slow plan for the same query is the most valuable artefact you can produce here.

### 6b. Extract the compiled parameter values — the actual smoking gun

Query Store stores the *compile-time* plan, which records the parameter values the optimizer used. If a plan was compiled for `@INT_Id = 3` and is being reused for `INT_Id = 1` customers, this proves it outright:

```sql
WITH XMLNAMESPACES (DEFAULT 'http://schemas.microsoft.com/sqlserver/2004/07/showplan')
SELECT   q.query_id,
         p.plan_id,
         OBJECT_NAME(q.object_id) AS proc_name,
         pr.value('@Column', 'nvarchar(128)')                    AS parameter_name,
         pr.value('@ParameterCompiledValue', 'nvarchar(4000)')   AS compiled_value
FROM     sys.query_store_plan  p
         JOIN sys.query_store_query q ON q.query_id = p.query_id
         CROSS APPLY (SELECT TRY_CAST(p.query_plan AS XML) AS px) AS x
         CROSS APPLY x.px.nodes('//ParameterList/ColumnReference') AS t(pr)
WHERE    OBJECT_NAME(q.object_id) LIKE 'k2_inv2_create_rows%'
AND      pr.value('@ParameterCompiledValue', 'nvarchar(4000)') IS NOT NULL
ORDER BY proc_name, p.plan_id, parameter_name;
```

Compare `compiled_value` for `@INT_Id` / `@CPR_Id` / `@DEL_Id` / `@REG_Id` against what CUS 5539 actually sends — from the ServerApi log, that was `@INT_Id=2, @CPR_Id=NULL, @DEL_Id=NULL, @REG_Id=NULL`. A mismatch on `@INT_Id` is the finding.

### 6c. What to look for in the graphical plan

* **`ParameterCompiledValue` vs the real values** (as above) — the direct evidence.
* **Estimated vs actual rows.** Query Store holds the *estimated* plan, so actual counts are not in it; but a nested-loop operator with `Estimated Number of Rows = 1` feeding something large is diagnostic on its own. For estimate-vs-actual you need an actual plan — see Step 7.
* **Nested Loops driving a large row count**, especially where the SP forces `INNER LOOP JOIN`.
* **Memory grant** — `avg_query_max_used_memory` from Step 2. A tiny grant on a plan that must sort many rows means spills.
* **A seek on `IX_DEL_ToBeInvoiced` with `DEL_DIS_Id = 2` as the only seek predicate** and everything else as a residual — expected given the non-sargable `OR @param` predicates, and worth confirming.

### 6d. SSMS GUI shortcut

Object Explorer → `budwheelsse` → **Query Store** →

* **Queries With High Variation** — this report is purpose-built for exactly this failure mode. Set the metric to Duration and look for `k2_inv2_*` entries with a large spread.
* **Top Resource Consuming Queries** — set the time window to the incident and switch the plan summary to see multiple plans per query side by side.

Both let you click a plan and hit **Compare Showplan** between the fast and slow plan, which is the fastest way to see what changed.

---

## 7. Step 4 — Working or waiting?

This distinguishes a plan problem from the `SYS_System` locking problem described in the companion document. It is the single most decisive query here.

```sql
SELECT   ws.wait_category_desc,
         OBJECT_NAME(q.object_id)                  AS proc_name,
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

| `wait_category_desc` | meaning |
|---|---|
| `Lock` | **this hypothesis is wrong** — it was blocked. See §4c of the companion document (the `SYS_System` exclusive lock held for the full transaction) |
| `Buffer IO` / `Buffer Latch` | reading pages — a scan/plan problem |
| `Memory` | waiting for a memory grant — supports the bad-plan/spill story |
| `CPU` or nothing significant | burning CPU in a bad plan |
| `Tran Log Write` / `Log Rate Governor` | Azure SQL log throttling on the writes, a different problem again |

---

## 8. Step 5 — Statistics and optimizer settings

```sql
-- Stale statistics, filtered ones first. These feed the estimates.
SELECT   OBJECT_NAME(s.object_id) AS table_name,
         s.name                   AS stats_name,
         s.has_filter,
         s.filter_definition,
         sp.last_updated,
         sp.rows,
         sp.rows_sampled,
         sp.modification_counter
FROM     sys.stats s
         CROSS APPLY sys.dm_db_stats_properties(s.object_id, s.stats_id) sp
WHERE    OBJECT_NAME(s.object_id) IN
         ('PIT_PriceItem','DEL_Delivery','PAS_PriceAddService',
          'DCO_DeliveryCorrection','PRI_Price','INR_InvoiceRow')
ORDER BY s.has_filter DESC, sp.last_updated;

-- Which optimizer protections are even available here.
SELECT name, compatibility_level FROM sys.databases WHERE name = DB_NAME();

SELECT name, value, is_value_default
FROM   sys.database_scoped_configurations
WHERE  name IN ('LEGACY_CARDINALITY_ESTIMATION','PARAMETER_SNIFFING',
                'QUERY_OPTIMIZER_HOTFIXES','LAST_QUERY_PLAN_STATS');

-- Is Azure automatic tuning already forcing plans?
SELECT name, desired_state_desc, actual_state_desc, reason_desc
FROM   sys.database_automatic_tuning_options;

SELECT p.plan_id, p.query_id, OBJECT_NAME(q.object_id) AS proc_name,
       p.is_forced_plan, p.force_failure_count, p.last_force_failure_reason_desc
FROM   sys.query_store_plan p
       JOIN sys.query_store_query q ON q.query_id = p.query_id
WHERE  p.is_forced_plan = 1;
```

Things that would matter:

* **`last_updated` old on `IX_PIT_DIS_Id_Filt`** with a large `modification_counter` → stale filtered statistics, a direct cause of bad estimates.
* **`compatibility_level` below 140** → no Memory Grant Feedback, no Adaptive Joins. Those are precisely the features that would otherwise blunt this failure mode, so an older compat level (common for legacy apps) means these SPs have no protection at all.
* **`PARAMETER_SNIFFING = OFF`** → someone has already globally disabled sniffing, which changes the picture entirely.
* **`is_forced_plan = 1` on a `k2_inv2_*` plan** → automatic tuning has pinned a plan, possibly a bad one. `sys.sp_query_store_unforce_plan` releases it.

---

## 9. Step 6 — Enable actual-plan capture (optional, useful for the next occurrence)

Query Store keeps only the estimated plan. This makes the *last actual* plan available, which is what shows estimate-vs-actual row counts and spill warnings:

```sql
ALTER DATABASE SCOPED CONFIGURATION SET LAST_QUERY_PLAN_STATS = ON;
```

Then, after a slow execution:

```sql
SELECT   qs.sql_handle, OBJECT_NAME(qs.objectid) AS proc_name, ps.query_plan
FROM     sys.dm_exec_query_stats qs
         CROSS APPLY sys.dm_exec_query_plan_stats(qs.sql_handle) ps
WHERE    OBJECT_NAME(qs.objectid) LIKE 'k2_inv2%';
```

There is a small per-query overhead. Reasonable to leave on while this case is open; agree it with whoever owns the database.

---

## 10. Step 7 — The recompile test

**This is the cheapest way to prove or kill the hypothesis, and with `db_ddladmin` you can run it today without any new permission.** It does not have to wait for the Query Store steps — see the correction at the top of §6: recompiling clears only the live plan cache, not Query Store's persisted record.

```sql
EXEC sp_recompile 'dbo.k2_inv2_create_rows_price_item';
EXEC sp_recompile 'dbo.k2_inv2_create_rows_delivery';
EXEC sp_recompile 'dbo.k2_inv2_create_rows_add_service';
EXEC sp_recompile 'dbo.k2_inv2_create_rows_direct_expense';
EXEC sp_recompile 'dbo.k2_inv2_create_rows_expense';
EXEC sp_recompile 'dbo.k2_inv2_create_rows_correction';
EXEC sp_recompile 'dbo.k2_inv2_create_invoice';
```

`sp_recompile` marks the procedure so its plan is rebuilt on next execution. It changes no data and no code. It briefly takes a schema-modification lock, so run it **between** invoice runs, not during one. Requires `ALTER` on the objects.

Then have support retry **[Alla]** for CUS 5539.

| outcome | conclusion | next |
|---|---|---|
| now fast | it was a bad cached plan — hypothesis confirmed | §11 for the durable fix |
| still ~300 s | plans are not the cause | go back to the Step 2 output and follow the statement it names; if waits showed `Lock`, see §4c of the companion document |

The test is safely repeatable and does not burn the evidence: Query Store keeps the pre-recompile plan and its runtime stats on disk, so §6 can still be done afterwards once the grant arrives. The one thing you lose is the live plan-cache entry, which the `sys.dm_exec_*` DMVs read — and those are blocked by the missing permission anyway.

If it comes back fast, record **which** plan replaced it once you have Query Store access: comparing the new fast plan against the old slow one for the same `query_id` is the artefact that turns "recompiling helped" into "here is exactly what the optimizer got wrong."

### Faster, reversible alternative

If Step 2 shows a **good** `plan_id` and a **bad** `plan_id` for the same `query_id`, you can pin the good one immediately — no deploy, no recompile, fully reversible:

```sql
EXEC sys.sp_query_store_force_plan @query_id = <query_id>, @plan_id = <good_plan_id>;
-- to undo:
EXEC sys.sp_query_store_unforce_plan @query_id = <query_id>, @plan_id = <good_plan_id>;
```

This is a good production stopgap while a proper fix is developed. Check `force_failure_count` afterwards to confirm the forced plan is actually being used.

---

## 11. Next steps — both possible with `db_ddladmin`, no new grant

The recompile test came back negative, so the priority now is to find **which statement** burns the 300 s. That does not actually need Query Store — `SET STATISTICS TIME` and `SET STATISTICS IO` require no special permission (unlike `SET STATISTICS XML` / `PROFILE`, which need `SHOWPLAN`).

> ## ✅ 11a RESULT, 2026-08-13 — `_rows_price_item` eliminated
>
> ```
> Table 'DEL_Delivery'.  Scan count 433093, logical reads 1384050, physical reads 0
> Table 'PIT_PriceItem'. Scan count 5,      logical reads 1711
> CPU time = 1296 ms,  elapsed time = 493 ms
> (0 rows affected)
> ```
>
> **The office-wide loop is confirmed to exist.** 433 093 seeks into `DEL_Delivery` ≈ **2 × the 216 435 pending price items** — one index seek plus one key lookup each — burning 1.38 M logical reads. The `Warning: The join order has been enforced because a local join hint is used` line confirms the forced `INNER LOOP JOIN` is dictating the shape, exactly as §2 predicted.
>
> **But it is not the timeout: 493 ms.** `physical reads 0` (warm cache) and `CPU 1296 ms > elapsed 493 ms` (it went parallel). This eliminates `k2_inv2_create_rows_price_item` as the cause and confirms the corrected arithmetic in the companion document's §4b rather than its original claim.
>
> ### Why "0 rows" does not weaken that conclusion
>
> The 433 093 seeks are paid **before** any of the customer/date filtering takes effect: `DEL_CUS_Id = @CUS_Id` and the `DEL_OrderDate` predicates sit in the loop join's `ON` clause and are evaluated as *residuals*, per row, after each seek. So the expensive part costs the same whether the answer is 0 rows or 9. The 493 ms is a fair measure of the office-wide scan regardless of the row count or the date range used.
>
> What the 0 rows *does* cost us is any information about downstream work — the `GROUP BY`/`HAVING` and the real `INSERT INTO INR_InvoiceRow`. For a customer this small that work is trivial, so it is a minor gap.
>
> ### ⚠️ The real problem with this test: it deleted the thing under investigation
>
> Confirmed by support: CUS 5539 **did not** have thousands of pending orders, and the failure **reproduced with only 9**. So the 9 is representative, §4a stays ruled out, and the cost genuinely does not scale with the customer's data.
>
> Which means my test was aimed at the wrong thing. I instructed you to **delete the `IDS_InvoiceDataSelection` `EXISTS`** on the grounds that it is a tautology in the `[Alla]` case. Logically that is true — it always evaluates to true. But **that predicate is the *only* structural difference between the slow `[Alla]` path and the fast individual-orders path.** Everything else in the statement is identical between the two. So the test removed precisely the mechanism under investigation and then reported the remainder as fast.
>
> **The 493 ms therefore proves less than it appears to.** It is a valid measurement of the office-wide `PIT` → `DEL` loop (which is real, and cheap). It is **not** a measurement of what `[Alla]` does differently, and it does not eliminate `_rows_price_item` as a whole — only its scan portion.
>
> A tautology is not free. The optimizer must still execute the semi-join, and the predicate form is hostile:
>
> ```sql
> AND COALESCE(IDS_CPR_Id, DEL_CPR_Id, 0) = ISNULL(DEL_CPR_Id, 0)
> AND ISNULL(IDS_PST_Id, DEL_PST_Id)      = DEL_PST_Id
> AND ISNULL(IDS_DEL_Id, DEL_Id)          = DEL_Id
> ```
>
> Every term wraps columns from **both** tables inside a function, so there is no sargable join key at all — a hash semi-join on `IDS_DEL_Id = DEL_Id` is impossible, and the expressions must be evaluated row by row. In the individual-orders case `IDS_DEL_Id` holds real values and the optimizer at least has selective data to work with; in the `[Alla]` case it has one all-NULL row and no usable predicate whatsoever.
>
> Two caveats against over-reading this, given how many times I have got ahead of the evidence in this case:
>
> * `IDS_InvoiceDataSelection` **does** have a covering index — `IX_IDS_INH_incl ON (IDS_INH_Id, OFF_Id) INCLUDE (IDS_CPR_Id, IDS_DEL_Id, IDS_PST_Id)` — so "huge unindexed table scanned per row" is *not* the story. Probing it by `@INH_Id` is cheap.
> * Whether the per-row expression evaluation costs 500 ms or 300 s is **measurable, not guessable**. See 11a-corrected below.
>
> ### Separately: check the repro still exists on production
>
> The 0 rows may simply be my placeholder dates (`'2026-07-01'`/`'2026-07-31'`), or may mean the orders have since been invoiced.
>
> ```sql
> SELECT COUNT(*) AS pending_now
> FROM   DEL_Delivery
> WHERE  OFF_Id = 1 AND DEL_CUS_Id = 5539
> AND    DEL_DIS_Id = 2 AND DEL_ReadyForInvoicing = 1 AND DEL_ScheduledTemplate = 0;
>
> SELECT   INH_Id, INH_InvoiceNo, INH_INP_Id, INH_CreateDate
> FROM     INH_InvoiceHeader
> WHERE    OFF_Id = 1 AND INH_CUS_Id = 5539 AND INH_CreateDate >= '2026-08-11'
> ORDER BY INH_CreateDate DESC;
> ```
>
> If `pending_now = 0`, production can no longer reproduce and you need the §3 Option C restore (`2026-08-11T12:25:00Z`) or a customer that fails today.

### 11a-corrected. Re-run 11a **with** the IDS `EXISTS`, both cases

This is the test that actually targets the `[Alla]` mechanism. It stays read-only and needs no new permission, because rather than inserting fake `IDS` rows (blocked by the FK to `INH_InvoiceHeader`) it reuses **real historical ones**.

**Step 1 — find one `INH_Id` of each shape:**

```sql
-- An [Alla]-style invoice: exactly one IDS row, all three filter columns NULL
SELECT TOP 5 IDS_INH_Id, COUNT(*) AS ids_rows
FROM     IDS_InvoiceDataSelection
WHERE    OFF_Id = 1
GROUP BY IDS_INH_Id
HAVING   COUNT(*) = 1
AND      MAX(ISNULL(IDS_DEL_Id,0)) = 0
AND      MAX(ISNULL(IDS_PST_Id,0)) = 0
AND      MAX(ISNULL(IDS_CPR_Id,0)) = 0;

-- An individual-orders invoice: many real DEL ids
SELECT   TOP 5 IDS_INH_Id, COUNT(*) AS ids_rows
FROM     IDS_InvoiceDataSelection
WHERE    OFF_Id = 1 AND IDS_DEL_Id IS NOT NULL
GROUP BY IDS_INH_Id
ORDER BY COUNT(*) DESC;

-- And how big is IDS overall?
SELECT COUNT(*) AS ids_total FROM IDS_InvoiceDataSelection WHERE OFF_Id = 1;
```

**Step 2 — the complete runnable script.** Self-contained: it finds both `INH_Id`s itself, then runs the query once per `IDS` shape with the `EXISTS` present. Read-only, no new permissions.

Set `@FromDate` / `@UntilDate` to the values support actually used. **Run the whole thing twice and use the second run's numbers**, so cold-cache effects do not distort the comparison.

```sql
SET NOCOUNT ON;
SET STATISTICS IO ON;
SET STATISTICS TIME ON;

DECLARE @OFF_Id int = 1,
        @CUS_Id int = 5539,
        @INT_Id int = 2,
        @CPR_Id int = NULL,
        @REG_Id int = NULL,
        @DEL_Id int = NULL,
        @DirectInvoiceOnly bit = 0,
        @FromDate  date = '2026-07-01',    -- ←← the dates support actually used
        @UntilDate date = '2026-07-31';

DECLARE @NegAsCorr bit;
SELECT  @NegAsCorr = OFF_InvoiceNegativePriceItemsAsCorrections
FROM    OFF_Office WHERE OFF_Id = @OFF_Id;

/* ---- find an [Alla]-shaped IDS: exactly one row, all three filter columns NULL ---- */
DECLARE @INH_Alla int, @INH_Ids int;

SELECT TOP (1) @INH_Alla = IDS_INH_Id
FROM     IDS_InvoiceDataSelection
WHERE    OFF_Id = @OFF_Id
GROUP BY IDS_INH_Id
HAVING   COUNT(*) = 1
AND      MAX(ISNULL(IDS_DEL_Id,0)) = 0
AND      MAX(ISNULL(IDS_PST_Id,0)) = 0
AND      MAX(ISNULL(IDS_CPR_Id,0)) = 0;

/* ---- find an individual-orders-shaped IDS: many real DEL ids ---- */
SELECT   TOP (1) @INH_Ids = IDS_INH_Id
FROM     IDS_InvoiceDataSelection
WHERE    OFF_Id = @OFF_Id AND IDS_DEL_Id IS NOT NULL
GROUP BY IDS_INH_Id
ORDER BY COUNT(*) DESC;

PRINT '=== INH_Id for [Alla] shape (1 all-NULL row): ' + ISNULL(CONVERT(varchar(20), @INH_Alla), 'NOT FOUND — abort') + ' ===';
PRINT '=== INH_Id for individual-orders shape (many DEL ids): ' + ISNULL(CONVERT(varchar(20), @INH_Ids), 'NOT FOUND — abort') + ' ===';

DECLARE @INH_Id int;

/* ================= RUN 1 : [Alla] shape ================= */
PRINT '';
PRINT '########## RUN 1 — [Alla] : single all-NULL IDS row ##########';
SET @INH_Id = @INH_Alla;

SELECT   DISTINCT DEL.OFF_Id, DEL_Id, PIT_Id,
         ISNULL(CRE_Name,'') AS CRE_Name, PST_Name, PVT_Name
FROM     PIT_PriceItem PIT
         INNER LOOP JOIN DEL_Delivery DEL
             ON  DEL.OFF_Id = PIT.OFF_Id
             AND DEL_Id = PIT_DEL_Id
             AND DEL_ScheduledTemplate = 0
             AND DEL_CUS_Id = @CUS_Id
             AND (DEL_OrderDate <= @UntilDate OR @DirectInvoiceOnly = 1)
             AND (DEL_OrderDate >= ISNULL(@FromDate, DEL_OrderDate) OR @DirectInvoiceOnly = 1)
             AND (DEL_DirectInvoice = 1 OR @DirectInvoiceOnly = 0)
             AND (ISNULL(DEL_CPR_Id,0) = ISNULL(@CPR_Id,0) OR @INT_Id <> 2)
             AND (COALESCE(DEL_REG_Id,-1) = @REG_Id OR COALESCE(@REG_Id, 0) = 0)
             AND (DEL_Id = @DEL_Id OR @INT_Id <> 3)
         LEFT OUTER JOIN CRE_CustomerReference CRE
             ON CRE.OFF_Id = DEL.OFF_Id AND CRE_Id = DEL_CRE_Id
         JOIN PST_PriceServiceType PST
             ON PST.OFF_Id = DEL.OFF_Id AND PST_Id = DEL_PST_Id
         JOIN PVT_PriceVehicleType PVT
             ON PVT.OFF_Id = DEL.OFF_Id AND PVT_Id = DEL_PVT_Id
         JOIN STS_Status STS
             ON  STS.OFF_Id = DEL.OFF_Id AND STS.STS_Id = DEL_STS_Id
             AND (STS_SendInvoice = 1 OR DEL_ValidOrder = 0)
         JOIN PRI_Price PRI
             ON  PRI.OFF_Id = PIT.OFF_Id AND PRI.PRI_PIT_Id = PIT_Id
             AND PRI.PRI_PTY_Id = 1
         LEFT OUTER JOIN IOR_InternetOrder IOR
             ON IOR.OFF_Id = DEL.OFF_Id AND IOR_DEL_Id = DEL_Id
WHERE    PIT_DIS_Id = 2
AND      DEL_DIS_Id IN (2,3)
AND      DEL_ReadyForInvoicing = 1
AND      PIT.OFF_Id = @OFF_Id
AND      IOR.OFF_Id IS NULL
AND      EXISTS (SELECT IDS_Id                      -- ←← THE PREDICATE UNDER TEST
                 FROM   IDS_InvoiceDataSelection IDS
                 WHERE  IDS.OFF_Id = DEL.OFF_Id
                 AND    IDS_INH_Id = @INH_Id
                 AND    COALESCE(IDS_CPR_Id, DEL_CPR_Id, 0) = ISNULL(DEL_CPR_Id, 0)
                 AND    ISNULL(IDS_PST_Id, DEL_PST_Id)      = DEL_PST_Id
                 AND    ISNULL(IDS_DEL_Id, DEL_Id)          = DEL_Id)
GROUP BY DEL.OFF_Id, DEL_Id, PIT_Id, CRE_Name, PST_Name, PVT_Name
HAVING   (SUM(PRI_Price_Standard) >= 0 OR @NegAsCorr = 0);

/* ================= RUN 2 : individual-orders shape ================= */
PRINT '';
PRINT '########## RUN 2 — individual orders : many real IDS_DEL_Id rows ##########';
SET @INH_Id = @INH_Ids;

SELECT   DISTINCT DEL.OFF_Id, DEL_Id, PIT_Id,
         ISNULL(CRE_Name,'') AS CRE_Name, PST_Name, PVT_Name
FROM     PIT_PriceItem PIT
         INNER LOOP JOIN DEL_Delivery DEL
             ON  DEL.OFF_Id = PIT.OFF_Id
             AND DEL_Id = PIT_DEL_Id
             AND DEL_ScheduledTemplate = 0
             AND DEL_CUS_Id = @CUS_Id
             AND (DEL_OrderDate <= @UntilDate OR @DirectInvoiceOnly = 1)
             AND (DEL_OrderDate >= ISNULL(@FromDate, DEL_OrderDate) OR @DirectInvoiceOnly = 1)
             AND (DEL_DirectInvoice = 1 OR @DirectInvoiceOnly = 0)
             AND (ISNULL(DEL_CPR_Id,0) = ISNULL(@CPR_Id,0) OR @INT_Id <> 2)
             AND (COALESCE(DEL_REG_Id,-1) = @REG_Id OR COALESCE(@REG_Id, 0) = 0)
             AND (DEL_Id = @DEL_Id OR @INT_Id <> 3)
         LEFT OUTER JOIN CRE_CustomerReference CRE
             ON CRE.OFF_Id = DEL.OFF_Id AND CRE_Id = DEL_CRE_Id
         JOIN PST_PriceServiceType PST
             ON PST.OFF_Id = DEL.OFF_Id AND PST_Id = DEL_PST_Id
         JOIN PVT_PriceVehicleType PVT
             ON PVT.OFF_Id = DEL.OFF_Id AND PVT_Id = DEL_PVT_Id
         JOIN STS_Status STS
             ON  STS.OFF_Id = DEL.OFF_Id AND STS.STS_Id = DEL_STS_Id
             AND (STS_SendInvoice = 1 OR DEL_ValidOrder = 0)
         JOIN PRI_Price PRI
             ON  PRI.OFF_Id = PIT.OFF_Id AND PRI.PRI_PIT_Id = PIT_Id
             AND PRI.PRI_PTY_Id = 1
         LEFT OUTER JOIN IOR_InternetOrder IOR
             ON IOR.OFF_Id = DEL.OFF_Id AND IOR_DEL_Id = DEL_Id
WHERE    PIT_DIS_Id = 2
AND      DEL_DIS_Id IN (2,3)
AND      DEL_ReadyForInvoicing = 1
AND      PIT.OFF_Id = @OFF_Id
AND      IOR.OFF_Id IS NULL
AND      EXISTS (SELECT IDS_Id
                 FROM   IDS_InvoiceDataSelection IDS
                 WHERE  IDS.OFF_Id = DEL.OFF_Id
                 AND    IDS_INH_Id = @INH_Id
                 AND    COALESCE(IDS_CPR_Id, DEL_CPR_Id, 0) = ISNULL(DEL_CPR_Id, 0)
                 AND    ISNULL(IDS_PST_Id, DEL_PST_Id)      = DEL_PST_Id
                 AND    ISNULL(IDS_DEL_Id, DEL_Id)          = DEL_Id)
GROUP BY DEL.OFF_Id, DEL_Id, PIT_Id, CRE_Name, PST_Name, PVT_Name
HAVING   (SUM(PRI_Price_Standard) >= 0 OR @NegAsCorr = 0);

SET STATISTICS TIME OFF;
SET STATISTICS IO OFF;
```

**Step 3 —** compare the two `elapsed time` values and the `IDS_InvoiceDataSelection` logical reads between RUN 1 and RUN 2.

> ## ✅ RESULT, 2026-08-13 — mechanism confirmed, magnitude still missing
>
> `@INH_Alla = 40`, `@INH_Ids = 207761`.
>
> | | RUN 1 — `[Alla]` (all-NULL IDS) | RUN 2 — individual orders |
> |---|---|---|
> | **elapsed** | **383 ms** | **31 ms** |
> | CPU | 1343 ms (parallel) | 48 ms |
> | `Worktable` logical reads | **433 304** | 0 |
> | `PIT_PriceItem` logical reads | **1 711** | **22** |
> | `DEL_Delivery` logical reads | 720 | 720 |
> | `IDS` logical reads | 312 (104 scans) | 312 (104 scans) |
>
> **The `[Alla]` mechanism is real and now measured: 12× slower, and the cause is visible in the numbers.** With real `IDS_DEL_Id` values the optimizer drives from `IDS` and touches almost nothing — `PIT_PriceItem` drops from 1 711 logical reads to **22**, and the 433 304-read worktable disappears entirely. With one all-NULL row it has no selective driver and materialises the whole office-wide set into a spool. That is exactly the §3 mechanism, demonstrated.
>
> Note the plan *shape* changed versus the EXISTS-less run: the ~433 k row volume moved from 433 093 seeks against `DEL_Delivery` into a 433 304-read `Worktable` spool. Same volume, different operator, similar cost (493 ms → 383 ms). And `IDS` itself is **not** expensive — 312 logical reads, 104 scans; the covering index does its job. The cost is the spooled office-wide row volume, not the `IDS` probe.
>
> **But 383 ms is not 300 s.** The ratio is right and the mechanism is right; the magnitude is ~800× short. So `k2_inv2_create_rows_price_item` is now conclusively **not** the timeout — in either selection mode.
>
> ### What this redirects to
>
> The structural pattern is confirmed: *all-NULL `IDS` → no selective driver → whole office-wide driving set materialised*. The question is now simply **which SP has a driving set big enough (or a fan-out bad enough) for that pattern to cost 300 s.** Known driving-set sizes:
>
> | SP | driving set | size |
> |---|---|---|
> | `_rows_price_item` | `PIT_DIS_Id = 2` | 216 435 → **measured 383 ms** |
> | `_rows_add_service` | `PAS_DIS_Id = 2` | 0 — cannot be it |
> | `_rows_direct_expense` | `DEX_DIS_Id = 2` | 0 — cannot be it |
> | `_rows_expense` | `PEX_DIS_Id = 2` | 39 — cannot be it |
> | **`_rows_delivery`** | `DEL_DIS_Id = 2` office-wide | **never measured** |
> | **`_rows_correction`** | `DCO_DIS_Id = 2` office-wide | **never measured** |
>
> By elimination it is `_rows_delivery` or `_rows_correction`.

> ### ❌ MEASURED 2026-08-13 — both eliminated, and with them the entire row-creation layer
>
> ```
> pending_del_officewide = 938
> pending_dco            = 15
> ```
>
> Final driving-set table:
>
> | SP | driving set | result |
> |---|---|---|
> | `_rows_price_item` | 216 435 | **measured 383 ms** |
> | `_rows_delivery` | **938** | 230× smaller than the one that took 383 ms |
> | `_rows_correction` | **15** | 14 000× smaller |
> | `_rows_expense` | 39 | — |
> | `_rows_add_service` / `_rows_direct_expense` | 0 | — |
>
> **None of the six row-creation SPs can produce a 300 s call.** The largest driving set in the entire layer is the one already measured at 383 ms; every other is orders of magnitude smaller. The `[Alla]` mechanism is real, confirmed and worth fixing — but it is **not** the timeout, and the office-wide-driving-table line of investigation (companion document §4b) is now closed as the cause.
>
> ### The rest of the chain, also checked
>
> * **No triggers anywhere in the schema** — the RedGate script tree has no `Triggers` folder and no `CREATE TRIGGER` in it. A per-row trigger on `INR_InvoiceRow` or `DEL_Delivery` would have been an excellent explanation; it is ruled out.
> * **`k2_inv2_complete_invoice` and its 7 children**: 44–161 lines each, **zero cursors or `WHILE` loops**, all scoped to `@INH_Id`. Not candidates.
> * **`_create_header` / `_get_next_invoice_number` / `_get_last_invoice_date`**: covered by `IX_INH_INP_Id_incl` and `IX_INH_CUS_Id_incl`.
> * **`_create_invoice_data_selection`**: its `WHILE` string parser runs **zero** iterations in the `[Alla]` case.
>
> ### What is left
>
> Only two things in the whole path have not been eliminated:
>
> 1. **Blocking on `SYS_System`** — the *first* statement of `k2_inv2_create_invoice` takes an exclusive lock on a single-row table and holds it for the whole transaction (companion §4c). If the 300 s is spent waiting here, zero work is done and every query-level measurement in this document would have looked fine — which is exactly what has happened.
> 2. **The construction-invoice path** (§11a-bis) — the only customer-switched branch, the only cursors in the chain, and it re-runs the entire SP. **Still unchecked; it is one query.**
>
> ### Stop reconstructing queries — instrument the whole SP
>
> I have spent several rounds extracting individual statements and timing them, eliminating them one at a time. That was the wrong method: `SET STATISTICS TIME ON` reports elapsed time for **every statement in every nested procedure**, so a single instrumented execution of `k2_inv2_create_invoice` would have named the 300 s statement immediately. It also answers "working or waiting", because if the time is in the `UPDATE SYS_System` on line 36 then it is blocking, not work. See §11e.

| result | meaning |
|---|---|
| all-NULL case ≫ many-ids case | **the `[Alla]` mechanism is found**, and it is this `EXISTS`. Fix by rewriting the predicate to be sargable, or by having the client always send `DEL_Ids` |
| both ≈ 500 ms | `_rows_price_item` really is exonerated — move to the other four row SPs and §11a-bis |

Also worth setting `@FromDate`/`@UntilDate` to the values support actually used, so the downstream `GROUP BY`/`HAVING` is exercised rather than short-circuited by an empty result.
>
> ### First, establish whether the production repro still exists
>
> ```sql
> -- Was it 9 on Tuesday and is it still 9? Or has it been invoiced away?
> SELECT COUNT(*) AS pending_now
> FROM   DEL_Delivery
> WHERE  OFF_Id = 1 AND DEL_CUS_Id = 5539
> AND    DEL_DIS_Id = 2 AND DEL_ReadyForInvoicing = 1 AND DEL_ScheduledTemplate = 0;
>
> -- Did anyone successfully invoice 5539 after the incident?
> SELECT   INH_Id, INH_InvoiceNo, INH_INP_Id, INH_CreateDate, INH_InvoiceDate
> FROM     INH_InvoiceHeader
> WHERE    OFF_Id = 1 AND INH_CUS_Id = 5539 AND INH_CreateDate >= '2026-08-11'
> ORDER BY INH_CreateDate DESC;
>
> -- Office-wide pending deliveries — never measured, and it is the driving set for _rows_delivery
> SELECT COUNT(*) AS pending_del_officewide
> FROM   DEL_Delivery WHERE OFF_Id = 1 AND DEL_DIS_Id = 2 AND DEL_ReadyForInvoicing = 1;
>
> -- Office-wide pending corrections — the driving set for _rows_correction, also never measured
> SELECT COUNT(*) AS pending_dco FROM DCO_DeliveryCorrection WHERE OFF_Id = 1 AND DCO_DIS_Id = 2;
> ```
>
> If `pending_now = 0`, production can no longer reproduce this and every remaining test needs either the restored copy or a **different** customer that currently fails. Ask support which customers are failing *today*.

### 11a-bis. The construction-invoice path — strongest untested customer-specific lead

Worth checking before grinding through the remaining four row SPs, because unlike everything else in the chain it is **switched on per customer** and it is the only place in the whole path with **cursors**.

[`k2_inv2_create_invoice.sql:168`](Database/RedGateScripts/Stored%20Procedures/dbo.k2_inv2_create_invoice.sql:168):

```sql
exec k2_inv2_filter_construction_items @OFF_Id, @INH_Id, @ConstructionOnly, @DirectInvoiceOnly, @RowsRemoved output
if @ConstructionOnly = 0 and @RowsRemoved > 0 begin
    exec k2_inv2_create_invoice ...   -- ← the ENTIRE SP runs a second time
end
```

[`k2_inv2_filter_construction_items.sql:38`](Database/RedGateScripts/Stored%20Procedures/dbo.k2_inv2_filter_construction_items.sql:38) exits immediately unless **`CUS_ConstructionCompany = 1` AND `OFF_SeparateConstructionInvoices = 1`**. If both hold, it calls [`k2_inv2_reset_rows`](Database/RedGateScripts/Stored%20Procedures/dbo.k2_inv2_reset_rows.sql:126) with `@Recalculate` defaulting to **0**, which means the guarded block at line 122 runs — and that block contains **three nested cursors** (lines 268–319) calling `k2_sw_pri_calculate_income`, `k2_sw_pri_calculate_cost_shi` and `k2_sw_pri_calculate_cost_pas` once per order, per shipment and per add-service. Those are full pricing recalculations, executed row by row.

So a construction-company customer with invoice-correction rows (`DCO_CTY_Id = 5`) gets: per-row pricing recalculation via nested cursors, **plus** a complete second pass through `k2_inv2_create_invoice` including all five office-wide row scans. That is a credible 300 s, and it is credibly customer-specific.

```sql
-- Is CUS 5539 a construction company, and is the office setting on?
SELECT CUS.CUS_Id, CUS.CUS_Name, CUS.CUS_ConstructionCompany,
       OFI.OFF_SeparateConstructionInvoices, OFI.OFF_TAX_Id_ReverseConstruction
FROM   CUS_Customer CUS JOIN OFF_Office OFI ON OFI.OFF_Id = CUS.OFF_Id
WHERE  CUS.OFF_Id = 1 AND CUS.CUS_Id = 5539;

-- How many customers are construction companies? If it is a small set, cross-check it
-- against the customers support reports as slow — that correlation would be the finding.
SELECT CUS_ConstructionCompany, COUNT(*) AS customers
FROM   CUS_Customer WHERE OFF_Id = 1 GROUP BY CUS_ConstructionCompany;
```

**If `CUS_ConstructionCompany = 1` for the slow customers and `0` for the fast ones, that is the answer to the question that has been open since the start.**

### 11a. Time the row-creation queries directly — the single most useful thing to do next

For the `[Alla]` case the `IDS_InvoiceDataSelection` `EXISTS` is a **tautology** — one row with all three filter columns NULL makes every `ISNULL(IDS_x, DEL_x) = DEL_x` term true. So it can simply be deleted, and the remaining query is a faithful, **read-only** reproduction of what `[Alla]` runs. Two changes from the source: drop `INTO #inr_tmp`, drop the `EXISTS`.

This is the prime suspect, `k2_inv2_create_rows_price_item`:

```sql
SET STATISTICS TIME ON;
SET STATISTICS IO ON;

DECLARE @OFF_Id int = 1,
        @CUS_Id int = 5539,
        @INT_Id int = 2,
        @CPR_Id int = NULL,
        @REG_Id int = NULL,
        @DEL_Id int = NULL,
        @DirectInvoiceOnly bit = 0,
        @FromDate date  = '2026-07-01',   -- ← use the dates support actually used
        @UntilDate date = '2026-07-31';

DECLARE @OFF_InvoiceNegativePriceItemsAsCorrections bit;
SELECT  @OFF_InvoiceNegativePriceItemsAsCorrections = OFF_InvoiceNegativePriceItemsAsCorrections
FROM    OFF_Office WHERE OFF_Id = @OFF_Id;

SELECT   DISTINCT DEL.OFF_Id, DEL_Id, PIT_Id,
         ISNULL(CRE_Name,'') AS CRE_Name, PST_Name, PVT_Name
FROM     PIT_PriceItem PIT
         INNER LOOP JOIN DEL_Delivery DEL
             ON  DEL.OFF_Id = PIT.OFF_Id
             AND DEL_Id = PIT_DEL_Id
             AND DEL_ScheduledTemplate = 0
             AND DEL_CUS_Id = @CUS_Id
             AND (DEL_OrderDate <= @UntilDate OR @DirectInvoiceOnly = 1)
             AND (DEL_OrderDate >= ISNULL(@FromDate, DEL_OrderDate) OR @DirectInvoiceOnly = 1)
             AND (DEL_DirectInvoice = 1 OR @DirectInvoiceOnly = 0)
             AND (ISNULL(DEL_CPR_Id,0) = ISNULL(@CPR_Id,0) OR @INT_Id <> 2)
             AND (COALESCE(DEL_REG_Id,-1) = @REG_Id OR COALESCE(@REG_Id, 0) = 0)
             AND (DEL_Id = @DEL_Id OR @INT_Id <> 3)
         LEFT OUTER JOIN CRE_CustomerReference CRE
             ON CRE.OFF_Id = DEL.OFF_Id AND CRE_Id = DEL_CRE_Id
         JOIN PST_PriceServiceType PST
             ON PST.OFF_Id = DEL.OFF_Id AND PST_Id = DEL_PST_Id
         JOIN PVT_PriceVehicleType PVT
             ON PVT.OFF_Id = DEL.OFF_Id AND PVT_Id = DEL_PVT_Id
         JOIN STS_Status STS
             ON  STS.OFF_Id = DEL.OFF_Id AND STS.STS_Id = DEL_STS_Id
             AND (STS_SendInvoice = 1 OR DEL_ValidOrder = 0)
         JOIN PRI_Price PRI
             ON  PRI.OFF_Id = PIT.OFF_Id AND PRI.PRI_PIT_Id = PIT_Id
             AND PRI.PRI_PTY_Id = 1
         LEFT OUTER JOIN IOR_InternetOrder IOR
             ON IOR.OFF_Id = DEL.OFF_Id AND IOR_DEL_Id = DEL_Id
WHERE    PIT_DIS_Id = 2
AND      DEL_DIS_Id IN (2,3)
AND      DEL_ReadyForInvoicing = 1
AND      PIT.OFF_Id = @OFF_Id
AND      IOR.OFF_Id IS NULL
-- ⚠️ SUPERSEDED: this script omits the IDS EXISTS. That was a mistake — see 11a-corrected.
--    The omitted predicate is the ONLY difference between [Alla] and individual orders,
--    so this version cannot test the [Alla] mechanism. Run 11a-corrected instead.
GROUP BY DEL.OFF_Id, DEL_Id, PIT_Id, CRE_Name, PST_Name, PVT_Name
HAVING   (SUM(PRI_Price_Standard) >= 0 OR @OFF_InvoiceNegativePriceItemsAsCorrections = 0);
```

Read `elapsed time` from the Messages tab, and the logical-read counts per table.

If it returns quickly, apply the same two transformations to the others and time each — `_rows_delivery`, `_rows_add_service`, `_rows_expense`, `_rows_direct_expense`, `_rows_correction`. **One of them will be the 300 s.** That identifies the culprit statement with no permissions and no production write.

Then two variations on whichever one is slow, which together separate the two surviving suspects:

| variation | if it gets fast |
|---|---|
| remove `INNER LOOP JOIN` → plain `JOIN`, and drop `FORCESEEK` / `INDEX=` hints | the **forced hints** are the problem (§2 amplifier 1) |
| keep hints, but first `UPDATE STATISTICS … WITH FULLSCAN` (see 11b) | **stale statistics** are the problem (§2 amplifier 2) |

### 11e. Instrument the whole SP — the test that should have come first

`SET STATISTICS TIME ON` prints elapsed time for **every statement in every nested procedure**. One instrumented execution names the 300 s statement outright, and distinguishes blocking from work: if the time lands on the `UPDATE SYS_System` at [`k2_inv2_create_invoice.sql:36`](Database/RedGateScripts/Stored%20Procedures/dbo.k2_inv2_create_invoice.sql:36) it is a lock wait, not a slow query.

Needs only `EXECUTE`, which you have. The `ROLLBACK` is **mandatory** — the SP writes `INH`/`INR`/`PRI` rows and flips `DEL`/`PIT` statuses.

**Step 1 — pick an existing invoice period** (the originals, 21472/21474, were deleted; and creating one is unnecessary since the SP only reads its dates and number series):

```sql
SELECT TOP 10 INP_Id, INP_PeriodNo, INP_InvoiceFromDate, INP_InvoiceUntilDate, INP_Complete
FROM     INP_InvoicePeriod
WHERE    OFF_Id = 1 AND INP_IPT_Id = 1
ORDER BY INP_Id DESC;
```

Pick one whose date range covers CUS 5539's pending orders.

**Step 2 — run it, rolled back:**

```sql
SET STATISTICS TIME ON;
SET NOCOUNT ON;

BEGIN TRANSACTION;

BEGIN TRY
    EXEC k2_inv2_create_invoice
         @OFF_Id             = 1,
         @CUS_Id             = 5539,
         @INP_Id             = <INP_Id from step 1>,
         @USR_Id             = 22,
         @INH_Id_Referer     = 0,
         @RPT_Id             = 0,
         @INH_InvoiceDate    = '2026-08-13',
         @INH_ValueDate      = '2026-08-13',
         @INH_AccountingDate = '2026-08-13',
         @INH_Text           = '',
         @ApplyFee           = 1,
         @DirectInvoiceOnly  = 0,
         @ConstructionOnly   = 0,
         @InvoiceCorrection  = 0,
         @INH_Id             = 0,
         @INT_Id             = 2,
         @CPR_Id             = NULL,
         @DEL_Id             = NULL,
         @REG_Id             = NULL,
         @CPR_Ids            = NULL,   -- NULL = [Alla]
         @PST_Ids            = NULL,
         @DEL_Ids            = NULL;
END TRY
BEGIN CATCH
    PRINT 'ERROR ' + CONVERT(varchar(20), ERROR_NUMBER()) + ': ' + ERROR_MESSAGE();
END CATCH

IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;   -- MANDATORY
SET STATISTICS TIME OFF;

SELECT @@TRANCOUNT AS should_be_zero;
```

Then scan the Messages tab for the one `SQL Server Execution Times` block with a large `elapsed time`. That is the answer.

> ⚠️ **Run off-hours, or on the §3 Option C restored copy.** The SP's first statement takes an exclusive lock on `SYS_System` and holds it until the rollback completes — so for however long this runs, **every other invoice operation in the installation is blocked**. That is the same defect described in companion §4c, and here it works against you.
>
> On the restored copy there is no such concern, and it is the better venue: you own it, so `SHOWPLAN` and Query Store also become available.

### 11b. Refresh statistics — also available today

`db_ddladmin` grants `ALTER ANY SCHEMA`, which is enough for `UPDATE STATISTICS`. This directly tests surviving suspect #2, and unlike `sp_recompile` it fixes the *numbers* the optimizer reasons from, not just the timing of compilation:

```sql
UPDATE STATISTICS dbo.PIT_PriceItem          WITH FULLSCAN;
UPDATE STATISTICS dbo.DEL_Delivery           WITH FULLSCAN;
UPDATE STATISTICS dbo.PRI_Price              WITH FULLSCAN;
UPDATE STATISTICS dbo.INR_InvoiceRow         WITH FULLSCAN;
UPDATE STATISTICS dbo.PAS_PriceAddService    WITH FULLSCAN;
UPDATE STATISTICS dbo.DCO_DeliveryCorrection WITH FULLSCAN;
```

`FULLSCAN` on tables this size is IO-heavy — **run it off-hours**, and consider starting with `PIT_PriceItem` alone since that carries the suspect filtered index. `UPDATE STATISTICS` invalidates dependent plans automatically, so no separate `sp_recompile` is needed afterwards. Then retry `[Alla]` for CUS 5539.

### 11c. Complete the recompile test properly

My original list missed most of the chain. This covers all of it:

```sql
DECLARE @sql nvarchar(max) = N'';
SELECT  @sql = @sql + N'EXEC sp_recompile ''' 
             + QUOTENAME(SCHEMA_NAME(schema_id)) + N'.' + QUOTENAME(name) + N''';' + CHAR(13)
FROM    sys.procedures
WHERE   name LIKE 'k2_inv%';
PRINT @sql;               -- review first
-- EXEC sys.sp_executesql @sql;
```

### 11d. A zero-permission signal worth collecting

Have support reproduce it **three times** and record each duration.

* Consistently ~300 s → deterministic work. Points at 11a.
* Wildly variable (30 s, 300 s, 90 s) → contention. Points at the `SYS_System` lock, companion document §4c.

Note that the `[Alla]`-vs-individual-orders split is itself evidence **against** pure blocking: a lock does not care what `IDS_InvoiceDataSelection` contains, so a reliable difference between the two selection modes implies the time is going into *work*, not waiting. That is the main reason 11a is the better bet — but 11d costs nothing and would settle it.

---

## 12. The durable fix, if the plan angle is ever confirmed

Add `OPTION (RECOMPILE)` to the `INSERT`/`SELECT` in each of the affected `k2_inv2_create_rows_*` SPs.

`OPTION (RECOMPILE)` rather than `OPTIMIZE FOR UNKNOWN` because:

* These statements run at most a handful of times per invoice, so compile cost is negligible against a 300 s worst case.
* Recompiling per execution lets the optimizer see the **actual** `@INT_Id` / `@CPR_Id` / `@DEL_Id` / `@REG_Id` for this customer, *and* the real contents of `IDS_InvoiceDataSelection` — which is a per-invoice work table whose cardinality the optimizer otherwise has no hope of estimating.
* It makes the `[Alla]` path (1 all-NULL `IDS` row) and the individual-orders path (N real ids) get appropriately different plans instead of sharing one. That is the `[Alla]` cliff described in the companion document, addressed at the plan level.

Do alongside it:

* **`UPDATE STATISTICS` with `FULLSCAN` on the filtered indexes** (`IX_PIT_DIS_Id_Filt`, `IX_PAS_DIS_Id`), and add a maintenance job — filtered-index statistics do not auto-update reliably.
* **Re-examine the forced hints.** They were added to avoid deadlocks, but they also block recovery from a bad estimate. If `OPTION (RECOMPILE)` produces good estimates, some may be unnecessary — but do **not** remove them without re-testing the deadlock scenarios they were added for. That is a separate, carefully-tested change.

Because the SPs are byte-identical between `releases/2025.12` and `master` (verified against `b2ee9c7831`), any fix here must be authored on `master` and cherry-picked to the release branch — this is not a release-branch-only defect.

---

## 13. Relationship to the other findings

Three separate issues surfaced in this case. They are independent and should not be conflated:

| # | issue | status |
|---|---|---|
| 1 | **This document** — bad cached plans causing per-`CUS_Id` slowness | hypothesis, testable, best current explanation of the reported symptom |
| 2 | **Stale-data leak** — 200 289 unpriceable `PIT_DIS_Id = 2` rows, ~96 % never-priced `AddService` items, leaking at ~2 000/month since at least 2013 | **confirmed by measurement**; see companion document §4b and §7 fix 1/3 |
| 3 | **Office-wide driving tables** — five `_rows_*` SPs drive from office-wide pending-item backlogs with the customer filter on the inner side of a forced loop join | **confirmed from code**; a real inefficiency, but the corrected arithmetic shows it is ~1 s warm, not 300 s |

Issue 2 is real and worth fixing on its own merits regardless of what Steps 1–7 conclude. Issues 1 and 3 interact: a bad plan (1) is far more damaging *because* the driving set is office-wide (3), and fixing either reduces the blast radius of the other.
