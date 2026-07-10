# WiX / MSI Installer — Lessons Learned

Field notes from building and debugging the Facility Dashboard Platform installer (WiX 7, MSI,
June 2026). Every item here was hit for real, diagnosed from logs, fixed, and verified. Written as
a reference for building WiX installers in future sessions — the *symptom* column is what you'll
actually see, which is rarely the real cause.

Working examples of everything below: `deploy/installer/Package.wxs`,
`deploy/installer-actions/` (DTF custom actions), `deploy/build-release.ps1`.

---

## 1. Elevation and the misleading "privileges" errors

| Symptom | Real cause |
| --- | --- |
| Shield button shows, but no UAC prompt appears | The account already holds a full admin token (VM built-in Administrator, or Admin Approval Mode off). msiexec elevates **silently**. Not a bug. |
| "Verify that you have sufficient privileges to **install** system services" (error **1923**) — while fully elevated | `ServiceInstall Account="[SERVICEACCOUNT]"` with an empty property formats to an empty account name, and `CreateService` rejects it. **Default the property to `LocalSystem`** — never let a service account property be empty. |
| "Verify that you have sufficient privileges to **start** system services" (error **1920**) | The service was *installed* fine but **exited during startup** (bad config, unreachable database, by-design startup gate). Nothing to do with privileges. |

Windows Installer reports almost every service failure as a privileges problem. When a user says
"it says I need elevation," read the log before believing it.

**Verify elevation metadata** without installing: Summary Information **Word Count (PID 15)** —
bit 8 set = "elevation not required" (no UAC will be requested); and check `ALLUSERS=1` in the
Property table. Both readable via the `WindowsInstaller.Installer` COM object.

## 2. Services

- **Never start a service via `ServiceControl Start`** if the service can legitimately refuse to
  start (e.g., a schema/startup gate). Even with `Wait="no"`, a service that exits during start
  fails the whole install with error 1920. Instead: best-effort start via a deferred
  `WixQuietExec` custom action running `sc.exe start <name>` with `Return="ignore"`, scheduled
  `After="RemoveExistingProducts"` (so an upgrade's old-product removal can't stop it again).
- **Stop** is fine as `ServiceControl Stop="both" Remove="uninstall" Wait="yes"` — the wait is
  needed so files can be replaced on upgrade.
- Add `util:ServiceConfig` failure actions (restart on crash) while you're in the component.
- The `ServiceInstall` must live in the component that owns the service EXE (its KeyPath).

## 3. Major upgrades

- **Use `MajorUpgrade Schedule="afterInstallExecute"`** for any product with a service and/or a
  preserved config file. The default (`afterInstallValidate`) has two traps:
  1. **NeverOverwrite + costing race:** file-install decisions are made at costing, while the old
     product still exists → "file exists, skip it" → old product removal deletes the file → the
     new install doesn't lay it down → anything that touches it later fails (for us: the
     config-write action, error 1603, upgrade rolls back).
  2. **No rollback of the removal:** the old product is removed *before* the install transaction,
     so a failed upgrade leaves **nothing** installed.
  With `afterInstallExecute` both are fixed: the config file survives untouched, failed upgrades
  roll back to the old version, and superseded files are still removed (components absent from
  the new package are deleted; shared components are reference-counted and kept).
- **Keep component GUIDs stable across versions** (hardcode them) — reference counting during the
  old product's removal depends on it.
- `AllowSameVersionUpgrades="no"` + `DowngradeErrorMessage` enforces strictly-newer upgrades.
  Side effect: re-running the same version enters **maintenance mode** (Repair/Remove), not the
  wizard — bump the version for every fix you hand to a tester.
- **ProductCode regenerates per build** (no explicit `ProductId`). `msiexec /x <newer .msi file>`
  cannot uninstall an older build of the "same" version — uninstall by ProductCode from
  `HKLM\...\Uninstall\*` instead.
