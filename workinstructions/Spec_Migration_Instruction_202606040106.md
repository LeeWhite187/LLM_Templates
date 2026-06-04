# Spec Document Retrofit Work Instruction

## Purpose

Update an existing project specification document from an earlier template convention to the current template convention. This is a mechanical retrofit. Do not change the substance of the spec — only its structural format in the title block and revision log.

## Workflow

The user will paste this instruction along with the spec document to be retrofitted. Apply the transformations described below to the spec and produce the complete updated document as your output. Do not summarize the changes; produce the full retrofitted document so the user can save it and replace the original.

If anything in the spec is ambiguous against this instruction, stop and ask the user rather than guessing. The user is available throughout the session for clarification.

---

## Transformations to apply

### 1. Title block changes

The title block sits at the top of the document, immediately under the H1 working title.

**Remove these fields entirely if present:**
- `**Author:**` (entire line)
- `**Last Updated:**` (entire line)

**Add this field:**
- `**Revision:** NN` — where `NN` is the integer matching the highest-numbered entry in the retrofitted revision log (computed after applying §3 below).

**Field order after retrofit:** Project, Short description, Status, Revision, Created, Related Documents. Insert the new Revision field in the correct position.

Preserve all other title block fields exactly as they are, including any project-specific fields not enumerated in this instruction.

### 2. Revision log order

The revision log is §14 of the spec. Entries are currently listed in reverse-chronological order (most recent first).

Reverse this so entries appear in chronological order — oldest first, most recent last (append-order). This is purely a reordering of existing entries; no entry is added or removed.

### 3. Revision log entry format

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

### 4. Honor pre-existing revision numbers

The user may have informally added explicit revision numbers to some entries in some specs (for example, an entry already labeled "Revision 2"). If you encounter pre-existing revision numbers:

- Treat them as authoritative.
- Number the surrounding entries to be consistent with them — e.g., if the second entry is already labeled "Revision 2," then the first entry must be "Revision 1" and the third must be "Revision 3."
- If pre-existing numbers conflict with chronological order (an older entry has a higher number than a newer one), **stop and ask the user** for guidance rather than guessing.

### 5. Handle entries without timestamps

Some specs may have entries with prose headings like `### Initial Draft` rather than ISO-8601 timestamps. For these:

- Convert the heading to `### Revision NN` per the standard rule.
- If the entry contains no embedded date anywhere, **stop and ask the user** what to use for the `**Date:**` field rather than inventing a value.

---

## What NOT to change

- Do not rewrite, reword, or restructure the narrative body of any revision log entry.
- Do not modify any section of the spec outside the title block and §14.
- Do not "improve" content, fix typos, or normalize formatting outside the scope of this retrofit.
- Do not add or remove revision log entries — only reformat the existing ones.
- Do not change cross-references elsewhere in the spec, even if they reference dates or other identifiers.

If the spec contains content that seems wrong, incomplete, or worth improving, do not act on that observation as part of this retrofit. Note it to the user at the end of your response if it seems significant, but leave the content unchanged.

---

## When to stop and ask the user

Surface friction rather than guessing whenever:

- Pre-existing revision numbers in the spec conflict with chronological order.
- An entry lacks a timestamp and there is no obvious replacement value.
- The title block contains fields not addressed by this instruction.
- The spec appears partially retrofitted already (mixed old/new conventions in different places).
- Any aspect of the spec's structure is ambiguous against this instruction.
- The spec doesn't match the general shape expected (e.g., no §14 Revision Log at all).

---

## Output

Produce the complete retrofitted spec as a single file. After the file, briefly state:
- The Revision number assigned to the latest entry (and therefore to the title block).
- Any decisions you made that the user should confirm.
- Any observations about the spec that fell outside the retrofit scope but seem worth flagging.

Do not produce a diff or a summary in place of the file. The user needs the complete updated document to save.

---

*End of work instruction.*
