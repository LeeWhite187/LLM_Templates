# User management — permanently removing a disabled, never-used account (cross-project note)

**Date:** 2026-10-28. **Reference implementation:** Facility Dashboard Platform (FDP), commit `6bf3770`
(`feat(users): reference-guarded "Remove" for disabled, never-used accounts`).
**Audience:** any project that inherited FDP's user-management model — Windows/AD identities stored as
`UserAccount` rows with **soft-delete (disable)** rather than hard-delete, group memberships, a
diagnostics/audit log, and the `ReferenceGuard` pattern for reference-guarded deletion. Portable:
adapt entity, table, and file names to the consuming project.

---

## Summary

The user list only had **Disable** (a soft delete). That is correct for any account that has *done*
something — its authored work and audit entries must stay attributable, so the row is kept (disabled)
forever. But it left no clean path for an account **created in error** — a mistyped identity, a
duplicate, an entry added and then abandoned — that never did anything. The only way to scrub such an
entry was a **manual database script**, which bypasses the audit trail entirely.

This feature adds **Remove**: a permanent delete that is allowed **only** for a disabled account with
**zero system activity**. It gives admins the "scrub a mistake" path without a hand-run SQL delete,
and it is itself audited. An account that ever acted is refused removal (with the list of what it did)
and stays disabled.

The whole feature rests on one distinction: **did this identity ever *act* in the system?** No →
removable. Yes → kept. Getting that question right — specifically, *not* counting the audit lines that
merely name the account as the subject of its own creation/disable — is the entire subtlety.

---

## How to tell if your project should adopt this

You want this if all of these are true:

1. User accounts are **soft-deleted** (a `DisabledAtUtc`/`DisabledBy` pair, or equivalent), never
   hard-deleted, so authored work and audit history stay attributable.
2. Admins occasionally create an account **in error** (a typo in the identity, a duplicate, an entry
   abandoned before first use) and currently have no way to remove it short of a DB script.
3. You have (or can add) a way to enumerate everywhere an identity can appear as an **actor**
   (CreatedBy/PublishedBy/RegisteredBy/…), and an audit/diagnostics log to record the removal.

If admins are reaching for manual `DELETE FROM UserAccounts …` to clean up mistaken entries, you have
the gap this closes.

---

## The design (conceptual model)

Three states for an account, two transitions the admin can drive:

| State | How reached | Disposition |
|---|---|---|
| **Active** | created + used | normal |
| **Disabled** | admin **Disable** (soft delete) | row + memberships kept (grayed), authorizes as no group |
| **Removed** | admin **Remove**, only from Disabled **and only if never acted** | row + memberships hard-deleted; a diagnostic records it |

Two rules make this safe:

- **Two-step by design.** An account must be **disabled first**, then removed. Removal is always a
  deliberate second act on an already-inert account — never a one-click destroy from the active list.
- **The activity gate.** Removal is permitted only when the identity appears in **no actor column
  anywhere**. Any hit → refused, and the account stays disabled so its history remains attributable.

Why keep accounts-with-history forever? Because their identity string is the author/actor stamped on
dashboards, versions, drafts, resolved problems, settings changes, and audit lines. Delete the row and
those references dangle — you lose the ability to say who did what. Soft-delete is the right permanent
disposition for them; **Remove is only for accounts that have nothing to dangle.**

---

## THE critical subtlety — scan ACTOR columns only, never audit message text

This is the one thing to get right, and the easiest to get wrong.

An audit/diagnostics log contains lines like *"DOMAIN\jsmith disabled (soft-deleted)."* or
*"DOMAIN\jsmith added to Admin group."* Those lines **name** the account — but they are records of an
action performed **by an admin** *about* the account, not actions **by** the account. The acting admin
is in the log's **actor** column (`User`/`By`), and the subject identity is only inside the free-text
message.

If your activity check searches the audit **message text** for the identity, then **every** disabled
account looks "active" (it always has at least its own "disabled" line), and **nothing is ever
removable** — the feature silently does nothing. Conversely, if you only scan actor columns, a
mistaken account that was created, maybe group-assigned, then disabled — all acts *by an admin* —
correctly shows **zero activity** and is removable.

**Rule: enumerate only the columns where this identity is the one who acted. Never grep audit/log
message bodies.** In FDP the guard documents this explicitly and scans only actor columns; see below.

---

## Implementation

### 1. Prerequisite — soft-delete on the account entity

```csharp
public class UserAccount
{
    public string WindowsIdentity { get; set; } = string.Empty; // unique AD/host identity (the actor stamp)
    public string DisplayName { get; set; } = string.Empty;
    public DateTime CreatedAtUtc { get; set; }
    public DateTime? DisabledAtUtc { get; set; }   // soft-delete marker
    public string? DisabledBy { get; set; }
    public List<GroupMembership> Memberships { get; set; } = new();
}
```

