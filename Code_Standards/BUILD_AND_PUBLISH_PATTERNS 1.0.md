# Build & Publish Patterns — 1.0

A cross-project standard for how an OGA Windows-service platform is **built, versioned, packaged, and
deployed**. It describes the *pattern*; the Facility Dashboard Platform is the reference implementation,
so concrete names (`Dashboards.Host`, `fd-dbtool`, `build-release.ps1`, …) are examples — keep the
**shape**, rename the parts. Companion docs: **WIX_INSTALLER_LESSONS** (deep WiX/MSI detail — do not
duplicate it here) and **PLATFORM_PATTERNS** (runtime/architecture patterns).

> Status: pattern distilled from Facility Dashboards (2026). Revise the version suffix when the pattern
> itself changes, and backport improvements learned downstream.

---

## 1. Shape of a release

One git repository produces one shippable artifact: a **single MSI** that installs a self-contained
Windows service plus its supporting tools and its web UI. The moving parts:

| Part | Reference example | Notes |
| --- | --- | --- |
| The service | `Dashboards.Host` → `Dashboards.Host.exe` | The long-running Windows service. |
| Support tools | `fd-recovery`, `fd-dbtool` | Self-contained single-file console exes shipped alongside. |
| Web UI | Angular apps (`primitives` lib, `display`, `authoring`) | Built to static bundles, staged into the service's `wwwroot`. |
| DB migration script | `create-or-migrate.sql` | Idempotent; applied at install/upgrade. |
| Installer | WiX MSI project + DTF custom-actions project | Packages the staged payload. |

The build assembles everything into a **staging layout**, then WiX packs staging into the MSI:

```
deploy/
  build-release.ps1        # the orchestrator (one entry point)
  version.ps1              # version authority CLI
  versions.json            # committed version state
  set-config.ps1           # runs at INSTALL time (MSI custom action), not build time
  installer/               # WiX project (*.wixproj, Package.wxs, License.rtf)
  installer-actions/       # DTF managed custom actions (C#)
  sql/create-or-migrate.sql
  staging/                 # BUILD OUTPUT (gitignored): service/, recovery/, dbtool/, sql/
  output/                  # BUILD OUTPUT (gitignored): FacilityDashboards-<version>.msi
```

Both `staging/` and `output/` are **gitignored** — they are build products, never committed.

---

## 2. Self-contained, single-file publish

The service and each tool are published **self-contained and single-file**, so the host needs no .NET
installed and each ships as essentially one executable:

```
dotnet publish <project> -c Release -r win-x64 --self-contained true `
    -p:PublishSingleFile=true -p:IncludeNativeLibrariesForSelfExtract=true -p:Version=<version> `
    -o <staging>/<component>
```

What each flag does — the names overlap in conversation, so be precise:

- **`PublishSingleFile=true`** — *single-file deployment*. Consolidates the **managed third-party DLLs**
  into the one executable; they load from inside the bundle (nothing is extracted for managed code).
- **`--self-contained true` (with `-r win-x64`)** — *self-contained deployment*. Also bundles the **.NET
  runtime**, so the target machine needs no .NET installed. (The opposite is *framework-dependent*.)
- **`IncludeNativeLibrariesForSelfExtract=true`** — packs **native** libraries into the bundle too; those
  *are* extracted to a temp dir at startup (native code cannot run from inside the bundle).
- **`-p:Version=<version>`** — stamps the apphost's **FileVersion** (see §4 — this matters for MSI upgrades).

Publish the service and every tool this way. The web UI is not a .NET publish — it is built separately
(§3) and copied into the service's `wwwroot`.

---

## 3. The orchestrator and the separable installer stage

A **single orchestrator script** drives the whole release, but its stages are **independently runnable**
via switches, so the installer build (and the web build) are conditional, not always-on:

```
build-release.ps1
  -Version <x.y.z>     # optional: when omitted, computed from the version authority (§4)
  -VersionPart Patch   # which component to bump when auto-deriving (Patch is the house default)
  -RecordVersion       # on SUCCESS, record the shipped version back to the authority (§4)
  -SkipWeb             # skip the Angular build (reuse whatever is already staged in dist/)
  -SkipMsi             # build + stage the payload only; do NOT build the installer
  -SignCertThumbprint  # optional Authenticode signing (§6)
```

