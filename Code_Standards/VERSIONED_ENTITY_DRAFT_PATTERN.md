# Versioned-Entity Draft Pattern

*A reusable design for editing durable entities through a journalled draft, with undo/redo, single-editor
locking, and publish-to-immutable-version. Extracted from the Facility Dashboard Platform (dashboards,
widget types, playlists); written to be copied into other projects.*

The Facility Dashboards code is the reference implementation — file/type names below are from there.
Where a concept generalizes, the **generic role** is named first and the **dashboard example** follows.

---

## 1. What it is, and when to use it

A pattern for any entity where edits must be **deliberate, reversible, auditable, and atomically promoted** —
not saved field-by-field straight onto the live record. You get:

- A **working copy** (draft) separate from what's published/live, so in-progress edits never affect production.
- A **durable, ordered change journal** per draft, giving **undo/redo that survives closing the editor** and a
  record of who changed what.
- **Single-editor locking** with liveness, so two people can't silently clobber each other.
- **Publish** that squashes the draft into a new **immutable, numbered version**; the live surface resolves to a
  version by a **pin-or-follow-latest** policy.

Use it when any of these are true: edits are high-stakes (a mistake is visible to users or expensive to undo);
you need an audit trail or rollback; multiple people edit the same entities; or you want "save a draft, publish
later" semantics. It is **overkill** for low-stakes CRUD (settings a user edits and immediately owns), for which a
plain record + optimistic concurrency is simpler.

---

## 2. The shape — four roles (tables)

```
  Entity  1───*  Version            (published, immutable, numbered)
    │
    │ 1
    *
  Draft   1───*  DraftChangeEvent   (ordered journal; one row per user gesture)
```

### 2.1 Entity — the durable, named thing
The stable identity users refer to. Holds metadata + a **live-resolution pointer**, owns a list of Versions, and
is **soft-deleted** (never hard-deleted while referenced).

> **Dashboard** (`Domain/Entities/Dashboards.cs`): `Id`, `Name`, `Description`, `Area`, `CreatedBy/At`,
> `LivePinVersionId` (null ⇒ follow latest published; otherwise that version renders live), `IsDeleted`,
> `Versions`.

### 2.2 Version — an immutable published snapshot
Produced only by **publish**. Never edited after creation. Carries an **incremental, non-recycled**
`VersionNumber`, who/when published, and the full resolved content.

> **DashboardVersion**: `Id`, `DashboardId`, `VersionNumber`, `PublishedAtUtc/By`, and the snapshot
> (`WidgetInstances`, canvas size, config overrides). Nested placements may themselves pin-or-follow a child
> entity's version (`WidgetInstance.PinnedWidgetTypeVersionId` — null ⇒ follow latest), i.e. **late binding**.

### 2.3 Draft — the working copy + edit state
One in-progress edit of an entity. Holds the **current working model** (as JSON — shape depends on the entity),
the **undo pointer**, the **editing lock**, and the **journal**.

> **Draft** (`Domain/Entities/Drafts.cs`): `EntityKind` (discriminator — see §7.6), `EntityId`,
> `BaseVersionId` (the published version it started from; null for a brand-new entity), `CreatedBy`,
> `ModelJson` (working state), `AppliedEventCount` (the undo pointer — §4), the lock fields
> (`LockedBy`, `LockSessionId`, `LockAcquiredAtUtc`, `LockLastLivenessUtc`), the takeover record
> (`TakenOverBy`, `TakenOverAtUtc` — §5.3), and `ChangeJournal`.

### 2.4 DraftChangeEvent — one row per gesture
The journal. Append-only within a draft, ordered by `Sequence`.

> **DraftChangeEvent**: `Sequence`, `AtUtc`, `By`, `Kind` (e.g. `widget-move`, `config-set`), `Summary`
> (human-readable, for the undo UI), and **`ModelJson` — the full model state *after* this event**. (Snapshot per
> event, not a delta. See the tradeoff in §7.2.)

