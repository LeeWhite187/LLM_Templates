# How a Web UI Should Perform and Feel — Finish Standard

**Purpose.** A reusable standard for judging and building "finished-feeling" web UIs, written for
engineers who know when software *works* but want shared vocabulary and concrete rules for when it
*feels done*. Grounded in the Facility Dashboard Platform's authoring UI, where each rule cites the
implementation that realizes it — but the rules are general and intended for future design/build
sessions on any project.

The single organizing idea: **a finished UI never leaves the user guessing** — not about whether a
click registered, not about what a field expects, not about why something moved. Every rule below
is that idea applied to one surface.

**Two lenses, two documents.** This guide is the *horizontal* lens — cross-cutting UI principles you
**review any screen against**, largely stack-agnostic and stable. Its companion, **`PLATFORM_PATTERNS.md`**,
is the *vertical* lens — capabilities described end-to-end (client ↔ API ↔ service ↔ data) that you
**build from**: live entity push, connectivity self-heal, security/authorization, the realtime
connected-clients admin, diagnostics, draft→publish editing, and data-edge resilience. Where a rule
here has a backend story, it points down with **→ Patterns: §X**; read the full mechanism there. The
**Appendix** at the bottom of this file is the design-token + component reference for the look itself.

**Self-contained handoff.** These two documents are the design basis on their own — you do **not** need
the FDP repo to use them. The `path` + `symbol` code references are *illustration* ("here's one that
works"), not required reading; each rule and pattern states the design in prose. The one artifact worth
copying verbatim — the stylesheet — is **embedded at the end of this file**.

---

## 1. Motion and scrolling (the "feel" of the machine)

Vocabulary first — "smooth" decomposes into four measurable properties:

| Property | Definition | Failure reads as |
| --- | --- | --- |
| **Responsiveness** | Gesture-to-first-pixel latency | "It ignores me, then lurches" |
| **Frame pacing** | Steady 60 fps motion, no dropped frames | "Jerky" — freezes mid-scroll, then jumps |
| **Stability** | Nothing shifts/flickers during motion | Hover strobing, scrollbar pop-in |
| **Predictability** | Equal input, equal motion | Wheel clicks travel different distances |

Rules:

- **Nothing repaints per scroll frame.** The classic killer is hover effects under a stationary
  cursor while rows stream past — suppress hover work *while scrolling is in progress* and restore
  it the instant motion stops. (Implementation: an `is-scrolling` class toggled by passive
  listeners running outside the framework's change detection, with a ~150 ms quiet period;
  CSS keys hover/transition suppression off the class.)
- **Scroll listeners must be passive and framework-invisible.** A scroll handler that triggers
  change detection turns every frame into render work. (Angular: `NgZone.runOutsideAngular`.)
- **Reserve scrollbar space** (`scrollbar-gutter: stable`) so content never shifts when a bar
  appears, and **contain overscroll** (`overscroll-behavior: contain`) so an inner list hitting
  its end doesn't jolt its parent.
- **Animations are seasoning, not theater**: 60–150 ms, ease-out, opacity/transform only (the
  compositor-cheap properties), and never gating input. If an animation can be noticed *waiting
  for*, it is too long.
- **Honor `prefers-reduced-motion`** — all decorative motion off.
- **Know the transport ceiling.** Over RDP/VDI, every frame is encoded and shipped; judge
  smoothness in a local browser before profiling the page.

## 2. Feedback (no dead controls, no silent actions)

- **Every enabled control acknowledges a press** — a visible pressed state (push-down transform).
  A button that does nothing visible on click reads as broken even when it worked.
- **Every persistent action confirms** — a success toast for gating actions ("Changes applied to
  the draft (not yet published)"), an error toast with the *server's* message for failures. Silence
  after a click that saved something is indistinguishable from a click that didn't.
- **State that applies immediately says so** ("Applied immediately" in the toast), and state that
  doesn't says what's pending ("not yet published").
- **Progress for anything longer than ~1 second** — uploads get per-file progress visible
  app-wide; long server jobs get a busy indicator ("backup running…"). A global in-flight
  indicator (a thin top progress bar driven by an HTTP interceptor counting outstanding
  requests) covers *every* page at once — but gate it behind a short anti-flicker threshold
  (~250 ms) so fast local calls never strobe it, while a slow fetch across a routed/firewalled
  link shows as genuinely in progress rather than a hung page.
- **A transient outage must not permanently break navigation.** In a code-split SPA, the browser
  caches a *failed* lazy-route import as permanently rejected — so a click made while the backend
  was unreachable stays dead even after recovery, until a full reload. Preload route code up
  front (also a latency win) so the code is resident before any outage, and as a backstop, on a
  navigation that fails with a code-load error, probe the backend and recover with a fresh
  document load when it answers. A dead button after the server returns is the user's first sign
  the SPA wasn't built for the real network.
- **No one-way doors**: every expanded panel, wizard, or mode has an explicit Cancel *and*
  dismisses on click-away or Escape. Destructive actions confirm with the consequence spelled out
  ("Its output source is removed with it").
- **Guard rails over warnings** where the consequence is unrecoverable by the UI: refuse the
  action server-side with a human explanation ("This is the last enabled admin account…"), don't
  warn-and-allow. The recovery path must never be reachable through planned UI actions.

## 3. Layout discipline (chrome stays, content moves)

- **Headings and action buttons never leave the viewport.** The page fills the window; only the
  content region scrolls. Primary actions (Save/Apply/Publish) are pinned — in a fixed toolbar or
  sticky at the bottom of their scrolling column — never buried below the fold.
- **Sticky table headers** inside every scrolling list, with an opaque background so rows slide
  under them.
- **No layout shift, ever**: reserve space for scrollbars, badges, and validation messages rather
  than letting them push content.
- **One spacing scale** (here: 4/6/8/12/16/24 px) and one corner radius family; misaligned
  edges read as unfinished faster than any missing feature.
- **Empty states are designed**, not blank: "No media assets yet." / "No problems. Nice." A blank
  table is indistinguishable from a broken one.
- **Expandable sections use disclosure controls**: a chevron that points right when collapsed and
  rotates down when expanded, with the control highlighted while open — the trigger must visibly
  *own* the section it reveals. A plain action button that secretly expands a row gives the user
  no model of what it did or how to undo it. (Implementation: shared `disclosure` button class;
  applied to Versions/Usage/Runs row expanders.)

## 4. Input ergonomics (the field teaches its own format)

- **Enumerable values are never free text.** Closed sets get a `<select>` (media fit, transform
  kind, permission group); large known catalogs get a populated select (IANA time zones via
  `Intl.supportedValuesOf`); suggestible-but-open values get a combo box (`<datalist>`: locales,
  date formats). A free-text field whose valid values the user cannot guess is a defect.
- **Domain-specific editors for domain values**: colors get a swatch + native picker *with the
  hex kept visible and editable* (paste-from-elsewhere is first-class); times of day get a time
  picker, the UI translating to whatever the backend stores.
- **Inheritance is explicit**: a value that falls back to a parent shows an *Override* checkbox
  and displays the effective inherited value ("inherited — global default (600 s)") instead of a
  mysteriously blank box. Checking the box seeds the field with the effective value. (→ Patterns:
  *Draft → publish & the config cascade*.)
- **Placeholders state format or meaning** (`DOMAIN\user`, `\\nas\backups\dashboards`), and units
  live in the label ("Interval (s)"), never guessed.
- **Validation happens at the field and again at the gate**, with messages that name the fix
  ("extends into the 60px dashboard border band — move it inside the bordered interior").

## 5. Communication of state and time

- **Timestamps**: store and transmit UTC (ISO-8601 with explicit designator), display in the
  viewer's local time zone, everywhere, with no exceptions a user must memorize.
- **Live data refreshes itself** — pages subscribe to change events and refetch — and where push
  isn't wired (rolling logs), offer an explicit auto-refresh toggle so reading and following are
  both possible. (→ Patterns: *Live entity push*.)
- **Status is shown with redundant encodings**: color + word + icon (a red dot *and*
  "Disconnected"), never color alone.
- **The UI never claims more than it knows**: a field nothing consumes says "reserved"; a value
  the server clears shows "—", not the stale last value.

## 6. Resilience (failure is a UI state, not an exception)

- **Errors arrive as dismissible toasts carrying the server's message plus context** ("The
  registration failed: …"), and the UI returns to a consistent state (checkbox reverts, list
  reloads).
- **Connection loss is handled invisibly where possible** (auto-reconnect, re-subscribe,
  re-identify) and visibly where it matters (a banner when showing last-good content).
- **Client and server versions self-heal**: clients detect a server upgrade on reconnect and
  reload themselves; no one walks the building pressing F5.
- **Connectivity is shown, never silent.** When the client loses its link to the backend, a
  stopped service must not read as "the buttons don't work." Surface it on two levels: an
  always-on status dot (green/amber/red + label) for ambient awareness, and a persistent banner
  the moment the link drops, cleared automatically when it returns. Detect on multiple channels
  so neither a transport quirk nor an idle page hides an outage — the live socket's own drop
  events, every HTTP response/failure, and a small periodic heartbeat that bounds worst-case
  detection. And **reconnect forever** with capped backoff: a fixed retry count silently strands
  the UI after a long outage, the exact moment recovery matters most. (→ Patterns: *Connectivity — detect / show / self-heal*.)

## 7. Selection-scoped list actions (the master-toolbar pattern)

A list whose every row carries its own button set stops scaling long before the list does: at
dozens of rows × half a dozen actions, the page renders *hundreds* of buttons, and the eye must
skip all of them to read the data. The cure is the file-manager model — **rows are calm data;
actions exist once, in a toolbar above the list, scoped to the current selection**:

- **Select by clicking anywhere on the row.** A leading checkbox column is the *affordance* — it
  announces that selection is the interaction model and shows which row holds it — but the whole
  row is the target; a ~14 px checkbox alone taxes every interaction. The selected row gets the
  accent treatment (tinted background, accent edge); selection is unmistakable from across the
  room, because the toolbar's actions will aim at it.
- **Single-select, honestly.** Most entity actions are single-target. Until a real bulk operation
  exists, one row is selected at a time — don't render checkboxes that secretly behave like
  radios *and* imply multi-select someday for free.
- **Entity-scoped buttons gray until a row is selected, and say why** — a disabled button's
  tooltip names the cure: "Select a playlist first", "This playlist has no published version to
  preview", "No drafts of this playlist exist". This is the gray-don't-hide philosophy (nav
  lock-outs, FR-111) and the no-dead-controls rule (§2) converging: a control that exists but is
  unavailable must always explain itself. Never *hide* an action because of state — a button that
  comes and goes teaches the user the UI is unstable.