Order of stages: resolve version → **clean staging** → publish service + tools (§2) → *(unless `-SkipWeb`)*
build the web apps (§3a) → stage web bundles + SQL + install-time config → *(if signing)* sign payload →
*(unless `-SkipMsi`)* build the custom-actions project, then the WiX project → *(if signing)* sign the CA
DLL then the MSI → *(if `-RecordVersion`)* record the version.

**Why the installer is its own stage:** packaging is slower and has a different toolchain (WiX) than
compiling the payload, and you frequently want to iterate on the payload without rebuilding the MSI
(`-SkipMsi`), or rebuild the MSI from an already-built payload. The installer is also a **separate project**
(the `.wixproj`) plus a **separate custom-actions project** — kept out of the main solution's normal build
so a routine `dotnet build` never drags in WiX.

**Standardize this separability.** For a CI pipeline (e.g. the planned Jenkins migration), split the stages
into discrete steps rather than one monolithic call:

1. **Version** — ask the authority for the next version.
2. **Payload build** — publish service/tools + web, stage.
3. **Installer build** — WiX pack of the staged payload.
4. **Record/publish** — on success, record the version and push the artifact to the store.

Each step is independently retryable and maps cleanly onto pipeline stages.

### 3a. Web build quirks (if the platform has a web UI)
- Build the component library first, then the apps (`ng build primitives`, then `display`, then `authoring`).
- The `display` app's `--base-href /display/` **must be built via PowerShell, not a POSIX/MSYS shell** —
  MSYS path-conversion mangles the leading-slash argument into a Windows path and corrupts `index.html`'s
  `<base href>`. Verify after building: `grep '<base href' dist/display/browser/index.html`.

---

## 4. Platform versioning (build-to-build)

**The platform version is manual and explicit — there is no auto-increment in the build itself** — but it
is tracked durably by a small **version authority** so it is consistent build to build and is never inferred
from fragile artifact filenames.

- **`versions.json`** — a **committed** file: `{ product, current, history[] }`. `current` is the source of
  truth for "the last version we shipped"; `history` is an append-only log (version, UTC, commit, note).
  Committed (unlike `output/`), so it survives a clean checkout and is visible in git.
- **`version.ps1`** — a tiny CLI over that file, designed to be sewn into pipeline steps:
  - `-Action Next [-Part Patch|Minor|Major]` — **read-only**; computes and prints the next version. Running
    it twice yields the same answer until a Commit happens (so a failed build never advances state).
  - `-Action Commit -Version x.y.z [-CommitSha .. -Note ..]` — records a shipped version (atomic temp-and-move
    write; **monotonic guard** refuses a version ≤ `current`). Call only on a successful build.
  - `-Action Current` — prints the last recorded version.
- **Flow:** `Next` → build with that `-Version` → on success `Commit`. `build-release.ps1 -RecordVersion`
  wires `Next`+`Commit` in for a local one-shot; a CI pipeline runs them as separate steps.
- The resolved version flows to **two places**: `-p:Version` → each apphost's **FileVersion**, and
  `-p:ProductVersion` → the **MSI ProductVersion**. Keep them equal.
- **Convention:** bump the **patch** each release (monotonic), so update churn is legible. The `Version`
  property in `Directory.Build.props` stays at a placeholder (`1.0.0`) and is **not** the source of truth.

**Against a real artifact store (Artifactory, etc.):** `versions.json` is a local stand-in. Replace it by
querying the store for the latest version (the `Next` step) and publishing the artifact (the `Commit` step);
the `Next → build → Commit` shape is identical.

### Two gotchas to standardize around
- **MSI major-upgrade needs an increasing FileVersion.** An MSI replaces a *versioned* file on upgrade only
  when the incoming FileVersion is **higher**. Because `-p:Version` stamps the apphost and no csproj pins
  `<FileVersion>`, each release's exe gets a distinct version — verify it after a build:
  `(Get-Item …\service\<Host>.exe).VersionInfo.FileVersion`, not just the MSI's ProductVersion.
- **Each local build deletes the previous locally-built MSI** from `output/` (MSBuild/WiX tracks and removes
  its prior tracked output). That is fine — `output/` is gitignored, every version is rebuildable from its
  commit, and the policy is to **keep the last couple of installers on the deploy target** for rollback, not
  on the build box. (In CI, the artifact store is the retention mechanism, so this is moot.)

