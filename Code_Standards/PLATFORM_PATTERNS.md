# Platform Patterns — Vertical Capability Blueprints

**Purpose.** The companion to `WEB_UI_FINISH_GUIDE.md`. Where the finish guide is the *horizontal*
lens — UI principles you **review any screen against** — this is the *vertical* lens: complete
capabilities described **end-to-end (client ↔ API ↔ service ↔ data)** that you **build from**. Each
pattern is grounded in the Facility Dashboard Platform (FDP) with file citations, but written so the
*shape* transfers to the next project.

**Stack.** Angular 18 (standalone components) · ASP.NET Core on .NET 8 · SignalR · EF Core 8 +
SQL Server · Windows service host. A different stack reuses the **shape and the contracts**; the code
is the reference implementation, not the requirement. (The finish guide, by contrast, is reusable
almost verbatim — it's about the medium, not the stack.)

**How to read a pattern.** Each section is the same five beats:

1. **Problem** — the user-or-operator pain it removes.
2. **Shape** — the pieces and how they connect, top to bottom.
3. **Contracts** — the few names/shapes another implementation must keep.
4. **Implementation (FDP)** — where it lives, with `file:line` citations.
5. **To reuse / pitfalls** — what to carry forward and the traps already paid for.

…and a one-line tie back to the finish-guide rule(s) it satisfies on the UI side.

**Self-contained handoff.** You do **not** need the FDP repo to build from this. The `file:line`
citations are *illustration* — "here's one that works" — not required reading; a pattern's **Shape**
and **Contracts** beats are the actual design, and **Implementation (FDP)** just points at where a
working copy lives. Carry the shape and the contracts forward; fork the code per project.

**Contents**

1. Live entity push
2. Connectivity — detect / show / self-heal
3. Security & authorization (+ pluggable providers)
4. Realtime connected-clients admin
5. Diagnostics & log viewing
6. Draft → publish & the config cascade
7. Render & transform pipeline
8. Data ingestion — scheduled producers
9. Versioned assets & telemetry-driven GC
10. Data-edge resilience
11. Operational backbone

Appendix — Feature patterns: broadcasting · backup/restore · export/import · station pairing

---

## 1. Live entity push

**Problem.** Every management screen must reflect mutations — made by anyone, anywhere — **without a
manual Refresh** and without polling. Two admins editing, a station coming online, a problem clearing:
the lists should just update.

**Shape.**
- **Server broadcasts a coalesced change signal.** At *every mutation site*, the controller/service
  calls `Notify("&lt;channel&gt;")` — `"dashboards"`, `"stations"`, `"problems"`, `"admin-sessions"`,
  `"tasks"`, `"sources"`, … A central broadcaster **coalesces per-channel** over a short window (so a
  burst of edits is one push) and sends a single `ModelChanged { area }` over SignalR to a group all
  management clients have joined.
- **Client turns a channel into an observable.** A live service connects to `/hubs/live`, joins the
  group, and exposes `of(channel): Observable&lt;void&gt;`. A page subscribes and refetches:
  `liveChanges.of('dashboards').pipe(takeUntilDestroyed(...)).subscribe(() => this.load())`. That's the
  whole contract — the event carries only the channel name; the client re-pulls the truth.
- **Selection survives the refresh.** Lists hold the selected row **by id** and re-resolve it on every
  reload, so a live update never loses or mis-aims the selection (and if another admin deleted it, the
  toolbar grays instead of pointing at a ghost).
- **Push, not fetch-on-interaction.** Where a control would otherwise GET on every open (a picker's
  list), subscribe to the relevant channels and refetch on the push instead — see the area picker.

**Contracts.**
- One event shape: `ModelChanged { area: string }`; a fixed set of **named channels**.
- Client `of(area)` ⇄ server `Notify(area)` — and `Notify` must fire at **every** write that changes
  what a channel represents.
- A **coalescing window** server-side so N rapid writes are one push.

**Implementation (FDP).**
- `src/Dashboards.Host/Services/ChangeBroadcaster.cs` — `Notify(area)`, the per-area coalescing
  (~1.5 s), and the `SendAsync("ModelChanged", new { area })` to the `admin-updates` group.
- `src/Dashboards.Host/Hubs/LiveHub.cs` — `JoinAdminUpdates()` (a management client opts in).
- `src/web/projects/authoring/src/app/services/changes.service.ts` — the SignalR connection and
  `of(area)`.
- Mutation sites call it: e.g. `TasksController` (`_changes.Notify("tasks"); _changes.Notify("sources")`),
  `BroadcastService`, `OpsController`, `AdminSessionRegistry`.
- Push-not-poll refinement: `src/web/projects/authoring/src/app/components/area-picker.component.ts`
  (merge of five channels, debounced, refetch on push).

**To reuse / pitfalls.**
- **Miss one `Notify` and that surface goes stale** — the discipline is "every write notifies its
  channel." Treat it like raising an event, not an afterthought.
- **Coalesce**, or a bulk operation becomes a push storm.
- **Drafts deliberately don't notify** — a private editing session shouldn't spam the facility; only
  the publish does. (This is why a draft-only area doesn't push; see the finish guide's input
  rules and the area picker.)
- The event is **just the channel** — never ship the payload in the event; let the client refetch, so
  authorization and shaping stay on the server.

*Finish-guide tie:* §5 (live data refreshes itself), §7 (no manual Refresh on a self-refreshing list;
selection held by id).

---

## 2. Connectivity — detect / show / self-heal

**Problem.** When the client loses the backend — a stopped service, a dropped link, a firewall hiccup —
the UI must **say so**, not read as "the buttons don't work." And after a server **upgrade**, stale
clients must not silently run old code against a new API.

**Shape.**
- **One connection state, three surfaces.** A single client signal — `connecting | online |
  reconnecting` — drives an always-on **status dot** (green/amber/red + label) and, the moment the link
  drops, a **"service unreachable" banner** with a *Retry now* button, auto-cleared on recovery.
- **Detect on three independent channels**, so neither a transport quirk nor an idle page hides an
  outage:
  1. **The live socket's own lifecycle** — `onreconnecting → lost`, `onreconnected → online`,
     `onclose → lost` — with **reconnect-forever, capped backoff** (e.g. 2 s for the first few tries,
     then 10 s; *never* a fixed retry count that gives up).
  2. **Every HTTP response** via an interceptor: status `0` (no response) = transport down → `lost`;
     any actual response (even 4xx/5xx) = the link is fine → `online`.
  3. **A small periodic heartbeat** — an ~8 s `GET /api/v1/me` — which **bounds worst-case detection
     latency** independent of socket timings.
- **Version self-heal.** Every response carries a build-stamp header (`X-FD-Version`). The client
  **baselines the first** value and, on any later mismatch, shows a *"server was updated — reload"*
  banner; reloading re-baselines. (Admins can also remote-reload outdated UIs — §4.)
- **Durable server liveness.** A `HeartbeatService` writes a heartbeat row (version, started-at) every
  ~30 s, which the offline recovery tool reads to tell "hung" from "stopped."

**Contracts.**
- A connection-state enum + the dot/banner surface.
- Multi-channel detection (socket + HTTP + heartbeat); **reconnect forever** with capped backoff.
- A version header + first-response baseline + mismatch banner.
- A durable liveness record for out-of-band tooling.

**Implementation (FDP).**
- `src/web/projects/authoring/src/app/services/connection-status.service.ts` (the `connecting/online/
  reconnecting` signal), `changes.service.ts` (`withAutomaticReconnect` + the socket lifecycle hooks),
  `version-watch.service.ts` (the HTTP interceptor + `X-FD-Version` baseline/mismatch).
- `app.component.ts` — the status dot, the unreachable + update banners, and the ~8 s heartbeat to
  `/api/v1/me`.
- Server: `Program.cs` stamps every response with `X-FD-Version`; `HeartbeatService` (in
  `Services/BackupService.cs`) writes the liveness row.

**To reuse / pitfalls.**
- **Reconnect forever.** A fixed retry count strands the UI after a long outage — the exact moment
  recovery matters most.
- **Multiple detection channels** — the socket alone misses a half-open connection; HTTP alone misses
  an idle page; the heartbeat bounds the worst case.
- **The dead-lazy-route trap** (finish §2): a code-split SPA caches a *failed* route import as
  permanently rejected, so a click made during an outage stays dead after recovery. Preload route code
  up front and, on a code-load failure, probe and hard-reload when the backend answers.
- **Stamp the version on the response, not a separate poll** — you get upgrade detection for free on
  traffic you're already making.

*Finish-guide tie:* §2 (no dead controls; outage-proof navigation), §6 (failure is a UI state;
connectivity shown, never silent), §5 (status as color + word).

---

## 3. Security & authorization (+ pluggable providers)

**Problem.** Authenticate against the corporate domain, but **authorize on the platform's own roles** —
OS/AD membership (including local Administrators and Domain Admins) must grant **nothing**. Seed the
first administrator, recover when everyone's locked out, and keep integrations (mail, TLS, render
primitives) **pluggable**.