The whole engine is one service — **`VersioningService`** — shared across all entity kinds.

---

## 3. Lifecycle

```
 create/clone ─▶  DRAFT  ──(gesture → append event, advance model)──▶  ... ──▶  PUBLISH ─▶  VERSION (immutable)
                   │  ▲                                                           │
                   │  └── undo/redo (move the pointer; non-destructive)           └── squash journal, VersionNumber++
                   └── rollback-to-base / discard (abandon)                           live resolves to pin-or-latest
```

1. **Open a draft** — created from a base version (`BaseVersionId`) or empty for a new entity. Multiple
   concurrent drafts of one entity are allowed; the UI warns about them (`ConcurrentDraftWarningDto`).
2. **Edit** — each completed user gesture calls **append-change**: discard any redoable tail, write a new
   `DraftChangeEvent` with the new full model, and advance `ModelJson` + `AppliedEventCount`.
3. **Undo / redo** — move the `AppliedEventCount` pointer (§4). Non-destructive.
4. **Publish** — validate, then **squash** the draft's journal into a **new immutable Version** built from the
   draft's final model, with `VersionNumber = max + 1`. The draft is consumed; the journal does not carry into the
   version. (`VersioningService.PublishDraftAsync`.)
5. **Abandon** — *rollback-to-base* returns the working model to the base version and clears the journal;
   *discard* deletes the draft entirely.

**Live resolution** (what a consumer actually renders): the Entity's `LivePinVersionId` picks the version — null
means "follow the latest published", non-null means "stay pinned to this one". Nested references resolve the same
way (follow-latest vs pinned), which is what makes **late binding** work: publish a child once and every
following parent picks it up without being re-published.

---

## 4. Undo/redo — a pointer over a persisted, linear journal

This is the crux and the most reused insight.

- The journal is a **single, linear, append-only** sequence (events `1..N`). `AppliedEventCount` is a **movable
  pointer** into it. The model at pointer `k` is event `k`'s snapshot (or the base model at `k = 0`).
- **Undo** = pointer − 1. **Redo** = pointer + 1. **Nothing is deleted** — undo just moves the pointer, so redo
  can move it forward again.
- **Making a new edit after an undo discards the redoable tail** (standard linear-history semantics): append
  removes events past the pointer, then adds the new one. So it is a *line*, not a *tree*.
- Because the journal and pointer are **server-persisted**, undo **survives closing the editor** — reopen a draft
  days later (even in a different session or by a different authorized user) and you can still step back through
  the whole history. This is the headline durability benefit.

> **Insight — undo cannot be "scoped by author."** Each event records its `By`. It is tempting, when a second
> person takes over a draft, to let them undo only *their own* steps. **On a linear stack this is incoherent**: to
> undo event `K` you must first undo everything after it, so you can't skip another author's event to reach yours.
> Author-scoped undo would require a fundamentally different model (branching history, or per-author
> checkpoints/squash-on-handoff). The pragmatic answer is **full-stack access + awareness, not restriction**:
> return each reverted/reapplied event's `By` + `Summary` so the editor can show *what* it undid and *who* made
> it, and flag steps authored by someone else. Don't try to partition the stack.

---

## 5. The editing lock (single-editor concurrency)

Authentication tells you *who*; the lock decides *who may edit this draft right now*.

### 5.1 Acquire / liveness / release
- **Acquire** is an **atomic conditional update** (one SQL `UPDATE … WHERE free-or-expired-or-mine`), so two
  sessions racing for the same draft cannot both win — rows-affected tells you who got it. Prefer this over
  read-then-write.
- The holder sends a **liveness ping** on a timer (30 s). A lock whose last ping is older than a configurable
  timeout is **considered expired** and can be taken by the next acquirer. This self-heals crashed/closed tabs
  without manual cleanup.
- Each editor tab carries a random **`sessionId`**, so the *same user* opening a second tab is detected and offered
  an explicit **reclaim** (rather than silently stealing from their own first tab).