---

## 5. Schema versioning (independent of the platform version)

Database schema has its **own integer version**, entirely separate from the `x.y.z` platform version. The
binaries declare the schema they require; the service **refuses to start against an out-of-date database**.

- **`SchemaInfo.CurrentVersion`** — a compiled `const int` in the domain assembly: the schema these binaries
  require. The service verifies it at startup (refuse-on-mismatch), the recovery tool verifies it before
  writing, and migrations advance it.
- **`SchemaVersions` table** — one row per applied schema version (version, applied-at, description). The
  running DB's max row is its current schema version.
- **Idempotent migration script** — `create-or-migrate.sql`, generated with
  `dotnet ef migrations script --idempotent`, staged into the MSI and applied at install/upgrade (or by the
  DB tool). Safe to run on a fresh *or* existing DB.

### Schema-change recipe (follow every time)
1. Edit the entity + DbContext config.
2. Add the EF migration (`dotnet ef migrations add <Name>`).
3. **Hand-add** the `SchemaVersions` row: `InsertData` in `Up()` and `DeleteData` in `Down()` — EF does not
   write it. Mirror an existing migration exactly.
4. **Bump `SchemaInfo.CurrentVersion`.**
5. Regenerate the idempotent script to the staged SQL path.

**Deploy consequence:** because the service refuses to start against an older DB, a release carrying a schema
bump **must migrate the live DB as part of the upgrade** (the installer/DB-tool does this). A plain release
with no schema change is a straight binary upgrade. State this in release notes either way.

---

## 6. Packaging & signing

- **Installer:** WiX (SDK-style `.wixproj`) + a DTF **managed custom-actions** project. See
  **WIX_INSTALLER_LESSONS** for the hard-won specifics (service install/start, major-upgrade scheduling,
  CA packaging, log hygiene, etc.) — not repeated here.
- **Upgrade model:** WiX **MajorUpgrade**; relies on the increasing FileVersion (§4).
- **Signing (Authenticode):** optional via `-SignCertThumbprint`. When signing, sign **(a)** the payload
  exes, **(b)** the custom-action DLL *before* WiX embeds it, and **(c)** the finished MSI — and sign **only
  the MSI this run built**, never a `*.msi` glob (a glob re-signs older artifacts, and MSI signatures cannot
  be stripped). Unsigned MSIs and temp-extracted CA DLLs **trip AV/XDR heuristics** (Defender PShellDown-class,
  Cortex XDR) on hardened hosts — string hygiene alone is not enough. Ship **unsigned** only where the target
  accepts it; sign for hardened fleets.

---

## 7. Toolchain & environment (pin these)

- **.NET SDK** pinned in `global.json` (the reference uses 8.0.x with `rollForward: latestFeature`) so a
  machine with a newer SDK doesn't emit a newer `TargetFramework`/project format.
- **PowerShell 7 (`pwsh`), not Windows PowerShell 5.1.** The release script sets
  `$ErrorActionPreference = 'Stop'`, and under 5.1 any native-command **stderr** write becomes a *terminating*
  error — which kills the Angular build step on its normal progress output. Run the script under pwsh 7. (If
  only 5.1 is available: build the web apps first, then `build-release.ps1 -SkipWeb`.)
- **Node/Angular** for the web build; **WiX 7** for the installer (pulled via the `.wixproj` PackageReferences).
- A **C# LangVersion / TFM pin** per portfolio policy if the platform standardizes one.

---

## 8. Deploy model

- Install/upgrade by running the **MSI** on the target (a WiX major-upgrade over the installed build).
- If the release bumps the schema, the **DB migrates during install** (or run the DB tool) — the service
  will not start otherwise.
- Keep the **last couple of installer versions on the target** for rollback.
- Back up the database before a schema-bearing upgrade.

---

## 9. Checklist for cutting a release

1. Clean working tree; everything committed.
2. (Web) build library → display (PowerShell for base-href) → authoring.
3. `build-release.ps1 [-SkipWeb] -RecordVersion` (version auto-derived, or pass `-Version`).
4. Verify: MSI present in `output/`; apphost **FileVersion** == release version; `version.ps1 -Action Current`
   advanced; staged SQL contains the new migration if schema changed.
