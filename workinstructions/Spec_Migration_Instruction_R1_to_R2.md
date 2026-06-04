# Spec Migration Instruction: Template Revision 1 → Revision 2

## Purpose

Migrate an existing project specification document from Template Revision 1 conventions to Template Revision 2 conventions. This is a structural migration. Do not change the substance of the spec — only the title block, the §1.9 conventions section, and the §14 Revision Log, per the rules below. Detect-and-report obligations cover content that may be invalidated by structural changes but must not be edited directly.

## Workflow

The user will paste this instruction along with the spec document to be migrated. Apply the transformations described below and produce the complete updated document as your output. Do not summarize the changes; produce the full migrated document so the user can save it and replace the original.

If anything in the spec is ambiguous against this instruction, stop and ask the user rather than guessing. The user is available throughout the session for clarification.

At the end of your response, after the file, report:
- The Revision number assigned to the latest entry.
- The Template Revision number set in the title block (should be `2`).
- Any decisions you made that the user should confirm.
- All detect-and-report findings, per the obligations in §B below.

---

## A. Transformations to apply

### A.1 Title block changes

The title block sits at the top of the document, immediately under the H1 working title.

**Remove these fields entirely if present:**
- `**Author:**` (entire line)
- `**Last Updated:**` (entire line)

**Add these fields:**
- `**Revision:** NN` — where `NN` is the integer matching the highest-numbered entry in the retrofitted revision log (computed after applying §A.3 below).
- `**Template Revision:** 2` — fixed value, identifying that this spec now adheres to Template Revision 2.

**Field order after migration:** Project, Short description, Status, Revision, Template Revision, Created, Related Documents. Insert the new fields in their correct positions.

Preserve all other title block fields exactly as they are, including any project-specific fields not enumerated in this instruction.

### A.2 Revision log order

The revision log is §14 of the spec. Entries are currently listed in reverse-chronological order (most recent first).

Reverse this so entries appear in chronological order — oldest first, most recent last (append-order). This is purely a reordering of existing entries; no entry is added or removed.

### A.3 Revision log entry format

Each entry's heading is currently an ISO-8601 timestamp, like:

```
### 2026-05-06T14:30:00Z

[Narrative description of the change.]
```

Convert each entry to the new format:

```
### Revision NN

**Date:** 2026-05-06T14:30:00Z

[Narrative description of the change, preserved verbatim.]
```

Where:
- `NN` is an integer revision number. Starting at the oldest entry, assign `Revision 1`, then increment by one for each subsequent entry in chronological order.
- The `**Date:**` field carries the original timestamp from what was previously the heading.
- The narrative body of each entry is preserved exactly as written. Do not reword, restructure, or improve it.

### A.4 Honor pre-existing revision numbers

The user may have informally added explicit revision numbers to some entries (for example, an entry already labeled "Revision 2"). If you encounter pre-existing revision numbers:

- Treat them as authoritative.
- Number the surrounding entries to be consistent with them — e.g., if the second entry is already labeled "Revision 2," then the first entry must be "Revision 1" and the third must be "Revision 3."
- If pre-existing numbers conflict with chronological order (an older entry has a higher number than a newer one), **stop and ask the user** for guidance rather than guessing.

### A.5 Handle entries without timestamps

Some specs may have entries with prose headings like `### Initial Draft` rather than ISO-8601 timestamps. For these:

- Convert the heading to `### Revision NN` per the standard rule.
- If the entry contains no embedded date anywhere, **stop and ask the user** what to use for the `**Date:**` field rather than inventing a value.

### A.6 Replace §1.9 Spec Conventions with a reference statement

Under Template Revision 1, each spec contained a full conventions block embedded in §1.9 — covering item identifiers, number stability, requirement language, table usage, and so on. Under Template Revision 2, conventions live in the template itself; specs reference the template revision they adhere to.

**Replace the entire content of §1.9 (everything under the `### 1.9 Spec Conventions` heading, up to but not including the next heading) with this reference statement:**

```
### 1.9 Spec Conventions

This spec adheres to the conventions defined in `SPEC_TEMPLATE.md` at the revision identified by the `Template Revision` field in this spec's title block. See the Conventions section of that document for identifier rules, number stability, requirement language, table usage, revision consistency, intra-log cross-references, and other conventions governing this spec.

The conventions are not duplicated here. The template is the single source of truth for the rules a spec follows; this spec references the template revision it was authored under, and migrations to newer template revisions are deliberate (see `SPEC_RETROFIT_INSTRUCTION.md`).
```