**Shape.**
- **AuthN: Windows/Negotiate** (Kerberos/NTLM) → an identity (`DOMAIN\user`). No application login
  screen; the OS already proved who you are.
- **AuthZ: the platform's own permission groups**, stored in the database, *not* AD. Named **policies**
  (`Admin`, `Developer`, `TaskManagement`, `Media`, `Broadcasters`, and `AnyManagement`) are backed by
  a single requirement handler that resolves the caller's groups from the DB (short-TTL cached) and
  succeeds if they hold any required group. **Windows/AD membership maps to no platform group by
  design** — elevation and "Run as administrator" change nothing.
- **Bootstrap + break-glass.** A config list (`BootstrapAdmins`) ensures an Admin membership for named
  identities at every startup. A **separate, elevation-gated recovery tool** runs directly against the
  database (works when the service is down) to restore group membership or fix TLS — elevation *is* the
  access gate (only local admins get an elevated token).
- **Gray, don't hide.** The nav shows pages a user's groups don't unlock **grayed with a lock + a
  why-tooltip** ("needs the Developer or Media group; an admin can grant it on the Users page"), never
  hidden — a control that comes and goes teaches the UI is unstable.
- **Live security invalidation.** When an admin changes a user's groups, the server pushes
  `SecurityChanged` to that user's open connections so their UI re-evaluates immediately.