5. Commit `versions.json`.
6. Release notes: note whether it carries a **schema migration** (DB migrates on deploy) or is a plain upgrade.

---

## Appendix A — `version.ps1` (canonical seed; copy nearly verbatim)

The version authority is **project-agnostic** — the only per-project change is `versions.json`'s `product`
string (§Appendix B) and the `.SYNOPSIS` wording. Copy it as-is into a new project's `deploy/`. This block
is maintained as a **verbatim copy of the live `deploy/version.ps1`** — keep the two in sync (it is small
and changes rarely, so drift risk is low).

```powershell
<#
.SYNOPSIS
    Release-version authority for the platform.
.DESCRIPTION
    Abstracts "what version comes next" and "record that we shipped one" behind a small
    script backed by a committed state document (deploy/versions.json), instead of inferring
    the last version from artifact filenames in deploy/output (which is gitignored and whose
    MSIs the build sometimes deletes). This mirrors how a real artifact store works
    (query latest -> build -> publish/record) and is designed to be sewn into CI as discrete steps.

    Robust against a build that fails partway:
      * -Action Next is READ-ONLY - it never advances state, so re-running after a failure
        returns the same next version.
      * -Action Commit is called ONLY after a successful build, and writes atomically
        (temp file + move), so a crash mid-write cannot corrupt the document.
      * Commit enforces monotonicity (refuses a version <= current).
.EXAMPLE
    # Pipeline (three discrete steps):
    $v = ./deploy/version.ps1 -Action Next            # e.g. 1.0.72   (no state change)
    ./deploy/build-release.ps1 -Version $v            # build; may fail -> state untouched
    ./deploy/version.ps1 -Action Commit -Version $v -CommitSha (git rev-parse --short HEAD)
.NOTES
    Runs under Windows PowerShell 5.1 and PowerShell 7+.
#>
[CmdletBinding()]
param(
    [ValidateSet('Next', 'Commit', 'Current')]
    [string]$Action = 'Next',
    # Which component to bump for -Action Next. Patch is the house convention (churn visibility).
    [ValidateSet('Patch', 'Minor', 'Major')]
    [string]$Part = 'Patch',
    # Required for -Action Commit: the version that was just built successfully.
    [string]$Version,
    # Optional metadata recorded with a Commit.
    [string]$CommitSha,
    [string]$Note,
    # The state document. Defaults to versions.json beside this script.
    [string]$StateFile = (Join-Path $PSScriptRoot 'versions.json')
)

$ErrorActionPreference = 'Stop'

function Assert-SemVer([string]$v) {
    if ([string]::IsNullOrWhiteSpace($v) -or $v -notmatch '^\d+\.\d+\.\d+$') {
        throw "Not a MAJOR.MINOR.PATCH version: '$v'"
    }
}

function Get-Parts([string]$v) { return ,@($v.Split('.') | ForEach-Object { [int]$_ }) }

function Compare-SemVer([string]$a, [string]$b) {
    $pa = Get-Parts $a; $pb = Get-Parts $b
    for ($i = 0; $i -lt 3; $i++) { if ($pa[$i] -ne $pb[$i]) { return $pa[$i] - $pb[$i] } }
    return 0
}

function Get-NextVersion([string]$current, [string]$part) {
    Assert-SemVer $current
    $p = Get-Parts $current
    switch ($part) {
        'Major' { $p[0]++; $p[1] = 0; $p[2] = 0 }
        'Minor' { $p[1]++; $p[2] = 0 }
        'Patch' { $p[2]++ }
    }
    return "$($p[0]).$($p[1]).$($p[2])"
}

function Read-State {
    if (-not (Test-Path -LiteralPath $StateFile)) {
        throw "Version state file not found: $StateFile. Create it (see deploy/versions.json) before building."
    }
    $state = Get-Content -LiteralPath $StateFile -Raw | ConvertFrom-Json
    if (-not $state.current) { throw "State file has no 'current' version: $StateFile" }
    Assert-SemVer $state.current
    return $state
}

function Write-JsonAtomic($obj, [string]$path) {
    # BOM-less UTF-8 so git/CI/other tools read it cleanly; temp+move so a crash mid-write
    # leaves the original intact.
    $json = $obj | ConvertTo-Json -Depth 8
    $tmp  = "$path.tmp"
    [System.IO.File]::WriteAllText($tmp, $json, (New-Object System.Text.UTF8Encoding($false)))
    Move-Item -LiteralPath $tmp -Destination $path -Force
}

$state = Read-State

switch ($Action) {
    'Current' {
        Write-Output $state.current
    }
    'Next' {
        Write-Output (Get-NextVersion $state.current $Part)
    }
    'Commit' {
        Assert-SemVer $Version
        if ((Compare-SemVer $Version $state.current) -le 0) {
            throw "Refusing to record '$Version': it is not greater than current '$($state.current)'. Release versions must be monotonic."
        }

        $entry = [ordered]@{
            version = $Version
            utc     = (Get-Date).ToUniversalTime().ToString('yyyy-MM-ddTHH:mm:ssZ')
        }
        if ($CommitSha) { $entry['commit'] = $CommitSha }
        if ($Note)      { $entry['note']   = $Note }

        $history = @()
        if ($state.history) { $history = @($state.history) }
        $history += [pscustomobject]$entry

        $new = [ordered]@{
            product = $state.product
            current = $Version
            history = $history
        }

        Write-JsonAtomic $new $StateFile
        Write-Output $Version
    }
}
```