- **Page-scoped actions never gray with the selection, and split by position.** "New X" / Import
  act on no entity; they sit **left-aligned** with primary styling. The selection-scoped actions
  push **right** (a flex spacer between). The horizontal gap does the separating that a mere
  divider can't: actions hugging the right edge read as "do this to the highlighted row" rather
  than blurring into the New button. Two button families, two scopes, two sides, one toolbar.
- **Don't ship a manual Refresh on a self-refreshing list.** A page wired to live model-change
  events (§5) reloads itself on every mutation, local or remote; a Refresh button there is
  vestigial reassurance. The genuine lost-notification fallback is a browser reload or
  re-navigation — not a button that mostly fires no-op reloads. Keep manual/auto refresh only on
  surfaces push doesn't cover (rolling logs). (→ Patterns: *Live entity push*.)
- **Selection is held by id and re-resolved every refresh.** These pages reload themselves on
  live model changes; the selection must survive that — and if another admin deleted the selected
  entity, the toolbar grays instead of aiming at a ghost. The page's own delete clears selection
  explicitly.
- **Accept the named trade-off**: Delete moves away from the thing it deletes, trading proximity
  for calm. The compensations are mandatory, not optional — a strongly highlighted selected row,
  and confirm dialogs that *name the entity* ("Delete playlist 'X'?").

