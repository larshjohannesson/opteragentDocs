# Bug report: Client crashes with unhandled `ConfigurationErrorsException` on `user.config` access

- **Reported:** 2026-09-02
- **Reported by:** Lars Johannesson (from a crash log seen for GlennGrimhag / Uppdraget i Göteborg Aktiebolag)
- **Assembly:** `Opter.Main.Client`
- **Area:** Login (`Opter.Main.ViewModels.Login.LogonWindowViewModel`), client startup (`Opter.Main.Client.App`)
- **Data source:** Azure Log Analytics `Logs_CL`, export `query_data (34).csv` (27 rows, 2026-08-07 → 2026-09-02)

## Summary

The Opter Main Client crashes with an unhandled `System.Configuration.ConfigurationErrorsException`
whenever .NET's `SettingsBase` fails to read or write the per-user `user.config` file under
`%LocalAppData%\Opter.Main.Client\...\user.config`. No code in the client caught this exception on
the write path, so it propagated to the WPF dispatcher and terminated the application with **no
error shown to the user** — the client simply disappears.

Log data confirms **two distinct triggers** for the same underlying symptom (an uncaught
`ConfigurationErrorsException` around `user.config`):

- **Pattern A — save fails at login** (dominant pattern: 26 rows / 10 crash attempts / 6 customers)
  - Trigger: `.NET` can't write the temp `.newcfg` file, or create its folder
  - Underlying `IOException`: "not enough space on the disk" (or, once, directory creation hit the same error)
  - Call site: `UserClientSettings.Save()` ← `LogonWindowViewModel.UpdateUserSettings()` ← `Login()`, i.e. **after a successful login**
- **Pattern B — read fails at startup** (1 row / 1 customer, related but distinct)
  - Trigger: `.NET` can't open `user.config` for read
  - Underlying `IOException`: "The file cannot be accessed by the system" (file locked/in use)
  - Call site: `UserClientSettings.get_StayLoggedIn()` ← `App.WPFMainLoop()` ← `App.StartUp()`, i.e. **before the login window is even shown**

## Impact (evidence)

All rows below are `Level = Error` in `Logs_CL`, `Message` = `Application crashed. { "unhandledCrash": true, ... }` where present.

### Pattern A — disk full while saving login settings

- **Fraktlogistik AB** — machine `FL-HEJE`, user Henrik Jernkrok, version `20251200.000312` — 3 failed login attempts, 2026-09-02 10:25:54 → 10:42:48 — "Application crashed" flag present
- **Ekdahl Miljö** — machine `EKDAHL-W052`, user carl.kensmar, version `20251200.000237` — 1 failed login attempt, 2026-09-02 05:52:18 — flag absent
- **Uppdraget i Göteborg Aktiebolag** — machine `DESKTOP-BP1TNO5`, user GlennGrimhag, version `20251200.000255` — 2 failed login attempts, 2026-08-11 05:59:15 → 05:59:45 — flag absent
- **HOLSHIP NORGE AS** — machine `021-017-MLU`, user mlu, version `20250600.000517` — 2 failed login attempts, 2026-08-12 07:54:08 → 07:54:51 — flag absent
- **Förlängda armen** — machine `farmen-avd-0`, user nicolas.veizaga, version `20250600.000506` — 1 failed login attempt, 2026-08-12 10:14:46 — flag absent
- **UT Transport i Norr AB** — machine `5CG3364394`, user zla (Zannah Norell), version `20251200.000221` — 1 failed login attempt, 2026-08-07 08:05:53 — flag absent