## Appendix B — `versions.json` (committed authority; seed shape)

Create it once per project, committed. `current` is the source of truth; `history` grows one entry per
recorded release.

```json
{
  "product": "FacilityDashboards",
  "current": "1.0.0",
  "history": []
}
```

`version.ps1 -Action Commit` advances `current` and appends `{ version, utc, commit?, note? }` to `history`.

## Appendix C — `build-release.ps1` (skeleton + fragments worth copying)

The orchestrator is **project-specific** (component names, staging paths) — adapt it rather than copy it
whole. The shape:

```powershell
param(
  [string]$Version, [ValidateSet('Patch','Minor','Major')][string]$VersionPart = 'Patch',
  [switch]$RecordVersion, [switch]$SkipWeb, [switch]$SkipMsi,
  [string]$SignCertThumbprint, [string]$TimestampUrl = 'http://timestamp.digicert.com'
)
$ErrorActionPreference = 'Stop'
# 1. Resolve the version from the authority (Appendix C.1)
# 2. Clean staging/, recreate staging/ + output/
# 3. dotnet publish service + each tool  (self-contained single-file - see §2), stamping -p:Version=$Version
# 4. unless -SkipWeb: build the web apps (library first), copy bundles into service/wwwroot
# 5. stage the migration SQL + the install-time config script
# 6. if -SignCertThumbprint: sign the staged payload exes
# 7. unless -SkipMsi: build the custom-actions project, then the WiX project (-p:ProductVersion=$Version)
# 8. if signing: sign the CA DLL BEFORE WiX embeds it, then sign the finished MSI (only this run's MSI)
# 9. if -RecordVersion (ignored under -SkipMsi): record the shipped version (Appendix C.2)
```

### C.1 — resolve the version (fragment)
When `-Version` is omitted, derive it from the authority; otherwise honor the explicit value.

```powershell
$versionTool = Join-Path $PSScriptRoot 'version.ps1'
if ([string]::IsNullOrWhiteSpace($Version)) {
    $Version = (& $versionTool -Action Next -Part $VersionPart).Trim()
}
```

### C.2 — record on success (fragment) — mind the splat
Reached only after a fully successful build (so a partway failure never advances state). **Use a hashtable
splat**, not an array splat:

```powershell
$sha = $null
try {
    $raw = & git -C $root rev-parse --short HEAD
    if ($LASTEXITCODE -eq 0 -and $raw) { $sha = "$raw".Trim() }
} catch { $sha = $null }

# Hashtable splat so these bind as NAMED parameters. An ARRAY splat passes them positionally,
# which makes '-Action' a positional VALUE and trips version.ps1's ValidateSet (a bug we hit the
# first time -RecordVersion ran).
$commitArgs = @{ Action = 'Commit'; Version = $Version }
if ($sha) { $commitArgs['CommitSha'] = $sha }
& $versionTool @commitArgs | Out-Null
```

The self-contained single-file **publish command** itself is in §2 — copy that fragment for step 3.