(Pilot implementation: the authoring UI's Playlists page; the pattern is the standard for any
management list whose rows would otherwise carry three or more actions.)

## 8. Working method for polish passes

1. **Bound the pass** — pick one dimension (motion, inputs, feedback) per pass; never "make it
   nicer" open-endedly.
2. **Fix classes, not instances** — a shared component/CSS rule (color field, scroll region,
   pressed state) upgrades every page at once and keeps future pages consistent for free.
3. **Verify in a real browser** — feel claims need a live check; automation verifies mechanics
   (states toggle, values round-trip), a human verifies feel. Beware automation blind spots
   (e.g., test browsers run with relaxed autoplay policy).
4. **Document the decision where the next person will trip on it** — a load-bearing CSS line gets
   a comment saying *why* it exists, or someone "simplifies" it away.

## Finish checklist (use in review)

- [ ] Scrolling: steady pacing, no hover strobing, no layout shift, chrome fixed, sticky headers
- [ ] Every button: pressed state; every action: success/error feedback; progress where >1 s
- [ ] Every panel/mode: Cancel + click-away; destructive actions confirm with consequences
- [ ] No free-text field whose valid values the user can't guess; units in labels
- [ ] Lists with 3+ row actions use the selection toolbar; disabled actions carry why-tooltips
- [ ] Inherited values show Override + effective value
- [ ] Timestamps local; empty/loading/error states designed; status color + word
- [ ] Errors show server message; reconnects self-heal; reduced-motion honored

