# How a Web UI Should Perform and Feel — Finish Standard

**Purpose.** A reusable standard for judging and building "finished-feeling" web UIs, written for
engineers who know when software *works* but want shared vocabulary and concrete rules for when it
*feels done*. Grounded in the Facility Dashboard Platform's authoring UI, where each rule cites the
implementation that realizes it — but the rules are general and intended for future design/build
sessions on any project.

The single organizing idea: **a finished UI never leaves the user guessing** — not about whether a
click registered, not about what a field expects, not about why something moved. Every rule below
is that idea applied to one surface.

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
  app-wide; long server jobs get a busy indicator ("backup running…").
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
  mysteriously blank box. Checking the box seeds the field with the effective value.
- **Placeholders state format or meaning** (`DOMAIN\user`, `\\nas\backups\dashboards`), and units
  live in the label ("Interval (s)"), never guessed.
- **Validation happens at the field and again at the gate**, with messages that name the fix
  ("extends into the 60px dashboard border band — move it inside the bordered interior").

## 5. Communication of state and time

- **Timestamps**: store and transmit UTC (ISO-8601 with explicit designator), display in the
  viewer's local time zone, everywhere, with no exceptions a user must memorize.
- **Live data refreshes itself** — pages subscribe to change events and refetch — and where push
  isn't wired (rolling logs), offer an explicit auto-refresh toggle so reading and following are
  both possible.
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

## 7. Working method for polish passes

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
- [ ] Inherited values show Override + effective value
- [ ] Timestamps local; empty/loading/error states designed; status color + word
- [ ] Errors show server message; reconnects self-heal; reduced-motion honored