`DisabledAtUtc == null` means active; non-null means disabled. Remove is gated on non-null.

### 2. The activity guard (actor columns only)

A companion to the versioned-entity `ReferenceGuard`, but for an identity. One method answers "did this
identity ever act?" and returns **human-readable** descriptions so the UI can explain a refusal.

```csharp
/// Scans ONLY actor columns (CreatedBy / PublishedBy / RegisteredBy / LockedBy / GrantedBy / ...).
/// NEVER scans free-text audit messages that merely name the account as the subject of its own
/// creation/disable — those carry the ACTING admin in their actor column, not this identity, and
/// counting them would make every disabled account look active and defeat the feature.
public async Task<List<string>> GetActivityReferencesAsync(string identity, CancellationToken ct = default)
{
    var refs = new List<string>();
    void Add(int n, string one, string many) { if (n > 0) refs.Add(n + " " + (n == 1 ? one : many)); }

    // Authored entities + their published versions (entity CreatedBy + version PublishedBy/UploadedBy)
    Add(await _db.Dashboards.CountAsync(x => x.CreatedBy == identity, ct)
        + await _db.DashboardVersions.CountAsync(x => x.PublishedBy == identity, ct),
        "dashboard authored/published", "dashboards authored/published");
    // … widget types, playlists, media assets, content sources, collection tasks …

    // Operational acts
    Add(await _db.Stations.CountAsync(x => x.RegisteredBy == identity, ct), "station registered", "stations registered");
    Add(await _db.BroadcastChannels.CountAsync(x => x.CreatedBy == identity || x.ActivatedBy == identity, ct),
        "broadcast channel created/activated", "broadcast channels created/activated");

    // Draft activity: authored, currently holding the lock, or took over
    Add(await _db.Drafts.CountAsync(x => x.CreatedBy == identity || x.LockedBy == identity || x.TakenOverBy == identity, ct),
        "draft authored/held", "drafts authored/held");
    Add(await _db.DraftChangeEvents.CountAsync(x => x.By == identity, ct), "draft edit recorded", "draft edits recorded");

    // Administrative acts performed BY this identity (not acts ABOUT it)
    Add(await _db.GroupMemberships.CountAsync(x => x.GrantedBy == identity, ct),
        "group grant made to another account", "group grants made to other accounts");
    Add(await _db.Problems.CountAsync(x => x.ResolvedBy == identity, ct), "problem resolved", "problems resolved");
    Add(await _db.SystemSettings.CountAsync(x => x.UpdatedBy == identity, ct), "system setting changed", "system settings changed");
    Add(await _db.Diagnostics.CountAsync(x => x.User == identity, ct), "diagnostic action logged", "diagnostic actions logged");

    return refs; // empty list == never acted == removable
}
```

**Porting note:** the one authoritative list of what to scan is "every actor column in your schema."
Grep your model for `…By` / `…By == ` / `RegisteredBy` / `User ==` and make sure each appears here.
Missing one is a *safety* bug (you might remove an account that did act and dangle its references), so
err toward over-inclusion. Identity comparison follows the DB collation (case-insensitive on a default
SQL Server install), matching how the identity was stamped — don't force your own case rules.

### 3. The endpoint (Admin-only, layered checks)

```csharp
[HttpPost("users/remove")]
[Authorize(Policy = Policies.Admin)]          // Admin only — this is destructive
public async Task<IActionResult> RemoveUser([FromBody] RemoveUserRequest request, CancellationToken ct)
{
    if (string.IsNullOrWhiteSpace(request.WindowsIdentity))
        return BadRequest(new ApiError { Message = "A Windows identity is required." });

    var identity = request.WindowsIdentity.Trim();

    // Defensive: you can't reach this for yourself (removal needs a disabled account and you can't
    // disable yourself), but refuse explicitly rather than rely on that chain.
    if (string.Equals(identity, UserName, StringComparison.OrdinalIgnoreCase))
        return BadRequest(new ApiError { Message = "You cannot remove your own account." });

    await using var db = await _dbFactory.CreateDbContextAsync(ct);
    var user = await db.UserAccounts.Include(u => u.Memberships)
                       .FirstOrDefaultAsync(u => u.WindowsIdentity == identity, ct);
    if (user == null)
        return NotFound(new ApiError { Message = "No such user account." });

    // Must be disabled first — removal is always a deliberate second act on an inert account.
    if (user.DisabledAtUtc == null)
        return BadRequest(new ApiError { Message = "Only a disabled account can be removed. Disable it first, then remove it." });

    // The real gate: any activity → kept for audit.
    var references = await new UserReferenceGuard(db).GetActivityReferencesAsync(identity, ct);
    if (references.Count > 0)
        return Conflict(new ApiError {
            Message = "This account has interacted with the system, so it cannot be removed; it stays " +
                      "disabled so its history remains attributable. Only a never-used account (created " +
                      "in error) can be removed.",
            References = references });

    db.GroupMemberships.RemoveRange(user.Memberships);
    db.UserAccounts.Remove(user);
    await db.SaveChangesAsync(ct);

    _users.Invalidate();                        // drop any cached identity/authorization snapshot
    await _diagnostics.LogAsync(DiagnosticLevel.Information, "groups",
        identity + " permanently removed - disabled account with no system activity (created in error).", UserName);
    _changes.Notify("users");                   // live-refresh any open admin UI
    return NoContent();
}
```