---

## Appendix: Design System (tokens + component reference)

The rules above describe *behavior*; this appendix is the *look* they ride on — the design tokens and
the component classes. It is the documentation of one stylesheet (`src/web/.../styles.css`): plain CSS
custom properties and utility/component classes, **no framework or preprocessor lock-in**. To reuse:
copy the stylesheet, keep the class names (the finish-guide rules assume them — `.scroll-region`,
`tr.selectable`, `button.disclosure`), and re-skin by editing `:root`.

### Tokens

**Color** — a three-tier dark elevation (`bg` < `surface` < `surface-2`), one accent, hairline borders,
three semantic states. The whole system is "dark surface + 1px border + one accent + status color."

| Token | Value | Role |
| --- | --- | --- |
| `--bg` | `#14181d` | app background (darkest tier) |
| `--surface` | `#1b2127` | cards, tables, modals, drawers (tier 2) |
| `--surface-2` | `#222a32` | buttons, badges, **row hover**, inset panels (tier 3) |
| `--border` | `#2c343c` | every hairline border and divider |
| `--accent` | `#4f9cf0` | primary actions, focus ring, selection, links |
| `--text` | `#e8eaed` | body text |
| `--muted` | `#9aa0a6` | secondary text, labels, table headers |
| `--ok` / `--warn` / `--fault` | `#34a853` / `#fbbc04` / `#ea4335` | semantic status (always paired with a word/icon — §5) |

**Spacing** — one scale: **4 / 6 / 8 / 12 / 16 / 24 px** for padding and gaps (§3 "one spacing scale").
**Radius family** — `4px` inputs/buttons, `6px` cards/drawers, `8px` modals, `10px` pill badges, `50%`
dots. **Typography** — body `14px 'Segoe UI'`; `h1` weight 300 / 24px, `h2` weight 400 / 17px (deliberately
light headings); labels 12px `--muted`; monospace `Consolas 12px` for code, textareas, and `pre.mono`.

### Component classes

| Class | Purpose | Realizes |
| --- | --- | --- |
| `.page` / `.page.fill` + `.scroll-region` | Page shell. `.fill` makes the page a viewport-height flex column with **fixed chrome**; only `.scroll-region` scrolls (overscroll contained, scrollbar gutter reserved, `table.grid th` sticky). | §1, §3 |
| `table.grid` | Standard list table: uppercase muted headers, hairline rows, `--surface-2` hover. | §3 |
| `tr.selectable` / `tr.selected` / `.sel-col` | Selection-scoped rows: whole row is the target, leading checkbox is the affordance, selected row gets the accent tint + an inset accent edge on the first cell. | §7 |
| `button.disclosure` + `.chev` | Expander control: chevron points right, rotates 90° and turns accent when `.open` — the trigger visibly owns the section it reveals. | §3 |
| `input` / `select` / `textarea` / `label` | Forms: dark field (`#11151a`), border goes accent on focus; block 12px muted labels; mono textareas. | §4 |
| `button` (+ `.primary` / `.danger` / `.small`) | Buttons. Universal **press feedback** (`:active` → `translateY(1px) scale(.97)` + brighten); hover lifts the border to accent (solid `.primary` brightens instead, since its border == its fill); `:disabled` → 45% opacity. | §2 |
| `.badge` (+ `.ok` / `.warn` / `.fault` / `.accent`) · `.dot` | Status pills and the connection/state dot (color **+** word/icon). | §5 |
| `.card` · `.toolbar` · `.row` · `.spacer` | Containers. `.toolbar` is the page action bar (flex, wraps); `.spacer` (`flex:1`) is the split that pushes selection-scoped actions right of page-scoped ones. | §7 |
| `.modal-backdrop` + `.modal` (+ `.actions`) · `.drawer` | Overlays. Backdrop fades, dialog pops in (opacity/transform only, never gating input); modal caps at 720px / 85vh and scrolls; `.actions` right-aligns; `.drawer` is the inline expandable panel. | §1, §2 |
| `pre.mono` | Wrapped, scrollable monospace block for logs/JSON. | — |

