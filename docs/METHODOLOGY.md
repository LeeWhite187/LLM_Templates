# Project Methodology

**Purpose:** This document describes how design-oriented projects are conducted between the user (the engineer) and Claude. It is the procedural authority for that work, complementing the structural authority of `SPEC_TEMPLATE.md`.

**Audience:** Claude, when conducting design discussions. The user, as a reference for the working agreement. Future contributors who pick up the practice.

**Status:** Living document. Revised out-of-band — not within project conversations. Project conversations may surface ideas for revision; those ideas are noted by the user and applied later, not treated as in-band edits.

---

## How this document reaches Claude

This document is provided to Claude at the start of design-oriented conversations, manually pasted by the user. A standing instruction in the user's Claude settings prompts Claude to ask for this document before substantive design discussion begins. The mechanism is intentionally manual — Claude does not autonomously fetch the document, and the user controls when and where it is loaded.

The companion document `SPEC_TEMPLATE.md` is provided when a project spec is about to be produced or revised. It is not needed for early design discussion that has not yet reached the spec-drafting stage.

A future `TESTING_METHODOLOGY.md` will exist as a sibling document, loaded when test work begins. It is not needed for design conversations and is not loaded by default.

---

## 1. The thesis: design mitigates the liability of code

Code is a liability. Every line of code is a thing that must be maintained, understood by future readers, refactored when the world changes around it, and reasoned about every time it is touched. Code has a cost basis you keep paying — at minimum, the cost of comprehension; at worst, the cost of reverse-engineering a system whose original intent has been lost.

Design is the cheap place to find that a feature is wrong, that an interface is awkward, that an edge case will bite. A bug found in design costs a paragraph. The same bug found in code costs a refactor. Found in production, it costs trust. The asymmetry is not subtle, and it compounds: design errors that survive into implementation tend to multiply into the implementations that depend on them.

The conclusion is straightforward and non-negotiable: **design exhaustively before implementing**. The upfront cost is real but bounded. The downstream savings are larger and compounding. A project that spends a week on careful design and a month on confident implementation is faster than one that spends two weeks on a quick design and three months refactoring.

This is not waterfall. Designs are revised. Specs evolve. New information surfaces during implementation that the design did not anticipate. But revision happens in the spec — where it is cheap — and propagates to code deliberately, not the other way around. We do not let implementation drift discover the design retroactively. When the implementing agent surfaces friction or finds a gap, the response is to revise the spec first and update the code second, not to patch the code and leave the spec stale.

The phrase that captures the stance: **design mitigates the liability of code.** Less code, written more deliberately, is the goal. The spec is where the deliberation happens.

---

## 2. Project order: top-down, requirements-first

A project proceeds in roughly this order. The order is not a rigid waterfall — earlier sections may be revisited as later sections surface new information — but the *direction* of work is consistently top-down.

**Requirements solicitation.** Background, problem statement, scope, goals. What world does this system live in? What are the people involved trying to do? What does success look like? This stage is conversational and exploratory. Requirements are extracted from the user's understanding of the problem, not invented by Claude.

**User requirements.** The goals, enumerated. Each user requirement is a single sentence describing what a user shall be able to do. Platform- and implementation-agnostic. These are the testable expressions of the goals.

**Functional requirements.** What the system shall do, expressed as system behavior. Derived from user requirements but more granular and more system-facing. Still platform- and implementation-agnostic.

**Non-functional requirements.** Quality attributes: performance, reliability, security, observability, regulatory compliance. These often surface in parallel with functional requirements rather than after them.

**Constraints and technology decisions.** Now the spec becomes opinionated: language, runtime, database, library dependencies. This is the boundary where platform-agnostic requirements become platform-specific commitments.

**Architecture and data model.** The shape of the system. Tiers, components, entities, schemas, identifiers, retention.

**Solution structure and interfaces.** The project graph — what library boundaries are drawn, what kinds of types belong in each, what reference rules govern. The principal interfaces between components.

**Protocols, flows, API surface.** The detailed contracts: bespoke protocols, narrated data flows, enumerated endpoints.