- **Release** happens on editor teardown (navigate away / close). An **admin force-release** exists for stuck
  locks.
- Client nicety: re-check the lock on `visibilitychange` so a backgrounded tab (whose timer throttled) surfaces a
  lost lock the instant it's looked at again.

### 5.2 Warned takeover handoff
Resuming a draft is a right the **author always has** (lost-connection protection). Letting *someone else* pick up
an unlocked draft is a **deliberate, warned handoff**: confirm "started by X — take over?" before acquiring.
Gate who *may* take over by the same permission that may edit the entity — acquiring the lock is the single choke
point (undo/redo/append all require holding it), so authorizing **acquire** authorizes editing.

### 5.3 Durable takeover record (there is usually no per-user push channel)
When a non-author acquires the lock, stamp `TakenOverBy`/`TakenOverAtUtc` on the draft (cleared when the author
retakes it). The original author learns of it **when they return** (a banner), because most systems like this have
**no per-user notification channel** — identities are directory principals with no stored email and no in-app
inbox. Don't assume you can "email the author"; a durable, pull-on-return record is the honest mechanism.

---

## 6. Benefits

- **Safety** — in-progress edits never touch the live/published record; publish is atomic.
- **Reversibility** — durable undo/redo that outlives the session; rollback-to-base; nothing is lost on undo.
- **Auditability** — every change has an author, timestamp, kind, and summary; every published version records
  who/when.
- **Immutable history** — numbered, never-recycled versions are a stable reference for pinning, rollback, and
  "what was live on date X".
- **Late binding** — pin-or-follow-latest lets you publish a child once and have dependents pick it up, or freeze a
  dependent to a known-good version.
- **Collaboration without clobbering** — single-editor lock + liveness + reclaim + warned takeover.
- **One engine, many entity kinds** — a single draft/journal/lock/publish service serves every versioned entity
  via a discriminator.

---

## 7. Design decisions & hard-won insights

### 7.1 Squash-on-publish; the journal does not live in the version
A version stores only the **final resolved snapshot**, not the journal that produced it. The journal is a *drafting
aid*; it is squashed away at publish. Keep published versions clean and self-contained — don't leak draft history
into them.

### 7.2 Snapshot-per-event vs delta (a real tradeoff)
This implementation stores the **full model after each event**. Pros: undo/redo is a trivial pointer move (no
replay), and each event is independently restorable. Con: storage grows with edit count × model size. For small
models (a dashboard layout) this is fine and much simpler than deltas. For large models, consider deltas +
periodic snapshots — but only if you measure a real problem; snapshot-per-event is the right default for its
simplicity.

### 7.3 Versions are immutable and numbers never recycle
Once published, a version is read-only; editing means a *new* draft → *new* version. Version numbers are
monotonic per entity and are never reused even after history collapse, so any external reference (a pin, a log,
"v7 was live when the incident happened") stays meaningful.

### 7.4 Reference-guarded deletion + history collapse
Don't hard-delete an entity that something still references (a station assigned to a dashboard; a widget type used
by a dashboard version). Soft-delete (`IsDeleted`) and guard. Offer a **history collapse** (truncate old versions)
as a separate, privileged, destructive operation — distinct from normal editing.

### 7.5 Concurrent drafts are allowed — publish is last-writer-wins, guarded by an acknowledged stale-base warning
Multiple drafts of one entity can coexist (two people exploring different changes); draft creation does not block a
second draft, and the lock is per-*draft*, not per-*entity*. **Be clear-eyed about the consequence at publish:**
each publish simply takes `VersionNumber = max + 1` from the publishing draft's own model. It does **not** merge
other drafts.

So if A and B both branch from version N: A publishes N+1; B's draft is untouched (still based on N); if B then
publishes N+2 built from N, **A's changes (N+1) are not included**, and if the entity follows latest, N+2
supersedes N+1 as live. Nothing is destroyed — N+1 remains an immutable version, recoverable by pinning or cloning —
but no merge happens.