**Global feel (no class needed):** theme-matched scrollbars; `:focus-visible` accent ring (keyboard
only, never on mouse click); `html.is-scrolling` suppressing hover/transition repaints during scroll
(the class is toggled *outside* the framework — §1); a `.12s` opacity-only page-in; and a
`prefers-reduced-motion` kill switch that drops all animation/transition. The only JS-coupled pieces
are `is-scrolling` (app shell toggles it) and the in-flight progress bar (HTTP-interceptor driven, §2);
everything else is pure CSS.

### Reusable components & services (the "fix classes, not instances" catalog)

The other half of consistency-for-free (§8): shared **components** for domain inputs and shared
**services** for cross-cutting concerns, so a new page inherits the right behavior by wiring, not
re-implementing. The FDP set, as a starting inventory (Angular standalone, but the *roles* transfer):

| Component | Role |
| --- | --- |
| `color-field` | Color input — swatch + native picker **with the hex kept visible/editable** (paste-first). §4. |
| `path-field` | Server-filesystem path picker (file/folder mode) — text field + a "…" modal that browses the *host*. §4. |
| `area-picker` | Pick-or-create combo box over a shared label pool, **push-refreshed** (no GET-per-open). `[(value)]` + a `committed` output for commit-on-settle. §4. |
| `area-filter` | Compact multi-select dropdown for list filtering; hides itself when there's nothing to choose between. §7. |
| `clone-modal` | "Copy one published version into a new entity at V1" dialog, shared by every versioned-entity list. |
| `nav-icon` | The icon set as **inline SVG** (no webfont/CDN) — kiosk reliability and no sanitization surface. |

| Service | Role |
| --- | --- |
| `toast` | Error/info/success queue; `error(err)` **unwraps the server's message + references** from an `HttpErrorResponse`. §2, §6. |
| `changes` | Live model-change channels — `of(area): Observable<void>`; SignalR `/hubs/live`; **reconnect-forever** capped backoff. (→ Patterns: *Live entity push*.) §5. |
| `connection-status` | One `connecting \| online \| reconnecting` signal driving the dot + banner. (→ Patterns: *Connectivity*.) §6. |
| `version-watch` | An HTTP interceptor reading a build-stamp header → reload banner on skew; also feeds `connection-status`. §6. |
| `loading` | The global in-flight progress bar — ref-counted interceptor with a **250 ms anti-flicker** threshold. §2. |
| `draft-session` | Per-tab editing lock + liveness pings, journal/undo/redo/publish. (→ Patterns: *Draft → publish*.) |
| `user` | Caches the current identity + groups; exposes the signals the **gray-don't-hide** nav reads. §7. |
| `api` | One typed wrapper over the management API surface. |

**App shell** (the page frame each route loads into): group-aware nav with **gray-don't-hide** lockouts;
the status dot + connection/update/broadcast banners + toast host; two HTTP interceptors (`loading`
progress and `version` skew/connectivity); an ~8 s heartbeat (raw `fetch`, so it bypasses the progress
bar); **route preloading** (`PreloadAllModules`) plus a chunk-load-failure recovery probe (the
dead-lazy-route fix, §2); and the is-scrolling guard. New pages get all of it for free.

### The stylesheet (copy-ready)

The entire visual system above is **one file** — copy it, keep the class names the rules assume, and
re-skin by editing `:root`. Framework-free plain CSS; the only JS-coupled hooks are the `is-scrolling`
class (the app shell toggles it) and the progress bar (an HTTP interceptor), both noted above.

> **Source of truth:** this block mirrors `src/web/projects/authoring/src/styles.css` in the FDP repo
> (which carries a matching "keep in sync" header). If the tokens or shared component classes there
> change, update this block to match.