- **Bump the payload's file version, not just the MSI `ProductVersion`.** MSI overwrites a file on
  upgrade only when the incoming file's version is *higher*. If the exe's `FileVersion` is hardcoded
  and unchanged, a major upgrade with a bumped `ProductVersion` installs the new package but silently
  **keeps the old exe on disk** (same file version → no overwrite), so the fix never lands — the
  symptom is "I upgraded but the bug is still there." Stamp `FileVersion` = `ProductVersion` at
  publish (`dotnet publish -p:Version=x.y.z`). Hit for real: a task-XML fix didn't take across a
  0.1.0→0.1.1 upgrade because both exes were version 0.1.0.0.

## 4. Preserving operator configuration across upgrades

Three cooperating pieces (all in `Package.wxs` + `CustomActions.cs`):

1. The config file component is `NeverOverwrite="yes"` (works correctly **only** with
   `Schedule="afterInstallExecute"`, see §3).
2. A managed `ReadExistingConfig` action (immediate, **both** UI and execute sequences — the
   execute one is what protects **silent** upgrades) locates the existing config — via a
   registry-recorded install dir (`RegistrySearch` + a `RegistryValue` component you write at
   install) — and primes the MSI properties from it. Explicit command-line properties win.
3. The config-write command is composed by a managed action that **omits any property still at
   its installer default**, so upgrades never silently reset operator-edited values the operator
   didn't pass. A formatted `SetProperty` string can't express this; managed code can.

## 5. Managed (DTF) custom actions

- The CA project **must embed `CustomAction.config`** (`<Content Include="CustomAction.config" />`)
  declaring `<supportedRuntime version="v4.0" ... />`. Without it the SfxCA shim fails with
  **0x80131700 "Failed to get requested CLR info"** and *every* managed action dies before
  reaching your code.
- Project shape: `net472`, `Platform=x64` (must match the package architecture for the shim),
  packages `WixToolset.Dtf.CustomAction` + `Microsoft.NETFramework.ReferenceAssemblies`. Use only
  GAC assemblies (`System.Data` for SqlClient, `System.Web.Extensions` for JSON) — zero deployed
  dependencies. The build output is `<Assembly>.CA.dll`; that's what `<Binary>` references.
- `session.GetTargetPath("SomeFolder")` throws **error 2727** unless that directory is actually
  referenced in the authoring (e.g., `System64Folder` vanishes from the Directory table when the
  last `[System64Folder]` formatted reference is removed). For system paths, use
  `Environment.GetFolderPath` in the CA instead.
- Deferred CAs read their command from a property with the **same name as the action**
  (CustomActionData pattern). Set it from an immediate action any time before the deferred action
  is reached in the immediate pass.

## 6. Custom wizard pages (inserting into a stock WixUI set)

- **You cannot insert a page by appending `Publish` rows with a high `Order`.** ControlEvents run
  in order, and the stock `NewDialog → VerifyReadyDlg` (Order 4) switches dialogs immediately —
  appended rows never execute. Symptom: your page silently never appears.
- The working approach: **clone the entire stock dialog-set navigation** (the `Publish` rows from
  `WixUI_InstallDir.wxs` in the wix repo) into your own `<UI>`, insert your page in the chain,
  drop `<ui:WixUI>` and keep `<UIRef Id="WixUI_Common" />` (which brings the stock dialogs, fonts
  and loc strings). Remember `DialogRef`s for ErrorDlg/FatalError/UserExit/Progress/etc.
- **`session.Message` from a DoAction inside a dialog's event loop is silently suppressed** — no
  message box will appear. Feedback pattern that works: the CA writes its result text into a
  property; the button publishes `DoAction` (Order 1) then `SpawnDialog` of a small result dialog
  (Order 2) whose `Text="[RESULT_PROPERTY]"` formats at creation. Gate `Next` on a property the
  CA sets (`CONNTEST_OK = "1"`), with an explicit "skip" checkbox as the operational escape hatch.