**Implementation.** Last. Implementation follows the spec. The spec is the input; the code is the output. If implementation discovers a design gap, the spec is revised first.

A few things this order rules out, deliberately:

- **No bottom-up design.** We do not start by sketching code and reasoning upward to requirements. Bottom-up design tends to produce systems shaped by what is easy to build rather than what users need.
- **No flashy quick-pass POCs unless requested.** A POC may be useful occasionally — to verify a library works as expected, to confirm a performance assumption — but it is requested explicitly, not produced reflexively. Demonstrating something runs is not the same as demonstrating it solves the problem.
- **No "let me just sketch some code to think about it."** Code is not a thinking tool. Specs are the thinking tool. If a question is hard to reason about in the spec, it is harder still to reason about in code.

---

## 3. The two-agent pattern

Project work happens across two Claude instances with distinct, complementary roles. The user is the connecting tissue between them.

**Web Claude — the design partner.** The instance the user converses with in a chat interface (web, desktop, or mobile). Holds the long context of the design conversation. Produces and revises the spec. Reasons about edge cases. Solicits requirements. Asks clarifying questions. Suggests alternatives. Maintains internal consistency across the spec as it evolves.

**Claude CLI — the implementation partner.** A separate instance running in a terminal, working in the project's repository. Reads the spec as authoritative input. Writes code, runs tests, surfaces issues. Reports findings, gaps, ambiguities, and suggested refinements back to the user.

**The user — courier and arbiter.** The user is the only entity with full context across both instances. They carry the spec from web chat into the repo for the CLI to consume. They carry the CLI's questions, friction reports, and discoveries back into the web chat. They decide whether a friction point warrants a spec revision (and bring it to design discussion) or a localized implementation choice (and let the CLI proceed).

This pattern has a few properties worth naming:

- **The spec is the handoff artifact.** Not the chat history, not screenshots, not verbal recollection. If something matters, it is in the spec. If it is not in the spec, it does not yet officially exist.
- **The spec and its referenced template revision are self-sufficient as a pair.** A CLI reading the spec cold, with no chat history, should be able to implement from the spec together with the template revision the spec adheres to (named in the spec's `Template Revision` title-block field). The chat history is not required; the template is. This shapes how the spec is written: explicit, declarative, concrete. Anything that exists only in the design conversation has not yet been captured properly.
- **The two agents do not communicate directly.** No autonomous handoff, no shared memory. The user is always the explicit connection. This is intentional — it keeps the spec as the durable artifact and prevents context drift between the two instances.

---

## 4. The spec as the contract between agents

The project spec is the authoritative source of truth for implementation. This is a stronger commitment than "the spec is a useful reference." A few rules follow from it:

**The implementing CLI does not improvise around ambiguity.** If the spec is unclear, contradictory, or silent on a point that affects implementation, the CLI raises that as friction — it does not pick an interpretation and proceed. The user carries that friction back to the design conversation.

**Implementation discoveries trigger spec revision before code change.** When the CLI finds that a design decision was wrong, incomplete, or impractical, the response sequence is: (1) the user surfaces the finding to web chat, (2) web chat and the user discuss the finding and update the spec, (3) the user carries the revised spec back to the CLI, (4) the CLI updates the code to match. The reverse order — code first, spec retrofitted — is not allowed. It produces drift that compounds.

**The spec describes the system as intended.** Not as it currently exists, not as it might someday become. The spec is the design commitment at the current revision; the code is what realizes it. When the two diverge, the divergence is either a bug (code does not match spec) or an opportunity for spec revision (the spec no longer reflects the intended design). It is never just left as a divergence to live with.

**Specs are versioned by timestamp and revision log.** The Last Updated field in the spec's title block matches the timestamp of the most recent Revision Log entry. Every substantive change to the spec earns a Revision Log entry describing what changed and why. This makes the spec's history auditable — a future reader can reconstruct the design's evolution.

---

## 5. Congruency-check Open Items

When the spec is revised mid-implementation, the implementing CLI may have built code against an earlier version of the spec. The risk: the code reflects the old design, and the divergence is silent until something breaks.

To guard against this, the design conversation deliberately plants **congruency-check Open Items** in the spec at points where revision affects already-built code. These are not bugs. They are audit prompts — items written specifically to direct the implementing CLI to verify that the built code matches the revised spec.

**Format.** A congruency-check OI follows the standard OI format and lives in §13 of the spec. Its narrative names what to check:

> **OI-NN — CLI congruence: [specific component or behavior].** [Brief description of what changed in the spec and what the CLI should verify in the code.] Confirm that [component X] reflects [decision Y] from §[Z]. Close this item with a brief resolution note when verification is complete.

**Lifecycle.** A congruency-check OI is opened in the design conversation when a spec revision affects existing code. The user carries the revised spec to the CLI. The CLI works through the open congruency checks, verifies or corrects the code, and reports back. The user closes the OI in the spec with a resolution note ("Closed: verified, no code change required" or "Closed: code updated to match §6.3"). Closed items remain in §13 with their resolution; they are never deleted, and their numbers are never reused.

**When to plant them.** Not every spec revision needs one. A small clarification or wording tweak does not. A change to schema, interface signatures, protocol semantics, identifier formats, or library boundaries does — anything where the implementation might already encode an assumption that the revision invalidates.

**Why this is a named pattern rather than ad-hoc bookkeeping.** Because spec revisions accumulate, and silent drift between spec and code is the failure mode this pattern exists to prevent. Making the audit explicit and tracked means it does not depend on memory or vigilance.

---

## 6. Spec evolution: what triggers a revision

The spec is a living document during a project's design and implementation phases. Revisions happen for several reasons, each with a different shape:

**Design-conversation revision.** Web chat and the user discuss a topic and reach a new conclusion. The spec is updated to reflect it. Items are added, withdrawn, or reworded. Sections are restructured. A Revision Log entry captures the change and its rationale.

**Implementation-discovery revision.** The CLI finds friction and the user brings it back to web chat. The design conversation evaluates the friction and decides whether to revise the spec. If yes, the spec is updated and a congruency-check OI is planted if existing code is affected. If no, the friction is resolved by clarifying the spec without changing the design (the CLI's interpretation was wrong; the spec was always correct, just unclear).

**Decision revisitation.** A Key Decision is reopened because new information has surfaced — a library has a bug, a constraint has changed, a tradeoff is now better understood. The KD is revised in place; alternatives and consequences are updated; the Revision Log notes what shifted.

**Maintenance sweep.** A periodic pass through the spec to bring it back into alignment with the forward-looking-reference principle (§9). The sweep tombstones closed items, replaces stale body prose with pointers to the current state, and validates that what remains describes current intent. Sweeps are triggered by accumulated drift, by a substantial design revision, or by reports from downstream consumers (other Claude chats, the CLI, external readers) that the spec is hard to derive current state from. A sweep is a substantive change and earns a Revision Log entry.

A few things that are *not* spec revisions:

- **Trivial edits.** Typo fixes, formatting cleanups, and rewording for clarity do not earn Revision Log entries and do not bump the spec's Revision number. They happen silently. The threshold is whether the change conveys anything to a reader who has read the previous version.
- **Code-level decisions made during implementation.** Choosing a variable name, picking between two equally valid algorithms, selecting a logging format — these are implementation choices, not design decisions, and live in the code, not the spec.

The Revision Log is the auditable history of *why the design is what it is*. Future readers — including future-self — should be able to read the log and understand how the spec arrived at its current state.

---

## 7. Structural transforms and downstream references

A structural transform — removing a field from the title block, reordering entries in the revision log, renaming an identifier scheme, migrating a spec from one template revision to another — can have semantic consequences in content that the transform does not directly edit. The transform's mechanical scope is bounded; its semantic reach is not.

Two examples make this concrete. First, removing the `Last Updated` field from the title block invalidates any prose elsewhere in the spec that references that field — such as a convention saying "the Last Updated field shall match the most recent revision log entry." Second, reversing the order of revision log entries invalidates any prose inside those entries that uses positional language ("the entry above") or that references other entries by bare timestamp — both fragile under reordering.

The principle: **a structural transform must include a detect-and-report obligation for downstream references to the changed elements.** The transform does not silently edit content outside its scope — that would be improvisation, and improvisation is what the methodology guards against. It also does not let stale references ship invisibly — that would be silent drift, and silent drift is what congruency-check Open Items exist to prevent. Instead, it scans for downstream references, surfaces them to the operator, and the operator decides whether to fix them, leave them, or escalate them to a wider revision.

This obligation applies to any migration instruction — between template revisions, between methodology revisions, between identifier schemes. When designing a migration instruction, ask: *what does this transform touch that lives outside its permitted region?* The answer is the list of detect-and-report checks the instruction must include.

A useful framing: the instruction's "what NOT to change" boundary and its "detect and report" obligation are complementary. The first prevents the agent from improvising; the second prevents the agent from being negligent. A migration that has the first without the second produces silent stale references. Both are required.

---

## 8. Item identifier discipline

Items in the spec (UR, FR, NFR, KD, OI) are identified with a type prefix and a stable number. The detailed rules are in the Conventions section of `SPEC_TEMPLATE.md` at the revision the spec adheres to. The methodology adds three pieces of context:

**Numbers are addresses.** When the spec says "as required by FR-12," that reference must continue to point at the same requirement across all revisions of the spec. Renumbering would invalidate every reference to the renumbered item — including references in code comments, commit messages, ticket trackers, and external documents. Therefore: numbers are stable, never reused, never reassigned.

**Terminal items remain in place as tombstones.** An item that reaches a terminal disposition — withdrawn, superseded, closed, cancelled — is not deleted from its section. It remains in place with its identifier and a disposition tag in the heading: `### FR-07 — (Withdrawn)`, `### KD-03 — (Superseded by KD-09)`, `### OI-12 — (Closed; resolved by §6.4)`. The body of a tombstoned item is empty or carries at most a one-sentence pointer to where the resolution lives. Full rationale for the disposition belongs in the Revision Log entry that effected it, not in the tombstone's body. This rule applies universally across item types (UR, FR, NFR, KD, OI); the same discipline keeps the spec scannable for current state regardless of which item type was retired.

**New items always take the next number.** New items take `max(existing_number) + 1` for their type, where the maximum includes all tombstoned entries. This rule is mechanical — the implementing CLI can rely on it when generating new identifiers.

The discipline is small but load-bearing. A spec where numbers wander is a spec that cannot be trusted as a reference. A spec whose terminal items carry their old bodies is a spec whose keyword search returns ghosts.

---

## 9. Forward-looking reference, not a history book

The spec describes the system as currently designed. It does not describe how the system used to be designed, what alternatives were considered before the current design was reached, or what content was previously true and is no longer. Content that does not describe current intent belongs in the Revision Log, not in the body.

This is a principle, not a rule of housekeeping. Downstream consumers of the spec — other Claude chat sessions, the implementing CLI, future readers including future-self — derive current state by reading the body and searching it for relevant content. Every sentence in the body that does not describe current intent is a sentence that can be hit by a keyword search and lead the consumer to confusion. Closed Open Items with their original bodies still present are the most common offender; stale prose in section bodies is another; old terminology that the spec has since revised away is a third. The principle directs all of them: out of the body, into the log (or out entirely, with a Revision Log entry recording the removal).

The principle has several concrete consequences elsewhere in the methodology and template:

- **Tombstoning** (see §8). Items at terminal disposition carry only their identifier and disposition tag; bodies collapse to at most a one-sentence pointer.
- **The maintenance sweep** (see §6). The periodic activity that operationalizes this principle — going through the spec and bringing it back into forward-looking shape when drift has accumulated.
- **The implementation guide convention** (see `IMPLEMENTATION_GUIDE_TEMPLATE.md`). Guides apply the same principle more strictly, with no revision log in their body at all, because consumer tolerance for sediment is lower than the design partnership's.

A useful test when deciding whether content earns its place in the body: *will a reader six months from now, searching this document for "how does X work today," benefit from this sentence?* If yes, it earns its place. If the sentence describes how X used to work, or why we don't do X anymore, the answer is no — and the sentence belongs in the Revision Log entry that effected the change, not in the body.

The principle applies to specs first and most directly. It applies to implementation guides with even more force, because guides are pure consumer references and have no equivalent of the spec's "history is preserved in the log" affordance. It applies less to working documents like the methodology and templates, which legitimately carry their own evolution history because their audience benefits from it.

---

## 10. Diligence is visible

A spec produced under this methodology demonstrates *thinking* in two ways: through the content of its sections, and through the explicit handling of sections that do not apply.

A section that does not apply to the project at hand is not deleted from the spec. It remains in place with a brief design statement explaining *why* the section was considered and found inapplicable. A bare "not applicable" is insufficient; the explanation is itself a small piece of design reasoning that confirms the consideration happened.

This rule serves the methodology in two ways. First, it prevents the silent omission of considerations that should have been made. Second, it makes the spec readable as a checklist of what was thought about, not just what was decided. A reader (human or CLI) can confirm, section by section, that each topic was considered — even when the consideration was "this does not apply, here is why."

The principle generalizes beyond unused sections: deliberation is part of the deliverable. Saying "we considered X and decided no, because Y" is more honest and more useful than silently omitting X.

---

## 11. Audience and voice

Specs produced under this methodology have two audiences: the design partnership (the user and web Claude), and the implementing CLI. The voice is shaped by both.

**Declarative, present-tense, technical.** The spec describes the system as it is intended to be. "The system shall validate the request before accepting it." Not "the system will eventually validate" or "the system would validate if implemented." The intended-state framing is clearer to read and easier to verify against code.

**Concrete over aspirational.** "The session token shall be a 32-character base64-encoded random value, transmitted in an HTTP-only cookie" is useful. "The system shall use secure session management" is not. When detail matters, the spec carries detail; when detail is genuinely deferred, the spec says so explicitly (often as an Open Item).

**Spare with adjectives, generous with rationale.** "Robust," "scalable," "maintainable" tend to communicate enthusiasm rather than substance. The methodology prefers "the system shall handle [specific failure mode] by [specific behavior]." Rationale, on the other hand, is welcome — Key Decisions exist precisely so the reasoning behind a choice is preserved alongside the choice itself.

**Written for the implementing CLI as a primary reader.** This shapes the spec in concrete ways: explicit cross-references rather than allusions, full names rather than pronouns, the `Template Revision` field in the title block naming which revision of the template's conventions the spec adheres to. A CLI reading the spec, together with the referenced template revision, should be able to derive the rules of how to read the spec from those two documents alone.

---

## 12. The relationship between this document, the spec template, and the implementation guide template

This document and `SPEC_TEMPLATE.md` and `IMPLEMENTATION_GUIDE_TEMPLATE.md` are companion documents:

- **The methodology document is procedural.** How we work. The thesis, the project order, the two-agent pattern, the spec-as-contract rule, the congruency-check pattern, the forward-looking-reference principle, the discipline around revision and identifier stability.
- **The spec template is structural for the build side.** What an individual project spec looks like — the artifact the design partnership produces and the implementing CLI consumes.
- **The implementation guide template is structural for the use side.** What an individual library's implementation guide looks like — the artifact downstream consumers (other Claude chat sessions, you, external developers) consume to use the library.

The spec and the guide describe the same library from different angles. The spec is the design record (how and why the library is built). The guide is the use reference (how to consume the library successfully). They are anchored to each other: the guide names a specific spec revision via its `Spec` field, which makes guide-vs-spec drift detectable.

Project conversations refer to the methodology when discussing *how* to proceed. They refer to the spec template when shaping *what* the spec looks like. They refer to the implementation guide template only when producing or revising a library's implementation guide.

All three documents are maintained in a public GitHub repository and revised out-of-band — not within project conversations. A project conversation may surface ideas for revising any of them; those ideas are noted by the user and applied later, on the user's editing timeline. This keeps the standing documents stable across projects and prevents per-project drift.

---

*End of methodology.*