```css
/* Global dark theme and shared widgets (tables, forms, buttons, cards, modals). */

:root {
  --bg: #14181d;
  --surface: #1b2127;
  --surface-2: #222a32;
  --border: #2c343c;
  --accent: #4f9cf0;
  --text: #e8eaed;
  --muted: #9aa0a6;
  --ok: #34a853;
  --warn: #fbbc04;
  --fault: #ea4335;
}

html, body { height: 100%; }
body { margin: 0; background: var(--bg); color: var(--text); font: 14px 'Segoe UI', sans-serif; }

/* Theme-matched scrollbars: stock light-gray Windows bars read as unfinished in a dark UI. */
* { scrollbar-width: thin; scrollbar-color: #3a444e transparent; }
::-webkit-scrollbar { width: 10px; height: 10px; }
::-webkit-scrollbar-track { background: transparent; }
::-webkit-scrollbar-thumb { background: #3a444e; border-radius: 5px; border: 2px solid var(--bg); }
::-webkit-scrollbar-thumb:hover { background: #4a565f; }
::-webkit-scrollbar-corner { background: transparent; }

/* Keyboard focus: visible accent ring for tab navigation (mouse clicks don't trigger it). */
button:focus-visible, input:focus-visible, select:focus-visible, textarea:focus-visible, a:focus-visible {
  outline: 2px solid var(--accent); outline-offset: 1px;
}

/* Route content settles in rather than popping. Cheap (opacity only), short, ease-out. */
.page { animation: fd-page-in .12s ease-out; }
@keyframes fd-page-in { from { opacity: .5; } to { opacity: 1; } }

/* Decorative motion off for users who asked their OS for that. */
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after { animation: none !important; transition: none !important; }
}

h1 { font-weight: 300; font-size: 24px; margin: 0 0 16px; }
h2 { font-weight: 400; font-size: 17px; margin: 24px 0 10px; }
a { color: var(--accent); text-decoration: none; cursor: pointer; }
a:hover { text-decoration: underline; }

.page { padding: 24px 28px; max-width: 1500px; }
.muted { color: var(--muted); }
.small { font-size: 12px; }

/* Fixed-chrome list pages: the page fills the viewport and heading/toolbars stay put;
   only .scroll-region (the list) scrolls, with a sticky table header. */
.page.fill { height: 100vh; display: flex; flex-direction: column; box-sizing: border-box; overflow: hidden; }
.page.fill > .scroll-region { flex: 1; min-height: 0; overflow-y: auto; }
/* Scroll-feel hygiene: stable scrollbar gutter (no layout shift) + contained overscroll. */
.scroll-region { overscroll-behavior: contain; scrollbar-gutter: stable; }
.scroll-region table.grid th { position: sticky; top: 0; background: var(--surface); z-index: 2; }

/* While a scroll is IN PROGRESS, suppress per-frame hover repaints (rows streaming under a
   stationary cursor repaint a highlight every frame — the classic table-scroll jank). The
   .is-scrolling class is toggled outside the framework by the app shell; hover returns on stop. */
html.is-scrolling table.grid tbody tr:hover { background: transparent; }
html.is-scrolling button { transition: none; }

/* Sortable column headers */
th.sortable { cursor: pointer; user-select: none; }
th.sortable:hover { color: var(--text); }
.sort-arrow { opacity: .8; font-size: 10px; }

/* Disclosure buttons: chevron points right collapsed, rotates down open; open stays accent-colored
   so the control visibly owns the section it revealed. */
button.disclosure .chev { display: inline-block; transition: transform .12s ease; margin-right: 5px; font-size: 9px; }
button.disclosure.open .chev { transform: rotate(90deg); }
button.disclosure.open { border-color: var(--accent); color: var(--accent); }

/* Tables */
table.grid { border-collapse: collapse; width: 100%; background: var(--surface); border: 1px solid var(--border); }
table.grid th, table.grid td { text-align: left; padding: 8px 12px; border-bottom: 1px solid var(--border); vertical-align: top; }
table.grid th { color: var(--muted); font-weight: 600; font-size: 12px; text-transform: uppercase; letter-spacing: .4px; }
table.grid tr:last-child td { border-bottom: none; }
table.grid tbody tr:hover { background: var(--surface-2); }

/* Selection-scoped list rows (§7): whole row selects; the leading checkbox is the affordance;
   the selected row gets the accent tint + edge so the toolbar's target is plain. */
table.grid tr.selectable { cursor: pointer; }
table.grid tr.selectable input[type=checkbox] { cursor: pointer; }
table.grid tr.selected td { background: rgba(79, 156, 240, .14); }
table.grid tr.selected:hover td { background: rgba(79, 156, 240, .18); }
table.grid tr.selected td:first-child { box-shadow: inset 3px 0 0 var(--accent); }
table.grid td.sel-col, table.grid th.sel-col { width: 34px; }

/* Forms */
input, select, textarea {
  background: #11151a; color: var(--text); border: 1px solid var(--border);
  border-radius: 4px; padding: 6px 8px; font: inherit; box-sizing: border-box;
}
input:focus, select:focus, textarea:focus { outline: none; border-color: var(--accent); }
textarea { font-family: Consolas, 'Courier New', monospace; font-size: 12px; }
label { display: block; margin: 10px 0 4px; color: var(--muted); font-size: 12px; }
input[type="checkbox"] { width: auto; }

/* Buttons */
button {
  background: var(--surface-2); color: var(--text); border: 1px solid var(--border);
  border-radius: 4px; padding: 6px 14px; font: inherit; cursor: pointer;
  transition: transform 60ms ease, filter 120ms ease, border-color 120ms ease;
}
button:hover:not(:disabled) { border-color: var(--accent); }
/* Solid buttons can't show a border-color change (border == background): brighten instead. */
button.primary:hover:not(:disabled) { filter: brightness(1.15); }
/* Press feedback on every enabled button: a visible push-down so a click never feels dead. */
button:active:not(:disabled) { transform: translateY(1px) scale(.97); filter: brightness(1.25); }
button:disabled { opacity: .45; cursor: default; }
button.primary { background: var(--accent); border-color: var(--accent); color: #0c1116; font-weight: 600; }
button.danger { border-color: #6e2b26; color: #f28b82; }
button.danger:hover:not(:disabled) { border-color: var(--fault); }
button.small { padding: 3px 8px; font-size: 12px; }

/* Badges and dots */
.badge { display: inline-block; padding: 2px 8px; border-radius: 10px; font-size: 11px; font-weight: 600; background: var(--surface-2); border: 1px solid var(--border); }
.badge.ok { color: var(--ok); border-color: var(--ok); }
.badge.warn { color: var(--warn); border-color: var(--warn); }
.badge.fault { color: var(--fault); border-color: var(--fault); }
.badge.accent { color: var(--accent); border-color: var(--accent); }
.dot { display: inline-block; width: 10px; height: 10px; border-radius: 50%; margin-right: 6px; vertical-align: middle; }

/* Cards / panels / layout */
.card { background: var(--surface); border: 1px solid var(--border); border-radius: 6px; padding: 16px; }
.toolbar { display: flex; gap: 8px; align-items: center; flex-wrap: wrap; margin-bottom: 14px; }
.row { display: flex; gap: 16px; align-items: flex-start; }
.spacer { flex: 1; }

/* Modal — backdrop fades, dialog pops in (opacity/transform only; never gates input). */
.modal-backdrop { position: fixed; inset: 0; background: rgba(0,0,0,.55); display: flex; align-items: center; justify-content: center; z-index: 50; animation: fd-fade-in .12s ease-out; }
.modal { background: var(--surface); border: 1px solid var(--border); border-radius: 8px; padding: 20px 24px; min-width: 420px; max-width: 720px; max-height: 85vh; overflow: auto; animation: fd-pop-in .14s ease-out; box-shadow: 0 12px 40px rgba(0,0,0,.45); }
@keyframes fd-fade-in { from { opacity: 0; } to { opacity: 1; } }
@keyframes fd-pop-in { from { opacity: 0; transform: translateY(6px) scale(.985); } to { opacity: 1; transform: none; } }
.modal h2 { margin-top: 0; }
.modal .actions { display: flex; gap: 8px; justify-content: flex-end; margin-top: 18px; }

/* Drawer */
.drawer { background: var(--surface); border: 1px solid var(--border); border-radius: 6px; padding: 14px 16px; margin: 8px 0 16px; }

pre.mono { font-family: Consolas, 'Courier New', monospace; font-size: 12px; background: #11151a; border: 1px solid var(--border); border-radius: 4px; padding: 10px; white-space: pre-wrap; word-break: break-word; max-height: 320px; overflow: auto; }
```