- **6 customers, 6 machines/users, 10 distinct failed login attempts, 26 log rows over 26 days** (2026-08-07 → 2026-09-02).
- Spans **both actively serviced release branches** — `releases/2025.06` (builds `.000506`, `.000517`) and `releases/2025.12` (builds `.000221` → `.000312`) — so this is a long-standing bug, not a recent regression.
- The underlying `IOException` message text is locale-dependent (`"Det finns inte tillräckligt med utrymme på disken."` on Swedish-locale Windows, `"There is not enough space on the disk."` on Norwegian-locale Windows for HOLSHIP) — same Win32 `ERROR_DISK_FULL`/`ERROR_HANDLE_DISK_FULL` condition in both cases, not a Sweden-specific issue.
- One occurrence (UT Transport i Norr AB / zla) failed at `Directory.CreateDirectory` rather than the temp-file write — same root cause (disk full), one I/O step earlier (the per-version settings folder didn't exist yet), same call stack (`UserClientSettings.Save()` ← `UpdateUserSettings()` ← `Login()`).
- **"Application crashed" flag confirms the reporter's hypothesis**: it's present only on the two newest builds in the sample (`.000288`, `.000312`); every older build (`.000221`, `.000237`, `.000255`, and both `2025.06` builds) is missing it, even though the crash itself clearly happened (client disappeared, exception logged). The flag is new client-side crash instrumentation, not a marker of a different bug.

### Pattern B — settings file locked/inaccessible at startup (related, distinct)

- **BDX Företagen** — machine `BXOJY01S-CTX901`, user BTR8773, version `20251200.000288` — 2026-08-26 05:37:37 — "Application crashed" flag present

Stack trace (abbreviated):
```
System.Configuration.ConfigurationErrorsException: Configuration system failed to initialize
 ---> ConfigurationErrorsException: An error occurred loading a configuration file: The file cannot be accessed by the system.
 ---> System.IO.IOException: The file cannot be accessed by the system. : '...\user.config'.
   at Opter.Main.Client.UserClientSettings.GetSetting(String key)
   at Opter.Main.Client.UserClientSettings.get_StayLoggedIn()
   at Opter.Main.Client.App.WPFMainLoop(...)
   at Opter.Main.Client.App.StartUp(StartupEventArgs e)
   at Opter.Main.Client.App.OnStartup(StartupEventArgs e)
```

This is **not the same code path** as Pattern A. It happens while the client is starting up, reading
`StayLoggedIn` to decide whether to auto-log-in — before `LogonWindowViewModel` is even constructed.
`App.xaml.cs` already has a narrow guard around this ([App.xaml.cs:188-198](../../Main/Client/App.xaml.cs))
that touches `Default.DTT_Id` once and deletes+reloads the config file if that specific read throws —
but it doesn't cover the later `StayLoggedIn` read, so a transient lock (e.g. antivirus scanning the
file, or a second client instance starting concurrently) still crashes the client unhandled. Only one
occurrence in this dataset, but the same class of bug (unhandled config I/O exception → silent crash)
as Pattern A.

## Root cause

`Opter.Main.Client.UserClientSettings.Save()` (and the read path used by `App.WPFMainLoop`) call
straight into `System.Configuration.SettingsBase`/`ApplicationSettingsBase` with no exception
handling anywhere in the call chain. Any transient local I/O failure — disk full, a locked file, a
missing folder — becomes an unhandled `ConfigurationErrorsException` that reaches the WPF dispatcher
and kills the process with no user-visible message.

## Suggested fix

Both patterns share the same shape of fix: catch `ConfigurationErrorsException` at the point where
`UserClientSettings` touches `SettingsBase`, and degrade gracefully instead of letting it reach the
WPF dispatcher unhandled.

- **Pattern A** — wrap the `_userClientSettings.Save()` call in
  `LogonWindowViewModel.UpdateUserSettings()` in a `try/catch (ConfigurationErrorsException)`. The
  login itself has already succeeded at that point, so the catch should show the user a warning
  (settings couldn't be saved, most likely due to disk space, and won't be remembered next time) and
  then continue rather than aborting — there is no reason to crash or block login over a failed
  settings write.
- **Pattern B** — wrap the `StayLoggedIn` read (and ideally the other `UserClientSettings` reads used
  during `App.StartUp`/`WPFMainLoop`) in `App.xaml.cs`. The existing guard at
  [App.xaml.cs:188-198](../../Main/Client/App.xaml.cs), which touches `Default.DTT_Id` once and
  deletes+reloads the config file if that specific read throws, does not cover this later read — a
  transient lock (e.g. antivirus scanning the file, or a second client instance starting
  concurrently) still crashes the client unhandled. A locked/inaccessible `user.config` here should
  degrade to "treat as not logged in" rather than crash.

## Recommendations

1. Implement the fixes above for both patterns.
2. Confirm whether `releases/2025.06` (still hit by this bug per HOLSHIP NORGE AS and Förlängda armen, both on that branch) needs the fix backported, or whether those customers are scheduled to move to `releases/2025.12`+.
3. Since the underlying trigger for Pattern A is genuinely low disk space on the end-user's machine, consider whether IT/support guidance to affected customers (Fraktlogistik AB, Ekdahl Miljö, Uppdraget i Göteborg Aktiebolag, HOLSHIP NORGE AS, Förlängda armen, UT Transport i Norr AB) is warranted independently of the code fix — a fix stops the crash, but the user's settings genuinely won't persist until they free up disk space.