- **A DoAction from a dialog freezes the wizard** for its whole duration (UI thread; "Not
  Responding"). You cannot show progress. Set expectations in the page text ("can take up to 30
  seconds; the wizard will not respond"), and bound the action's worst case.
- A connection test should probe at **server level** (database may not exist yet), with a retry
  for spin-up-shaped failures — LocalDB / auto-closed SQL Express instances *start because of*
  the first probe and outlive a single 10s timeout.

## 7. Logging and diagnosis

- **`<Property Id="MsiLogging" Value="voicewarmupx" />`** makes *every* run (double-click,
  silent, upgrade, uninstall) write a verbose log to the invoking user's `%TEMP%\MSI*.LOG`
  automatically. Ship it always; it turns "it failed" into a readable trace.
- Reading a log: search `Return value 3` (the failing action), `returned actual error`,
  `WixQuietExec:` (your CA's stdout), and `MainEngineThread is returning`.
- `Note: 1: 2727 2: <name>` style lines decode via MSI error codes (2727 = directory not in
  Directory table; 2826 = control outside dialog bounds — the stock WixUI lines trigger 2826
  *benignly* in every debug log; ICE03 string-overflow on a long CustomAction Target is a
  build-time validation warning, not a runtime problem).

## 8. Headless testing techniques (no clicking required)

- **Layout check:** administrative extract — `msiexec /a pkg.msi /qn TARGETDIR=...` — then diff
  the file tree.
- **Table check:** `WindowsInstaller.Installer` COM → `OpenDatabase` + SQL. MSI SQL quirks: no
  `IN`, quote reserved words with backticks (`` `Key` ``, `` `Order` ``).
- **Custom-action check:** COM `OpenPackage(path, 0)` gives a real session; `DoAction("Name")`
  runs immediate CAs for real (option `1` opens a *restricted* engine — "not permitted in a
  restricted engine"). Combine with `Installer.EnableLog` to capture a full trace of just that
  action.
- **Wizard check:** UI Automation can drive the real wizard, with caveats — MSI controls degrade
  from typed controls to bare Panes after modal round-trips (patterns stop working; fall back to
  coordinate clicks + SendKeys), and spawned MSI dialogs are **child windows** of the wizard, not
  top-level windows.
- Each `dotnet build` of a wixproj with a different `OutputName`/version needs an `obj/` clean
  first, or MSB3030 "could not copy ... was not found".

## 9. Hosting gotcha discovered via the installer (but not an installer bug)

Blank SPA pages with browser console errors *"Expected a JavaScript module script but the server
responded with a MIME type of text/html"* = the SPA fallback served `index.html` for the chunk
files. In ASP.NET Core minimal hosting, **without an explicit `UseRouting()` call, routing runs
at the very start of the pipeline**, a `MapFallback` catch-all matches every request, and
`UseStaticFiles` deliberately skips endpoint-matched requests. Fix: register static files first,
then call `UseRouting()` explicitly, then auth and `Map*`.

## 10. Scheduled tasks (no first-class WiX element — the `schtasks` pattern)

MSI has a `ServiceInstall` element but **nothing** for scheduled tasks. To register an installed
binary to run on a schedule (e.g. a daily agent run as `SYSTEM`), ship a Task Scheduler **XML
definition** and register it with `schtasks.exe` from deferred custom actions. Prefer XML over
inline `schtasks` flags — only XML can express the two settings that matter for VMs that aren't
always on: missed-run catch-up and a fleet-spreading random delay.

Working examples: `installer reference/ScheduledTask.sample.wxs` (the WiX fragment) and
`installer reference/SurveyTask.sample.xml` (the task definition).

**The pattern**

1. **Author the task as XML** (Task Scheduler 1.2 schema). The settings that earn their keep:
   - `<Principal>` `<UserId>S-1-5-18</UserId>` + `LogonType=ServiceAccount` + `RunLevel=HighestAvailable`
     → runs as `SYSTEM`, elevated, **no stored password**, and its network identity is the machine
     account (which is what makes domain secure-channel / Kerberos checks possible).
   - `<StartWhenAvailable>true` → catches runs **missed while the VM was powered off**. Without it a
     VM that's down at the trigger time simply never reports that day.
   - `<RandomDelay>PT1H` on the trigger → spreads a fleet so every host doesn't hit the listener in
     the same second.
   - `DisallowStartIfOnBatteries=false` + `StopIfGoingOnBatteries=false` (VMs can report as "on
     battery"), `ExecutionTimeLimit`, `MultipleInstancesPolicy=IgnoreNew`.
2. **Resolve the exe path.** The `<Command>` isn't known until `INSTALLFOLDER` is chosen, so ship the
   XML with a token (`{{INSTALLFOLDER}}`) and have a managed immediate CA substitute it before
   registration (same shape as the config-write action in §4). If you always install to a fixed
   ProgramFiles path, hardcode `<Command>` and drop that action.
3. **Register**, idempotently: deferred, `Impersonate="no"` (runs elevated in the install's system
   context), `"[System64Folder]schtasks.exe" /create /tn "\OGA\HostSurvey" /xml "[INSTALLFOLDER]SurveyTask.xml" /f`.
   `/f` overwrites on repair/re-register.
4. **Deregister on uninstall:** `schtasks /delete /tn ... /f` with `Return="ignore"` (deleting a
   missing task returns non-zero).
5. **Rollback:** pair the create with an `Execute="rollback"` CA (also a `/delete`) sequenced
   *immediately before* the create, so a later install failure doesn't orphan the task.

**Sequencing — the same trap as services (§3).** Under `MajorUpgrade Schedule="afterInstallExecute"`,
`RemoveExistingProducts` runs **late**, so the *old* product's `schtasks /delete` fires after the new
install. Schedule the **create `After="RemoveExistingProducts"`** (with its rollback just before it),
exactly like `StartFdService` — otherwise the old product's uninstall deletes the task the new
install just created. Belt and suspenders: also guard the delete with `NOT UPGRADINGPRODUCTCODE` so a
newer package's authoring skips the delete during an upgrade in the first place.

**Gotchas**

- **Quoting:** `/tn`, `/xml`, and `/tr` values with spaces need embedded quotes inside the
  `WixQuietExec` command. Compose in a `SetProperty` and verify the formatted result in the verbose
  log (`WixQuietExec:` lines) — this is the #1 source of silent `schtasks` failures.
- **Deferred CAs can't read properties** — pass the composed command via the same-named property
  (CustomActionData, §5).
- `[System64Folder]schtasks.exe`: referencing it here keeps `System64Folder` in the Directory table
  (see the §5 2727/`System64Folder` note).
- **Elevation:** `RunLevel=HighestAvailable` (XML) or `/rl HIGHEST` (flags) — the task needs elevation
  to read the registry/MSI/secure-channel data.
- **The XML file must be UTF-16 *with a BOM*.** `schtasks /create /xml` reports *"The task XML is
  malformed"* if the file is UTF-16 without a BOM (the bytes don't match the `encoding="UTF-16"`
  declaration and it can't decode). Write it with the BOM — .NET `Encoding.Unicode` via
  `File.WriteAllText` does. (Both symptoms were hit for real: no-BOM → "malformed".)
- **Omit `<LogonType>` for a SYSTEM principal.** With `<UserId>S-1-5-18</UserId>`, including
  `<LogonType>ServiceAccount</LogonType>` fails validation — *"a value which is incorrectly formatted
  or out of range"* pointing at the LogonType line. Windows' own exported SYSTEM tasks omit it; just
  `UserId` + `RunLevel` is correct.
- **Verify headlessly:** `schtasks /query /tn "\OGA\HostSurvey" /xml` after a test install; the empty
  `\OGA\` scheduler folder left after `/delete` is harmless.

**Inline-flags quick alternative** (no XML, no substitution CA), with its limitation:
`schtasks /create /tn "\OGA\HostSurvey" /tr "\"[INSTALLFOLDER]OGA.HostSurvey.Agent.exe\"" /sc DAILY /st 02:00 /ru SYSTEM /rl HIGHEST /f`
— simplest, but **cannot** express `StartWhenAvailable` (missed-run catch-up) or `RandomDelay`. Fine
for always-on servers; not for VMs that are frequently powered off.