- **Pluggable providers** (the extensibility seams):
  - **Notifications** — an `INotificationProvider` interface with SMTP + always-on Log implementations,
    fanned out by a dispatcher that **throttles per event-key** (a flapping station can't spam).
  - **TLS/certificate** — a service with clear **precedence** (admin-selected → deploy-time default →
    self-signed bootstrap persisted with DPAPI) and **hot-reload** via Kestrel's per-handshake selector
    (no dropped connections).
  - **Render primitives** — a manifest-described **catalog**; a new primitive is a manifest + a render
    component, with no change to the dashboard/widget/runtime core.

**Contracts.**
- AuthN scheme = OS/Negotiate; AuthZ = a **group enum + named policies + a handler reading DB
  membership** (with a short cache so grants/revokes apply fast).
- Bootstrap seeding from config; an **out-of-band, elevation-gated** recovery path.
- Provider **interfaces + DI registration**; "list of providers, dispatch to all."

**Implementation (FDP).**
- `src/Dashboards.Host/Program.cs` — `AddNegotiate()`; `AddPolicy(Policies.*, … new GroupRequirement(…))`;
  the `BootstrapAdmins` seeding; the `X-FD-Version` stamp.
- `src/Dashboards.Host/Security/GroupAuthorization.cs` (`GroupRequirementHandler`),
  `Security/UserDirectory.cs` (`GetGroupsAsync` + ~10 s cache), `Dashboards.Domain/Enums.cs`
  (`PermissionGroupKind`).
- `src/Dashboards.RecoveryTool/` (`RecoveryApp.cs`) — elevation gate + DB-direct membership/TLS repair.
- Nav lockout: `app.component.ts` (`canAccess` / `lockTitle`).
- Providers: `Dashboards.Domain/Contracts/NotificationContracts.cs` (`INotificationProvider`),
  `Services/NotificationDispatcher.cs` (SMTP/Log + per-key throttle), `Services/CertificateService.cs`
  (precedence + hot-reload + DPAPI bootstrap), `Dashboards.Domain/Contracts/PrimitiveCatalog.cs`.

**To reuse / pitfalls.**
- **Authorize on your roles, not the OS's.** That AD/local-admin grants nothing is a *deliberate,
  must-document surprise* — the #1 "I'm an admin but every page says no privileges" support call. Say
  it loudly in the guide.
- **Short cache TTL** on group resolution so a recovery-tool grant takes effect in seconds, not on
  restart.
- **The break-glass tool is separate from the service and gated by elevation** — it must work when the
  service won't start, and it must not be reachable through normal UI.
- **Throttle notifications by key**; **hot-reload TLS** without dropping the listener; keep the
  primitive catalog **additive** (new manifest, no core edits).

*Finish-guide tie:* §2 (guard-rails — refuse to remove the last admin, with a human reason), §7
(gray-don't-hide — a control that exists but is unavailable always explains itself).

---

## 4. Realtime connected-clients admin

**Problem.** An operator needs to see, **live and from one screen**, which displays and which
management UIs are connected right now, what each is showing, and to **act on them remotely** — assign
a dashboard, reload, flash an identify overlay, decommission a screen; reload or disconnect a stale
admin UI — without walking the building.

**Shape.**
- **Two in-memory registries of *current* connections** (distinct from the durable DB records): one
  for **displays** (by connection id / token / station id, plus identify flags) and one for **admin
  UIs** (user, current page, last-activity, UI version). Each registry **`Notify`s its channel** on
  change, so the Stations and Admin Sessions pages **self-refresh** (vertical §1).
- **Telemetry in, commands out, over the hub.** Displays report what dashboard/version they're
  presenting (and any render errors → problems, §5). A **`StationMonitor` sweep** reconciles each
  station's state — *Connected* (in the live directory) vs *Transient* (seen within the threshold) vs
  *Disconnected* (past it) — raising/clearing offline problems and notifying. Remote actions are hub
  messages pushed to the **target connection**: reload, identify overlay, admin-reload/disconnect;
  assignment rebinds the station's hub group and pushes fresh state.
- **Don't-target-self guard.** An admin can't reload/disconnect the very session they're working in.
- **Restart-safe by construction.** The registries are ephemeral: a service restart clears them, every
  client reconnects and re-registers, and transient flags (identify overlays) reset — *no stuck state*.

**Contracts.**
- An **in-memory connection directory** keyed by connection id (and entity id/token), separate from the
  persistent entity row.
- Hub methods for **telemetry-in** and **command-out**; a periodic **reconcile** that derives
  connection state from the directory + last-seen.
- A **self-target guard** on destructive remote actions.

**Implementation (FDP).**
- Displays: `src/Dashboards.Host/Services/StationDirectory.cs` (the live set; `IsStationConnected`,
  `ConnectionsByToken`), `StationService.cs` (register/assign/identify/decommission + rebind +
  push-state) and its `StationMonitor` sweep; `Hubs/LiveHub.cs` `ReportTelemetry`. UI:
  `pages/stations.page.ts`.
- Admin UIs: `Services/AdminSessionRegistry.cs` (register/update/`ConnectionsForUser`; `Notify
  ("admin-sessions")`), the admin-session command endpoints (with the self-target guard). UI:
  `pages/admin-sessions.page.ts`.

**To reuse / pitfalls.**
- **"Who's connected *now*" is in-memory; the durable record is in the DB.** Don't try to persist the
  live set — restart-clears-it is the *correct* behavior, and clients re-register on reconnect.
- **Reconcile against last-seen** to separate a brief blip (Transient) from a real outage
  (Disconnected) — and notify only once per outage (a notified-at marker), throttled.
- **Guard remote actions against the caller's own session**, and make identify/diagnostic overlays
  **fail safe** (clear on restart) so a screen never gets stuck wearing one.

*Finish-guide tie:* §5 (live status as color + word; the UI shows only what it knows), §2 (destructive
remote actions — decommission/disconnect — confirm with consequences).

---

## 5. Diagnostics & log viewing

**Problem.** When something goes wrong, an operator needs the **"why"** (a searchable log) and a
**consolidated, self-clearing list of what's currently broken** — both from the admin UI, without
remoting into the host to tail a file.

**Shape.**
- **Structured diagnostics, dual-written.** Every entry goes to *both* the database (queryable: level,
  category, time, user) *and* the standard log sink (file + Windows Event Log). A **Diagnostics page**
  filters by level / category / trailing-window with an **auto-refresh toggle** — push isn't wired for a
  rolling log, so reading and following are an explicit choice (finish §5).
- **The Problems overview is a keyed upsert store.** `RaiseAsync(kind, key, title, detail)` and
  `ClearAsync(key)`: the **stable key** is the identity — the same condition updates one record
  (occurrence count + last-seen) instead of spawning duplicates, and **clears itself** when the
  condition resolves. Low-priority entries dim rather than shout. The page subscribes to `"problems"`
  (self-refresh, §1); admins resolve.
- **Two feeders, proactive + reactive.** Proactive: a hosted sweep renders the dashboards actually live
  on stations and raises/clears a problem per failing widget — catching **server-side** resolution
  failures (unbound/missing/unreadable source) that a client never reports. Reactive: stations report
  their own **client-side** render errors via telemetry. Together they cover both blind spots.
- **In-editor diagnostics** close the loop: the dashboard editor's *Check render* runs one widget
  through the real pipeline on demand and shows its resolved error / bound source / view-model.

**Contracts.**
- A **dual-write logger** (DB for the UI, sink for ops) with category + level + time.
- A **keyed raise/clear** problem store with occurrence-count upsert and a resolved state.
- **Proactive + reactive** feeders into the same store.

**Implementation (FDP).**
- `src/Dashboards.Host/Services/DiagnosticsService.cs` — `LogAsync` (DB + Serilog dual-write,
  truncation, fail-soft) and `ProblemService` (`RaiseAsync`/`ClearAsync` keyed upsert, `Notify
  ("problems")`).
- `Services/RenderProblemMonitor.cs` — the proactive sweep (live dashboards → per-widget raise/clear).
- `Hubs/LiveHub.cs` — the reactive `ReportTelemetry` render-error path.
- `Controllers/OpsController.cs` — the `diagnostics` (level/category/since/take) and `problems`
  (+ resolve) endpoints. UI: `pages/diagnostics.page.ts`, `pages/problems.page.ts`; the editor's
  *Check render* in `pages/dashboard-editor.page.ts`.

**To reuse / pitfalls.**
- **Key problems by stable identity** so one condition is one row that clears cleanly — the same
  discipline as the *Operational backbone* sweeps. Get the key wrong and the list fills with near-duplicates that never
  resolve.
- **Proactive *and* reactive** — server-side failures the client can't see, plus client-only render
  errors the server can't see; either alone has a blind spot.
- **Dual-write the log** so a DB hiccup doesn't lose the operator's trail (and vice-versa); truncate
  long messages.
- **Auto-refresh only where push isn't wired** (a rolling log); everywhere push exists, self-refresh.

*Finish-guide tie:* §5 (auto-refresh toggle where push is absent; status as color + word), §6 (failure
is a UI state, surfaced — not a silent gap).

---

## 6. Draft → publish & the config cascade

**Problem.** Configuration that drives live screens must be **editable by several people without
clobbering**, **safe to try before it goes live**, and able to **inherit deployment defaults** while
allowing per-entity overrides — and a later change to a default must not silently move things already
published.

**Shape.**
- Each versioned entity is a **head row + an immutable version chain**. Consumers either **follow
  latest** (late binding — a publish reaches them automatically) or **pin** a specific version.
- Editing never touches a published version. It happens in a **draft**: a working-model JSON, a
  **change journal** (every edit is an event — undo/redo walk it; rollback resets to the base
  version), and a **single-editor lock** with periodic liveness so a buried browser tab can't
  silently hold or overwrite.
- **Publish** validates, snapshots the draft into a *new immutable version*, **squashes** the journal,
  releases the lock, and pushes a live update to consumers following latest.
- **Config resolves global → entity → version.** A value either inherits the global default or is
  overridden; the UI shows an **Override** toggle and the *effective inherited value* (never a blank
  box). **Structural** values (those a layout was validated against — e.g. a border that consumes
  space) are **snapshotted at publish** so a later default change can't move a published layout;
  **presentation** values (colors, fonts) **late-bind** through the theme.

**Contracts.**
- A draft = `{ EntityKind, EntityId, BaseVersionId?, ModelJson, ChangeJournal[], AppliedEventCount,
  LockedBy?/LockSessionId?/LockLastLivenessUtc? }`.
- Version numbers are **monotonic and forward-only** per entity.
- The publish merge records **author intent** alongside the resolved number (an `inherited` marker),
  so a re-draft of an inherited value resumes inheriting rather than freezing to whatever the default
  happened to be at publish.

**Implementation (FDP).**
- `src/Dashboards.Host/Services/VersioningService.cs` — the engine: `CreateDraftAsync` /
  `ResolveBaseModelAsync` (build the draft model from a version, carrying head fields like name &
  area), `AcquireLockAsync` / `ReportLivenessAsync` / `ReleaseLockAsync` (single-editor lock, FR-62..66),
  `UndoAsync` / `RedoAsync` / `RollbackToBaseAsync`, `PublishDraftAsync` (validate → new version →
  squash journal → swap live), and the config-snapshot helpers `MergeDashboardOverrides` /
  `ReadBorderInherited` (the structural-vs-presentation + inherited-marker logic).
- `src/Dashboards.Host/Controllers/DraftsController.cs` — REST surface; note the **lock guard** on
  destructive ops (`Discard` refuses when another user holds the lock, `DraftsController.cs:263`).
- Client: `src/web/projects/authoring/src/app/services/draft-session.ts` (`DraftSession` — open/lock/
  liveness/commit/undo/redo/publish) and the editor pages, which **commit on settle** and show
  read-only state when displaced.
- Config DTO with the override fields: `src/Dashboards.Dtos/DraftModels.cs`; the resolver
  `RenderService.MergeConfig` / `BuildRenderAsync` (global → dashboard → version).

**To reuse / pitfalls.**
- **Forward-only versioning is load-bearing** — a startup guard refuses to run against a schema ahead
  of the binaries; never publish a version going *backward*. (See *Operational backbone*.)
- The **structural-vs-presentation split** is the subtle part: snapshot what a layout was validated
  against, late-bind the rest. Getting this wrong means either published screens drift on a global
  change, or per-entity overrides can't take effect.
- The **inherited marker** (record intent, not just the resolved number) was field-found: a value
  published while the default was 0 otherwise pins to 0 forever and ignores a later default.
- **Lock liveness + displaced-editor read-only**: a draft left open in an idle tab must drop to
  read-only on its next liveness check and be force-releasable by an admin — otherwise it strands the
  entity.

*Finish-guide tie:* §4 (inheritance explicit), §2 (confirm/guard-rails), §7 (selection-scoped lists).

---

## 7. Render & transform pipeline

**Problem.** Turn arbitrary producer data into pixels on a wall **safely** (untrusted scripts and HTML
can't escape), **cheaply** (compute once, not once per screen), and **extensibly** (a new visual is a
manifest + a component, not a core change).

**Shape.**
- **A manifest-described primitive catalog.** Each renderer primitive declares a manifest — `typeId`,
  category/icon, **config schema**, **view-model schema** (the contract a transform must produce),
  sample view-model, accepted binding kinds, whether it runs a transform. The client dispatches on
  `primitiveTypeId` to a render component; the server validates transform output against that schema.
  Adding a primitive = manifest + component, with no edits to dashboards/widgets/runtime.
- **A sandboxed transform shapes source → view-model.** An author writes an **Expression** (one JS
  expression), **Script**, or **Template** (returns `{html|svg}`). It runs in an embedded JS engine with
  **no host surface** under **hard limits** (timeout / memory / statements / recursion — each a default
  clamped to a maximum). The script sees `sources.<name>` (`.data` parsed-JSON, `.text` raw, `.fresh`,
  `.asOf`) and `config`, and returns the view-model; output is **schema-checked** before it's trusted.
- **Compute-once, fan-out-many.** A view-model is computed **once per source-data version** and cached
  (key = widget-type-version + config + the sources' content-versions), then a tick pushes
  `WidgetUpdate`s to every station showing that instance — **hash-gated**, so an unchanged widget
  re-renders nobody. Stations apply the update **in place** (no component teardown).
- **The iframe is the security boundary for untrusted output.** The **markup** primitive renders server
  HTML in a `sandbox=""` iframe — *no scripts, no navigation, isolated origin* — delivered via `srcdoc`
  set **imperatively** (a framework's attribute sanitizer would strip the author's `<style>`; the
  sandbox, not sanitization, is the boundary). The **embed** primitive allows scripts but blocks
  top-navigation and popups.
- **Preview is the real pipeline, not a mock.** Authoring renders a draft/transform through the *exact*
  path a station uses (same version resolution, bindings, transforms, staleness) — a *try-transform*
  endpoint for the script editor, a *preview-render* for the dashboard editor.

**Contracts.**
- A **primitive manifest** = id + config-schema + view-model-schema + sample + binding kinds +
  supports-transform.
- A **transform sandbox** with enforced limits and a fixed scope (`sources.<name>.{data,text,fresh,
  asOf}`, `config`) → schema-validated view-model.
- A **cache key from content versions** ("same inputs → same output, served once").
- **Untrusted output renders in a sandboxed iframe**, never in the app's own DOM.

**Implementation (FDP).**
- Catalog: `src/Dashboards.Domain/Contracts/PrimitiveCatalog.cs` (the 12 manifests). Dispatch:
  `src/web/projects/primitives/src/lib/widget-host.component.ts` (`ngSwitch` on `primitiveTypeId`;
  `applyUpdate` for in-place streamed updates).
- Sandbox: `src/Dashboards.TransformRuntime/JintTransformExecutor.cs` (engine limits —
  `TimeoutInterval`/`MaxStatements`/`LimitRecursion`/`LimitMemory`; the `sources`/`config` scope;
  expression-vs-script wrapping; schema check against the manifest). Limit clamping:
  `GlobalConfig.SandboxLimitsConfig.ResolveOverride`.
- Cache: `src/Dashboards.TransformRuntime/ViewModelCache.cs` (SHA-256 of widget-version + config +
  sorted source-content-versions; oldest-10%-evict).
- Fan-out: `src/Dashboards.Host/Services/LiveUpdateService.cs` (≈2 s tick, `BuildWidgetAsync`,
  hash-gated `WidgetUpdate`); `Hubs/LiveHub.cs`.
- Sandboxing: `src/web/projects/primitives/src/lib/components/document-markup-embed.ts` (markup
  `sandbox=""` + imperative `srcdoc`; embed `allow-scripts` minus top-nav).
- Preview seam: `WidgetTypesController` `try-transform`, `DashboardsController` `preview-render`,
  `RenderService.BuildDraftPreviewAsync`.

**To reuse / pitfalls.**
- **The manifest is the extensibility contract** — keep the view-model schema additive; a breaking
  change is a major-version event.
- **Never render untrusted HTML in your own DOM** — the sandboxed iframe is the boundary, and beware a
  framework's "safe" attribute binding *sanitizing away* exactly the styling the author needs (set
  `srcdoc` imperatively, let the sandbox secure it).
- **Key the cache on content versions, not wall-clock** — "compute once per input change" is what lets
  one transform serve a whole building.
- **Hash-gate the fan-out** — pushing unchanged view-models to N stations is the easy way to melt the hub.
- **Make preview the real pipeline** — a preview that mocks the render path lies exactly when it matters.

*Finish-guide tie:* §2 (try/preview = immediate feedback), §6 (render/transform failure is a UI state —
the stale/error overlay; → *Diagnostics & log viewing*).

---

## 8. Data ingestion — scheduled producers

**Problem.** Pull content from **systems you don't control** — a script that hits an FTP, a query that
renders a chart, a slide deck split to images — on a schedule, **chained**, bounded, and **without ever
serving a half-written file**.

**Shape.**
- **A task = an external process + a schedule + an output folder.** The process (executable, optional
  interpreter, args, working dir) writes into *one* folder; the platform serves the **newest file**
  there as that task's source. Schedules are **interval / hourly-at-minute / daily-at-time /
  after-another-task-succeeds** (chaining).
- **One scheduler ticks them all.** A short tick (≈5 s) computes each task's next-due (DST-safe local
  arithmetic for hourly/daily), runs those due **under a concurrency cap**, and — key — leaves an
  over-cap task **still due** so it is *delayed, never skipped*. **Chaining** fires a follower the moment
  its predecessor *succeeds*; a follower already mid-run gets **exactly one** coalesced re-trigger, never
  a backlog.
- **The runner is defensive.** It captures stdout/stderr (bounded), enforces a **timeout** (killing the
  whole process tree), resolves success against the task's **success exit codes**, and writes a **run
  history** row (started/completed/exit/output/error) the UI shows and the problems overview consumes.
- **Producers publish atomically; readers resolve newest-wins.** Convention (the producer's job): write
  a temp name, then **rename**. Readers ignore `*.tmp`/`*.partial`, take the newest match, and stamp a
  **content-version = path + last-write-ticks** (the render cache key). An optional per-binding **file
  selector** (glob) lets one folder feed several widgets.
- **A source is the binding seam.** A `ContentSource` is a **task output** (freshness = last successful
  run) or an **external folder** (freshness = newest file mtime); widgets bind to the source's identity,
  so a moved producer is a one-place fix.

**Contracts.**
- A **schedule-kinds** set including event-driven **chaining** with "exactly one follow-up, no backlog."
- A scheduler that is **idempotent and skip-proof** (over-cap stays due) and re-anchors on edit.
- A runner with **timeout-kills-the-tree**, configurable success codes, bounded capture, run history.
- The **atomic-write + newest-wins + content-version** producer/reader contract.

**Implementation (FDP).**
- Entities/schedule: `src/Dashboards.Domain/Entities/Sources.cs` (`CollectionTask`, `TaskRun`),
  `Enums.cs` (`TaskScheduleKind`).
- Scheduler: `src/Dashboards.TaskEngine/CollectionScheduler.cs` (tick, the `_nextDueUtc`/`_running`/
  `_chainPending` state, concurrency cap, `TriggerChained`). Next-due math: `ScheduleCalculator.cs`
  (DST spring-forward/fall-back; AfterTask = never-due-by-clock).
- Runner: `src/Dashboards.TaskEngine/ProcessRunner.cs` (launch, capture cap, timeout/kill-tree, exit-code
  resolution). Completion fan-out: `Abstractions.cs` `ITaskEventSink` → problems + source recompute.
- Readers/conventions: `FolderSourceReader.cs` / `ImageSourceReader.cs` (newest-wins, `.tmp`/`.partial`
  filter, `ContentVersion = path|ticks`), `SourceFilePattern.cs` (the glob selector, FR-121).
- API/UI: `TasksController`, `tasks.page.ts` / `sources.page.ts`.

**To reuse / pitfalls.**
- **Over-cap = delayed, not skipped** — the subtle correctness point; otherwise you silently drop runs
  under load.
- **Chaining is coalesced, not queued** — exactly one follow-up after a busy predecessor, or a slow
  follower builds an unbounded backlog.
- **Atomic write is the producer's contract** (temp→rename); the reader can only ignore in-progress
  names and ride the lock window (→ *Data-edge resilience*).
- **Content-version = path + mtime ticks** is the cheap, stable key that ties ingestion to the
  compute-once render cache (→ *Render & transform pipeline*).
- **DST-safe local time** for hourly/daily, or a clock change double-runs or skips.

*Finish-guide tie:* §2 (run history + schedule preview answer "did it work / did I get the schedule
right?"), §5 (freshness is a first-class signal).

---

## 9. Versioned assets & telemetry-driven GC

**Problem.** Operators upload managed assets (images, video, PDFs) that screens present; a new upload
must **replace** the old **without yanking it off a wall mid-play**, and old versions must be
**reclaimed** — but only once **no screen is still showing them**.

**Shape.**
- **Each asset is a version chain.** An asset has a `CurrentVersionId` pointer and immutable versions,
  each in a state: **Active → Superseded → Collectible → Collected**. Files are written **atomically**
  (temp→rename) under a **collision-safe stored name** (`{assetId}_v{n}_{safe}.{ext}`), immutable once
  written.
- **Replacement is graceful or immediate.** Publishing a new version marks the old **Superseded** and
  either pushes an **immediate** swap or lets consumers **adopt at the next loop boundary** (a playlist
  finishes its cycle first) — never a mid-frame yank.
- **GC is reference-counted by live telemetry.** Stations report the **asset-version ids they're
  currently presenting**. A periodic sweep collects a superseded version only when it's **in no live
  station's presented-set** *and* past a short **grace age** — with a **time backstop** so a silent,
  non-reporting station can't pin a version forever. The sweep is **paused during the backup window**
  (consistent snapshot) and also reaps **orphan `*.tmp`** files from aborted uploads.

**Contracts.**
- An asset **version chain** with an explicit state machine and a "current" pointer.
- **Atomic, immutable, collision-safe** stored files.
- **Graceful vs immediate** replacement (adopt-at-boundary).
- **Telemetry-driven ref-counting GC** with a grace age **and** a time backstop for non-reporters.

**Implementation (FDP).**
- Entities: `src/Dashboards.Domain/Entities/Media.cs` (`MediaAsset`, `MediaAssetVersion`, the state
  enum). Upload/replace: `src/Dashboards.Host/Services/MediaService.cs` (atomic write, safe naming,
  supersede + two-phase pointer swap, immediate-vs-graceful push).
- GC: `MediaGcService` (≈2 min tick; reads `presentedAssetVersionIds` from each station's telemetry;
  grace age + `AssetGcBackstopHours`; orphan `.tmp` sweep; gated by `MediaGcGate` during backups).
- The presented-set comes from station telemetry (→ *Realtime connected-clients admin*).

**To reuse / pitfalls.**
- **Ref-count by what's *actually on screen*, reported by the consumer — not by config references.** The
  version a dashboard *references* may differ from what a station is *presenting*; telemetry is the truth.
- **Always pair the ref-count with a time backstop** — a station that stops reporting must not pin a
  version (or its disk) forever.
- **Quiesce the GC during backup** so a snapshot never captures config pointing at a just-deleted file.
- **Graceful adoption at a boundary** keeps a swap from yanking content mid-play; offer immediate only
  when the operator asks for it.

*Finish-guide tie:* §5 (live telemetry drives correctness, not just display), §6 (replacement degrades
gracefully).

---

## 10. Data-edge resilience

**Problem.** The platform reads files a **foreign producer writes concurrently** and serves them to
displays. A read that lands mid-write must not become a user-visible failure (a 500, a broken-image
icon, a blank screen) — and the operator should be able to tell *why* when it does.

**Shape.**
- **Producers write atomically**: to a temp name, then **rename** into place. The reader **ignores
  in-progress names** (`*.tmp` / `*.partial`) and serves **newest-wins**, so a half-written file is
  never visible and the only race left is a single atomic rename.
- **Readers ride through** the rename/lock window: open allowing concurrent writers
  (`FileShare.ReadWrite | Delete`), **retry** transient I/O a few times, then **degrade gracefully**
  (a 503, not a 500) and **log a warning that names the file** so it correlates with the producer's
  timing.
- **The display holds the last good frame** rather than painting a failure: it preloads the next
  asset off-screen and swaps only on a successful load; a failed/locked fetch leaves the previous
  frame up.
- **Freshness is a first-class signal, not an error**: each source carries a *fresh-as-of*; past a
  per-widget window it shows the **stale** indicator (color + word/icon), distinct from an **error**.

**Contracts.**
- Atomic-write convention (temp/rename) is the **producer's** responsibility — the reader can ride a
  *lock*, but cannot un-tear an in-place partial write.
- Error messages **distinguish "missing / unreadable" from "empty"** — the first is usually a path or
  service-account-permission problem, the second is genuinely no content.

**Implementation (FDP).**
- Readers: `src/Dashboards.TaskEngine/ImageSourceReader.cs` (`FindNewest` / `FindAllSorted` — the
  image-extension floor, the `.tmp`/`.partial` exclusion, case-insensitive sort) and
  `FolderSourceReader.cs`.
- Ride-through + logging: `src/Dashboards.Host/Controllers/LiveController.cs` `SourceSlide` (the
  `FileShare.ReadWrite|Delete` open, the bounded retry loop, the `LogWarning` naming the file, the
  graceful 503).
- Last-good-frame: `src/web/projects/primitives/src/lib/components/dynamic-slideshow.component.ts`
  (`showSrc` preloads via `new Image()` and swaps on `onload`; on `onerror` it keeps the current
  frame; a sequence stamp drops stale loads).
- Stale/error overlay: `src/web/projects/primitives/src/lib/widget-host.component.ts` (the badge /
  border / dim styles, driven by `isStale` + `errorState`).
- The render path that distinguishes "folder not found / unreadable" from "no images":
  `RenderService.cs` (the dynamic-slideshow branch).

**To reuse / pitfalls.**
- A UNC/network source must be readable by the **service account**, not the operator — `LocalSystem`
  hits a share as the machine account. A `dir` that works for you proves nothing about the service.
- On a **regenerating set** (count shrinks), an index from the previous view-model can fall out of
  range: 404 it quietly and let the new view-model (a version token that flips on change) re-sync;
  don't error.
- Distinguish-the-error wording pays for itself — it turns "the widget is blank" support tickets into
  a one-line answer.

*Finish-guide tie:* §6 (failure is a UI state), §5 (stale signaling, color + word).

---

## 11. Operational backbone

The two infrastructure patterns the verticals above quietly rely on. (Operator *procedures* — install,
migrate, recover — live in the admin guide; this is the architectural shape only.)

**Hosted-service sweeps.** Periodic `BackgroundService`s that **reconcile state and raise/clear
problems by a stable key**. The shape is always: a `while (!stopping)` loop wrapping a `TickAsync`
with its own `try/catch` (one bad tick never kills the loop) and a `Task.Delay(interval)`; the tick is
**idempotent** (it computes the desired state and raises or clears, so running it twice is harmless).

- Registered in `src/Dashboards.Host/Program.cs` (`AddHostedService<…>`): `StationMonitor` (offline
  detection), `RenderProblemMonitor` (proactive render-error sweep — see §5), `MediaGcService`,
  `RetentionPurgeService`, `HeartbeatService` (see §2), `BackupService`, `CollectionScheduler`,
  `LiveUpdateService`.
- Template: `src/Dashboards.Host/Services/StationService.cs` `StationMonitor` (a 30 s tick that flips
  connection state and `RaiseAsync`/`ClearAsync`es problems by key).
- **Pitfall:** raise/clear must key on a **stable identity** (`"station-offline:" + id`) so the same
  condition updates one problem rather than spawning duplicates, and clears cleanly when resolved.

**Monotonic schema versioning.** A single integer `SchemaInfo.CurrentVersion`
(`src/Dashboards.Domain/SchemaInfo.cs`) that the binaries require. The service **refuses to start when
the database is behind** (the guard), the recovery tool verifies it before writing, and migrations
advance it. Versions are **forward-only**: shipping vN then vN-1 strands a database that's already at
vN. Recipe (entity → `dotnet ef migrations add` with the DataAccess project as its own startup → hand-add
the `SchemaVersions` row → bump `SchemaInfo` → regenerate the idempotent SQL) is in the project notes.

- **Pitfall:** the service never auto-migrates (KD-21) — a behind database is an explicit operator
  action (`fd-dbtool apply`), so an upgrade can't silently rewrite data on first run.

*Finish-guide tie:* §6 (the version guard is a guard-rail, not a warning); §2 (long jobs report
progress).

---

## Appendix — Feature patterns

More domain-specific than the verticals above, but each carries a reusable idea worth naming. Brief
treatment; see the cited files for depth.

- **Broadcasting / emergency override** — `Services/BroadcastService.cs`, `BroadcastResolution.cs`,
  `Entities/Broadcasting.cs`. *Reusable idea:* a **render-time override of normal routing**. A channel
  names a target dashboard and a membership (an explicit set, or a dynamic "all stations" rule); at
  resolve time the **most-recently-activated** matching channel wins, and deactivation falls straight
  back to the station's real assignment — the override **never mutates** it. Generalizes to any
  alert / maintenance / lockdown "force a view onto a fleet" need.
- **Backup / restore** — `Services/BackupService.cs`, `RemoteBackupSupport.cs`,
  `docs/CROSS_HOST_SQL_BACKUP.md`. *Reusable idea:* a **consistent snapshot** taken by **quiescing the
  GC** for the window (so config never references a just-collected file), plus a **cross-host DB backup**
  technique — the engine writes under *its own* identity to a single-principal UNC share, so you grant
  the SQL host's **machine/service account**, not the operator. Retention keeps N newest; restore is the
  out-of-band dbtool.
- **Export / import** — `Controllers/DashboardsController.cs` (export/import), `Dtos/DraftModels.cs`
  `DashboardExport`. *Reusable idea:* a **self-describing JSON portability format** (a `format` stamp +
  the entity model + its *referenced* sub-entities) with **resolve-by-id-then-by-name** on import and an
  actionable failure ("import widget type 'X' first"). Import lands as a **draft**, never a live change.
- **Station pairing / registration** — `Entities/Stations.cs` (`PendingStation` / `Station`),
  `Services/StationService.cs`. *Reusable idea:* a **persistent device token** (issued on first boot,
  survives everything) plus a **human-readable pairing code** an operator claims in the admin UI;
  claiming **rebinds the already-open live connection** so the screen flips holding-page→live without
  waiting for a reconnect. A station can also **pin a version** (freeze a screen on last-known-good),
  independent of the dashboard's live pin.

---

*This document and `WEB_UI_FINISH_GUIDE.md` are the two halves of the template: build a screen to the
finish guide, build a capability to these patterns. Both are grounded in the Facility Dashboard Platform
— carry the shapes forward, fork the code.*