Order matters: **validate → refuse self → exists → is disabled → activity gate → delete**. The
activity gate is last and authoritative. **Audit the removal itself** (the diagnostic line above) —
that is what keeps this from being an audit-trail bypass.

### 4. The error contract (so the UI can explain a refusal)

```csharp
public class ApiError
{
    public string Message { get; set; } = string.Empty;
    public string? Detail { get; set; }
    public List<string> References { get; set; } = new();   // the activity list on a 409
}

public class RemoveUserRequest { public string WindowsIdentity { get; set; } = string.Empty; }
```

A refusal is **409 Conflict** with `References` populated (same shape your versioned-entity reference
guard already returns). Not-disabled / missing-identity are **400**; unknown account is **404**.

### 5. The UI (button, guarded visibility, confirm, refusal surfacing)

- **Button** shows only for a **disabled** account that is **not yourself**:
  `*ngIf="u.disabled && !isSelf(u)"`. Style it as destructive.
- **Tooltip** states the rule in plain words, e.g.:
  > "Permanently delete this disabled account. Allowed only for an account that never interacted with
  > the system — one that authored or published nothing and holds no audit history (an entry created by
  > mistake or abandoned on creation). If the account has any activity it is kept (removal is refused)
  > so its history stays attributable. The removal is itself recorded in the diagnostics log."
- **Confirm** before calling, since it is irreversible.
- **Surface the 409.** Reuse the toast/error handler that already renders `{ message, references }`
  from your reference-guard failures — the admin sees *why* and *what the account did*. On success,
  toast and reload the list.

```ts
remove(user: UserDto): void {
  if (!confirm(`Permanently remove ${user.windowsIdentity}?\n\n… (only a disabled, never-used account) …`)) return;
  this.api.removeUser(user.windowsIdentity).subscribe({
    next: () => { this.toasts.success(`${user.windowsIdentity} permanently removed.`); this.load(); },
    error: err => { this.toasts.error(err, 'The account could not be removed.'); this.load(); } // renders references
  });
}
```

---

## How to verify

1. **Mistaken account path:** create an account (or let one be auto-registered), do nothing with it,
   **Disable** it, then **Remove** it → it disappears; a diagnostics line records the removal.
2. **Protected path:** take an account that authored/published something (or registered a station,
   resolved a problem, etc.), disable it, try **Remove** → **409**, and the toast lists what it did; the
   account stays in the list, disabled.
3. **Two-step guard:** confirm **Remove** is not offered for an *active* account, and the endpoint
   returns 400 if called on one.
4. **Audit-text trap (the important one):** confirm a disabled account whose *only* log footprint is
   its own "created/disabled/added-to-group" lines is **removable** — proving the guard scans actor
   columns, not message text. If it is refused, your guard is grepping audit messages; fix it.
5. **Self-removal:** confirm you cannot remove your own account.

---

## Portability checklist for an inheriting project

- [ ] Accounts are soft-deleted (disable), with a `DisabledAtUtc`/`DisabledBy` (or equivalent) marker.
- [ ] Add an identity activity guard that scans **every actor column** in your schema — and **no audit
      message text**. Grep the model for `…By`/`User ==` columns; over-include rather than miss one.
- [ ] Guard returns **human-readable** descriptions, reused in the refusal response.
- [ ] Add an **Admin-only** remove endpoint: validate → refuse self → exists → **is disabled** →
      **activity gate** → delete row + memberships.
- [ ] **Audit the removal** in the diagnostics/audit log (otherwise you have just rebuilt the DB-script
      bypass this feature exists to avoid).
- [ ] Invalidate any cached identity/authorization snapshot and notify open admin UIs.
- [ ] Return **409 + references** on refusal; **400** not-disabled/bad-input; **404** unknown.
- [ ] UI: destructive button shown only for disabled, non-self accounts; plain-language tooltip;
      confirm dialog; surface the 409 reference list on refusal.
- [ ] Verify all five paths above — especially the audit-text trap (#4).