**What the reference implementation ships to make this safe-by-acknowledgment (optimistic concurrency):**
1. **Author-time awareness — including *live*.** `/drafts/concurrent` reports other drafts of the same entity. The
   editor shows an **acknowledgeable banner** (dismissed per open, so it re-shows every time the draft is reopened)
   naming the other draft's author, so a second editor can coordinate, **take over** the existing draft instead of
   branching, or knowingly proceed. Crucially the banner is also raised **live**: the editor subscribes to the
   model-changed signal for its area (the same SignalR feed the list pages use) and re-queries `/drafts/concurrent`
   when it fires; if a *new* concurrent draft appeared, the acknowledgement is reset so the banner re-shows. This
   closes the asymmetry where only the *second* author would otherwise see it — a **first** author who opened with no
   concurrent draft now gets the identical notice the moment a second author branches one, without a reload. (Draft
   creation already fires the area's change notification, so no extra server plumbing is needed; the reset-on-grow is
   driven off the set of other-draft ids, so a mere edit elsewhere doesn't nag.)
2. **Stale-base check at publish.** Before publishing, compare the draft's `BaseVersionId` to the entity's current
   latest version; if a newer version was published since the draft branched, it is a **publish warning that must be
   acknowledged** (reusing the existing `acceptWarnings` mechanism, so it flows through the normal validate →
   publish-dialog UX with no new client plumbing). The message names the versions: *"based on v{base}, but v{latest}
   has since been published; publishing creates v{latest+1} from v{base} and will NOT include v{latest}."* The same
   warning is surfaced in the draft `validate` path so it appears in the publish dialog. Handle a base version that
   was removed by a history collapse gracefully (warn without naming a number).

This is **advisory, not preventive** — the author can acknowledge and publish anyway (superseding), which is also the
deliberate *promote-an-older-version-to-latest* pathway, so the warning correctly fires there too. It does **not**
close a true *simultaneous*-publish race (two publishes passing the check in the same instant still resolve
last-writer-wins at `max + 1`); a hard guarantee would need entity-level row-versioning/transaction. **Automatic
rebase/merge** ("replay this draft onto the latest") remains deliberately **out of scope** — a separate, larger
feature. Immutable history is the backstop throughout: a superseded version is never lost, only no longer latest.

### 7.5a A draft's base version must never be orphaned — so it counts as a reference
A draft carries a `BaseVersionId`: the version it branched from, which publish reads to compute `max + 1`. If that
version were removed while the draft is still open, the draft would be orphaned (its base gone, its stale-base warning
unable to name a number). So **a version referenced by an open draft is treated as referenced for removal purposes**,
the same as a live-pin or a station-pin. The reference-guard that backs both version deletion *and* history collapse
lists "base of a draft by X" among the things depending on a version, which has two consequences worth stating
explicitly as decided behavior:

- **Hard-deleting a version** that is a draft's base is **refused** with a reference list that names the blocking
  draft(s), exactly as a live-pinned or station-pinned version is refused. The author resolves it by publishing or
  discarding the draft first.
- **History collapse is "reduce history", not "delete a chosen range".** Our collapse removes *non-latest, unreferenced*
  versions; because a draft's base counts as a reference, collapse **succeeds while retaining** any version a draft
  still needs (and always retains the latest). So collapse never orphans a draft and needs no special-case — the
  reference count already does the right thing. (This is deliberately *not* a user-selected contiguous-range collapse,
  which would instead have to *fail* when the range covered a referenced version. If you implement that variant, make
  it fail loudly rather than silently drop the referenced version.)

For consistency the **entity-level** delete guard also lists open drafts (deleting the whole entity would orphan the
draft too), so all three kinds — dashboard, widget type, playlist — block entity deletion on an open draft identically.

### 7.6 One discriminator, one engine
`EntityKind` on the draft (Dashboard / WidgetType / Playlist) lets a **single** `Draft`/`DraftChangeEvent` table,
`VersioningService`, lock, undo/redo, and draft-editor API serve every versioned entity. The per-kind differences
(what the model JSON means, how a version is built, who may edit) are small branch points, not separate stacks.
Replicating this: resist the urge to build a bespoke draft stack per entity type.

### 7.7 Authorization: gate *acquire-lock*, enforce per-kind
Because undo/redo/append all require holding the lock, **acquiring the lock is the single gate** — put the
edit-rights check there and the rest follow transitively. Edit rights can differ per kind (in Facility Dashboards,
dashboards/widget-types are a "developer" capability, playlists are "developer or media"); mirror the same per-kind
check on **publish**.

### 7.8 Client session mechanics worth copying
A thin client "draft session" object: owns the `sessionId`, loads the draft, acquires the lock, runs the liveness
timer, exposes `undo/redo/rollback/publish`, re-checks on tab-visibility, and releases on teardown. Keeping all of
that in one place (not spread across the editor component) is what let three different editors (dashboard, widget
type, playlist) share identical behavior — including, later, the takeover handoff, added in **one** place.

### 7.9 Schema / migration notes
The journal is a child table with a FK + ordering index on `(DraftId, Sequence)`. Model/JSON columns are large
text. Lock + takeover fields are a few nullable columns on the draft. Adding takeover later was a 2-column,
single-migration change — the model absorbs incremental additions cleanly. If your platform refuses to start on a
schema-version mismatch, remember each such addition is a real migration to run on deploy.

---

## 8. Reuse checklist — applying this to a new entity

1. **Model the three/four tables**: `Entity` (metadata + `LivePinVersionId` + soft-delete + `Versions`),
   `Version` (immutable snapshot + `VersionNumber` + published-by/at), `Draft` (EntityKind, EntityId,
   BaseVersionId, ModelJson, AppliedEventCount, lock fields, takeover fields), `DraftChangeEvent`
   (Sequence, By, AtUtc, Kind, Summary, ModelJson). Reuse one `Draft`/`DraftChangeEvent`/engine across kinds.
2. **Decide the model-JSON shape** per kind (what "the working model" is) and how a Version is built from it.
3. **Implement the engine ops**: open/create draft, append-change (discard-tail → write event → advance),
   undo/redo (pointer ± 1, return reverted/reapplied `By`+`Summary`), rollback-to-base, discard, publish
   (validate → squash → new version → consume draft), and the live-resolution rule (pin-or-latest).
4. **Implement the lock**: atomic conditional acquire, liveness ping + timeout, same-user reclaim via sessionId,
   release on teardown, admin force-release, warned takeover + durable takeover record.
5. **Authorize at acquire-lock and publish**, per kind.
6. **Build the client draft-session** object (§7.8) and have every editor use it.
7. **Guard deletion** (soft-delete + reference check) and offer history-collapse separately.

## 9. Pitfalls

- **Don't** edit the live record in place "just this once" — it defeats the whole safety story.
- **Don't** try to make undo author-aware on a linear stack (§4). Add awareness, not restriction.
- **Don't** store the journal inside the published version (§7.1).
- **Don't** assume a per-user push channel for takeover/notification; use the durable record (§5.3).
- **Don't** hard-delete referenced entities; recycle version numbers; or let two sessions read-then-write the lock.
- **Don't** assume concurrent drafts merge — publish is last-writer-wins (§7.5). The reference implementation guards
  it with an acknowledged stale-base warning at publish + an author-time concurrent-draft banner (optimistic
  concurrency, advisory); automatic rebase/merge is out of scope.
- **Do** gate editing at acquire-lock (§7.7), and keep client session logic in one object (§7.8).

---

*Reference implementation: `Dashboards.Host/Services/VersioningService.cs`, `Dashboards.Host/Controllers/DraftsController.cs`,
`Dashboards.Domain/Entities/{Drafts,Dashboards}.cs`, and the web `draft-session.ts`. FR/KD numbers (FR-57..FR-66,
KD-03, KD-14, KD-15) in those files point at the originating requirements.*
