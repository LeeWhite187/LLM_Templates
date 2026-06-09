# Spec Migration Instruction: Template Revision 2 → Revision 3

## Purpose

Migrate an existing project specification document from Template Revision 2 conventions to Template Revision 3 conventions. The structural changes from R2 to R3 are small — the spec's `Template Revision` field bumps from 2 to 3, and the §1.9 reference statement gets a minor wording update. The substantive change introduced in R3 — universal terminal-item tombstoning, in service of the forward-looking-reference principle — is **not** applied automatically by this migration. Tombstoning is a content-discipline activity (a "maintenance sweep" per `METHODOLOGY.md` §6) and requires the operator's judgment on which items are actually terminal. This migration surfaces tombstone candidates via detect-and-report; the operator applies them as a follow-up activity.

## Workflow

The user will paste this instruction along with the spec document to be migrated. Apply the structural transformations in §A and produce the complete updated document as your output. After the file, report the detect-and-report findings from §B so the operator can plan the follow-up sweep.

If anything in the spec is ambiguous against this instruction, stop and ask the user rather than guessing.

---

## A. Transformations to apply

### A.1 Title block changes

Update the `Template Revision` field value:

- From: `**Template Revision:** 2`
- To: `**Template Revision:** 3`

No other title block fields change in this migration.

### A.2 §1.9 Spec Conventions wording update

Under R2, the §1.9 reference statement enumerated a list of convention names. Under R3, the list grows to include the new conventions introduced in R3 (terminal item disposition, forward-looking reference). Replace the existing §1.9 content with:

```
### 1.9 Spec Conventions

This spec adheres to the conventions defined in `SPEC_TEMPLATE.md` at the revision identified by the `Template Revision` field in this spec's title block. See the Conventions section of that document for identifier rules, number stability, terminal item disposition (tombstoning), requirement language, table usage, revision consistency, intra-log cross-references, the forward-looking-reference principle, and other conventions governing this spec.

The conventions are not duplicated here. The template is the single source of truth for the rules a spec follows; this spec references the template revision it was authored under, and migrations to newer template revisions are deliberate (see `SPEC_RETROFIT_INSTRUCTION.md` or the relevant migration instruction).
```

The substance is unchanged — the spec still references the template for its conventions. The list of convention names is updated to include the R3 additions.

### A.3 Revision number bump and Revision Log entry

Bump the spec's `Revision` field (in the title block) by one and add a corresponding Revision Log entry. Use the next available integer.

Format for the entry:

```
### Revision NN

**Date:** YYYY-MM-DDTHH:MM:SSZ

Migrated to Template Revision 3 conventions. Updated `Template Revision` field in the title block from 2 to 3. Updated §1.9 reference statement wording to enumerate the conventions introduced in R3 (terminal item disposition, forward-looking reference). No content tombstoning was applied as part of this migration; see the detect-and-report findings for tombstone candidates to be addressed in a follow-up maintenance sweep.
```

Use the current ISO-8601 datetime for the Date field, or ask the user for the intended timestamp if a specific moment matters.

---

## B. Detect-and-report obligations

### B.1 Tombstone candidates among existing items

R3 introduces universal terminal-item tombstoning: items at terminal disposition (Withdrawn, Superseded, Closed, Cancelled) collapse their bodies to either nothing or a single sentence pointing to the resolution. The operator decides which items are actually terminal — this migration does not apply tombstoning, but it surfaces candidates so the operator can plan the sweep.

Scan all items in the spec (UR, FR, NFR, KD in their respective sections, OI in §13) and identify candidates that appear to be terminal but have not yet been tombstoned. Indicators include:

- The item's title contains `(withdrawn)`, `(closed)`, `(deferred)`, `(superseded)`, `(resolved)`, or similar parenthetical markers — but the body still carries substantive content beyond a single pointer sentence.
- The item's body contains closure language ("resolved by §X", "closed: ...", "deferred to v2", "see KD-NN", "no longer applicable", etc.) but the body is longer than one or two sentences.
- The item appears to describe a question, decision, or requirement that has been visibly resolved elsewhere in the spec.

For each candidate, report:
- The item's identifier (e.g., FR-07, OI-12, KD-03).
- The current title of the item.
- A brief excerpt or summary of the body content.
- A suggested terminal disposition (Withdrawn / Superseded / Closed / Cancelled), based on the indicators present.
- A suggested pointer for the tombstone body, if one is evident from the existing body content.

The operator reviews this list and applies tombstoning to confirmed terminal items in a follow-up sweep. The operator may also identify additional terminal items the detection missed and may reject candidates that are not actually terminal.

### B.2 Stale prose in section bodies

R3 elevates the forward-looking-reference principle to a named convention. Section bodies should describe the system as currently designed, not how it used to be.

Scan section bodies for prose that appears to describe historical state or superseded design. Indicators include:

- Phrases like "previously", "originally", "used to", "before we switched to", "in earlier versions of this spec", "the old approach was".
- Discussion of alternatives that were rejected (rather than chosen) — these belong in the relevant KD's "Alternatives considered" subsection, not in section bodies.
- References to features, modes, or behaviors that other parts of the spec indicate have been removed or superseded.

For each finding, report:
- The section and a brief excerpt.
- A note on why it appears stale (the indicator that triggered the flag).

The operator decides whether each finding warrants rewriting (in a follow-up sweep) or whether the prose is legitimately current despite the indicator.

### B.3 References to OIs in section bodies

R3 conventions discourage references to OIs from elsewhere in the spec (because OIs may tombstone, leaving the references rotted). Prefer references to KDs or section numbers.

Scan section bodies (outside §13) for inline references to Open Items (e.g., "see OI-12", "pending resolution of OI-18"). For each finding, report:
- The section and a brief excerpt.
- The OI(s) referenced.

The operator decides whether the reference should be rewritten to point at a KD or section instead. This is a soft recommendation; some OI references may be legitimate (e.g., "this section will be revised when OI-04 is resolved" is itself a current-state statement about an unresolved question).

---

## C. What NOT to change

- Do not apply tombstoning to any items as part of this migration. Tombstoning is a separate sweep activity and requires operator judgment.
- Do not modify item bodies (UR, FR, NFR, KD, OI) for any reason during this migration.
- Do not modify section content outside the title block, §1.9, and §14 (the Revision Log).
- Do not "improve" content, fix typos, or normalize formatting outside the scope of this migration.
- Do not add or remove items.
- Do not change cross-references in the body of the spec.

---

## D. When to stop and ask the user

Surface friction rather than guessing whenever:

- The spec's title block `Template Revision` field is not `2` (it might already be `3`, or might be older — neither case is handled by this instruction).
- The §1.9 reference statement has been substantially modified from the R2 baseline and the replacement risks losing project-specific content.
- The spec lacks §1.9 or §14 (the migration assumes both are present in standard form).
- Any aspect of the spec's structure is ambiguous against this instruction.

---

## E. Output

Produce the complete migrated spec as a single file. After the file, briefly state:

1. The new `Revision` number assigned to the spec.
2. Confirmation that `Template Revision` is now `3`.
3. Confirmation that the §1.9 reference statement was updated.
4. The complete list of detect-and-report findings from §B, organized by check (B.1 tombstone candidates, B.2 stale prose, B.3 OI references). If a check returned no findings, state that explicitly.
5. A recommendation that the operator plan a follow-up maintenance sweep to address the B.1 findings (and optionally B.2 and B.3).

Do not produce a diff or a summary in place of the file. The user needs the complete updated document to save.

---

*End of migration instruction.*