Do not preserve any part of the existing §1.9 content. The reference model deliberately externalizes the conventions; preserving fragments of the old block would undermine the single-source-of-truth property.

---

## B. Detect-and-report obligations

The transformations in §A change the structure of the spec. Some content elsewhere in the spec may reference the changed structural elements and become stale, but is outside the editing scope of this instruction. For each of the checks below, scan the migrated spec, list any findings, and report them to the user as part of your final response. **Do not edit these references directly** — the user resolves them as a follow-up.

### B.1 Positional language in revision log entry bodies

After reversing §14's order (per §A.2), scan the body of each revision log entry for positional language that may have been valid in the old order but is wrong in the new order:

- `above`, `below`, `previous`, `next`, `earlier`, `later`, `preceding`, `following`
- Phrases like "the entry above," "described previously," "as noted later in the log"

For each match, report:
- The revision number of the entry containing the language.
- The exact phrase.
- A note that the user should rewrite the reference using the stable `Revision NN` identifier per Template Revision 2 conventions.

### B.2 Bare timestamp cross-references between revision log entries

Some entry bodies may reference other entries by quoting their timestamp directly, e.g., "the 2026-05-11T06:15:07Z entry." Under Template Revision 2, such references should use the stable `Revision NN` identifier instead.

Scan each entry's body for ISO-8601-shaped timestamps that appear to reference other entries (not the entry's own `**Date:**` field). Report each finding so the user can rewrite it as a stable revision-number reference.

### B.3 References to removed title block fields

The migration removes the `Author` and `Last Updated` fields from the title block. The spec may contain prose elsewhere — in the body, in §1.9 (now replaced), in section narratives — that references those fields.

Scan the entire spec for the literal strings `Author` and `Last Updated` (case-insensitive) appearing outside the title block. Report each finding so the user can decide whether to rewrite or remove the reference.

Note: the literal §1.9 conventions block being replaced in §A.6 contained the old "Timestamp consistency" rule referencing `Last Updated`. That instance is handled by the replacement and does not need to be reported.

### B.4 Other downstream references

If, while applying this migration, you notice other prose in the spec that appears to reference the old structural conventions (revision log order, identifier formats, embedded conventions) and might be invalidated by the changes, report those findings as well. Use judgment: only flag prose that genuinely appears stale, not every mention of a related concept.

---

## C. What NOT to change

- Do not rewrite, reword, or restructure the narrative body of any revision log entry beyond the heading-and-Date transformation in §A.3.
- Do not modify any section of the spec outside the title block, §1.9 (replaced per §A.6), and §14 (transformed per §A.2 through §A.5).
- Do not edit content flagged by the detect-and-report obligations in §B. Flag it; the user resolves it.
- Do not "improve" content, fix typos, or normalize formatting outside the scope of this migration.
- Do not add or remove revision log entries — only reformat the existing ones.
- Do not change cross-references in the body of the spec to sections, items (FR, KD, OI, etc.), or other identifiers.

If the spec contains content that seems wrong, incomplete, or worth improving, do not act on that observation as part of this migration. Note it to the user at the end of your response if it seems significant, but leave the content unchanged.

---

## D. When to stop and ask the user

Surface friction rather than guessing whenever:

- Pre-existing revision numbers in the spec conflict with chronological order.
- An entry lacks a timestamp and there is no obvious replacement value.
- The title block contains fields not addressed by this instruction.
- The spec appears partially migrated already (mixed Revision 1 / Revision 2 conventions in different places).
- §1.9 does not exist or has been restructured beyond recognition.
- The spec is missing §14, or §14 has no entries.
- Any aspect of the spec's structure is ambiguous against this instruction.

---

## E. Output

Produce the complete migrated spec as a single file. After the file, briefly state:

1. The Revision number assigned to the latest entry (and matching the title block's `Revision` field).
2. Confirmation that the title block's `Template Revision` field is set to `2`.
3. Any decisions you made that the user should confirm.
4. The complete list of detect-and-report findings from §B, organized by check (B.1, B.2, B.3, B.4). If a check returned no findings, state that explicitly.
5. Any observations about the spec that fell outside the migration scope but seem worth flagging.

Do not produce a diff or a summary in place of the file. The user needs the complete updated document to save.

---

*End of migration instruction.*
