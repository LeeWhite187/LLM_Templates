# User Display Names: store the identity, render the person

*Transfer note written from Prospector (bmx.Historical_Queries), where this shipped in builds
1.1.53–1.1.54 (commits `1a0d672`, `574e259`). Written to be carried to sibling projects — the
code excerpts are current Prospector source, trimmed only where marked.*

## The problem

Apps that authenticate with Windows/Negotiate end up showing raw identities everywhere a person
appears: `PLANT\074211` in the top-right corner, in audit logs, in CreatedBy / ChangedBy /
RequestedBy columns. On domains with numeric or coded usernames, nobody can tell who did what
without a lookup in their head.

The obvious fix — store the friendly name in the records — is the wrong one. Names drift
(marriages, rebrands, directory cleanups), records are forever, and a name baked into a row at
write time degrades the audit trail the moment it goes stale.

## The design in one sentence

**Rows and logs store the durable identity; the friendly name lives in exactly one place (the
user-account row), is resolved from the directory exactly once (at account creation), and every
surface that *shows* a person translates identity → name at render time from a shared map.**

Consequences that make this worth copying:

- **Old records improve retroactively.** The moment a name resolves (or an admin fixes one),
  every historical row that references that identity starts reading as a person — no backfill of
  the record tables, ever.
- **The audit record never degrades.** What's stored is what authenticated. A rename is one
  UPDATE on one row, not a sweep through history.
- **No joins in the hot paths.** Record queries never touch the user table; the UI holds one
  small `{identity → name}` map and does the translation client-side.
- **Deleting or renaming an account can't orphan anything.** Unresolvable identities simply
  render as themselves.

Two deliberate scope cuts, both worth keeping:

- **The directory lookup is one-time**, at account creation (plus a one-time backfill for
  pre-existing accounts). Name drift *after* creation is out of scope; a name can be corrected
  by hand (a direct row edit — Prospector ships no UI for it, and the automation never
  overwrites a hand-set value). No periodic AD sync service, no drift detection.
- **A lookup failure is silent and harmless.** Off-domain dev box, unreachable DC, unknown
  account — the identity simply stands as the display name. The feature degrades to exactly the
  pre-feature behavior, never to an error.

## Pieces

### 1. Storage: one nullable column

On the user-account table (`tbl_UserAccount` here):

```sql
,[DisplayName] NVARCHAR(256) NULL
```

The account-ensure path seeds it with the identity itself, so the column is never
meaningfully empty:

```sql
IF NOT EXISTS (SELECT 1 FROM [dbo].[tbl_UserAccount] WHERE [WindowsIdentity] = @Identity)
    INSERT INTO [dbo].[tbl_UserAccount] ([WindowsIdentity], [DisplayName]) VALUES (@Identity, @Identity);
```

That seed is also the sentinel: *display name equals identity* means "nothing better is known,"
which both the backfill and the names endpoint key off (below). A hand-set name, by definition,
differs from the identity and is therefore never overwritten by automation.

### 2. The directory lookup (`IIdentityNameLookup`)

A tiny abstraction so the rest of the app never touches directory APIs:

```csharp
/// Resolves a Windows identity (DOMAIN\user) to its directory display name — Active
/// Directory on a domain, the local SAM otherwise. A ONE-TIME lookup when an admin first
/// adds the user; name drift over time is explicitly out of scope. Null when the directory
/// is unreachable, the account is unknown, or no friendlier name exists — the identity then
/// stands as the display name.
public interface IIdentityNameLookup
{
    Task<string?> GetDisplayNameAsync(string identity, CancellationToken ct = default);
}
```

The Windows implementation, complete — the details here were all earned:

```csharp
using System.DirectoryServices.AccountManagement;   // NuGet: System.DirectoryServices.AccountManagement

[SupportedOSPlatform("windows")]
public sealed class WindowsIdentityNameLookup : IIdentityNameLookup
{
    public Task<string?> GetDisplayNameAsync(string identity, CancellationToken ct = default) =>
        Task.Run(() => Lookup(identity), ct); // the AD API is synchronous/blocking

    private static string? Lookup(string identity)
    {
        var parts = (identity ?? "").Split('\\', 2);
        var (domain, sam) = parts.Length == 2 ? (parts[0], parts[1]) : (null, parts[0]);
        if (string.IsNullOrWhiteSpace(sam)) return null;

        var isLocal = string.IsNullOrWhiteSpace(domain) || domain == "." ||
                      string.Equals(domain, Environment.MachineName, StringComparison.OrdinalIgnoreCase);
        try
        {
            using var ctx = isLocal
                ? new PrincipalContext(ContextType.Machine)
                : new PrincipalContext(ContextType.Domain, domain);
            using var principal = UserPrincipal.FindByIdentity(ctx, IdentityType.SamAccountName, sam);
            if (principal is null) return null;

            var display = principal.DisplayName;
            if (string.IsNullOrWhiteSpace(display))
                display = ($"{principal.GivenName} {principal.Surname}").Trim();

            display = display?.Trim();
            // A "friendly" name identical to the account name is no improvement — keep null so
            // callers know nothing better exists.
            return string.IsNullOrWhiteSpace(display)
                || string.Equals(display, sam, StringComparison.OrdinalIgnoreCase)
                ? null : display;
        }
        catch
        {
            return null;
        }
    }
}
```

Points that matter when porting:

- **Local-machine fallback.** When the "domain" is the machine itself (or `.`, or absent), use
  `ContextType.Machine` — local test accounts resolve from the SAM instead of failing an AD bind.
- **`DisplayName`, then `GivenName + Surname`.** Some directories leave `displayName` blank but
  fill the name parts.
- **The no-improvement rule.** A directory whose display name just echoes the sam account name
  must return null, so the caller knows nothing friendlier exists (and the backfill can retry
  another day rather than stamping the sentinel as if it were a real name).
- **Catch-everything.** Any directory exception → null. The feature must work (by doing nothing)
  on an off-domain dev machine.
- **`Task.Run` wrapper.** `System.DirectoryServices.AccountManagement` is blocking; don't let it
  stall a request thread pool synchronously.

### 3. Resolve at the one moment an account is born

Accounts here are created only by an admin granting a group (no auto-registration on
authentication). The grant handler checks whether the account existed *before* the grant, and
only a newly created one gets the lookup:

```csharp
/// When a grant CREATES the account, resolve the directory's friendly name (AD on a domain,
/// local SAM otherwise) into DisplayName — once. A failed or blank lookup leaves the seeded
/// fallback: the identity itself.
private async Task FillDisplayNameIfNewAsync(bool existedBefore, string identity, CancellationToken ct)
{
    if (existedBefore) return;
    var friendly = await _names.GetDisplayNameAsync(identity, ct);
    if (friendly is not null)
        await _users.SetDisplayNameAsync(identity, friendly, ct);
}
```

Called from every path that can create an account (here: the single-grant endpoint and the
set-all-groups endpoint). If your sibling project auto-creates accounts on first authentication
instead, hook the same call there — the principle is *once, at birth*.

### 4. One-time backfill for accounts that predate the feature

A hosted `BackgroundService` that runs once per database, ever:

- Waits ~10 s after startup (let the service settle; the directory may not be reachable in the
  first seconds after boot).
- Checks a marker row in the config table (`Users_DisplayNameBackfillDone = '1'`); exits if set.
- Lists all accounts; candidates are those whose `DisplayName` is null **or equals the
  identity** — a hand-set name is never touched.
- Looks each candidate up exactly as the add-time path would; writes only non-null results.
- Writes the marker **after** the pass, so a shutdown mid-pass retries on the next start.
- Logs one summary diagnostic line ("resolved N of M candidate account(s)…", source `system`)
  and notifies the UI's live-change channel so open Users pages refresh.
- Any exception: log a warning, *don't* write the marker, retry next start.

(Full source: `src/Prospector.Api/Ops/DisplayNameBackfillService.cs` — it's ~100 lines and all
of it follows from the bullets above.)

### 5. The names endpoint: a map, not a user list

```csharp
/// The identity → display-name map: every surface that SHOWS a person resolves the friendly
/// name at RENDER time from this map, while rows and logs keep STORING the durable identity.
/// Readable by any authenticated user (coworkers' names are not a secret in-plant) — unlike
/// the Admin-only Users page this carries names and nothing else.
[ApiController]
[Route("api/users/names")]
[Authorize(Policy = Policies.AnyAccess)]
public sealed class UserNamesController : ControllerBase
{
    private readonly IUserDirectory _users;
    public UserNamesController(IUserDirectory users) => _users = users;

    [HttpGet]
    public async Task<IActionResult> Get(CancellationToken ct) =>
        Ok((await _users.ListUsersAsync(ct))
            .Where(u => !string.IsNullOrWhiteSpace(u.DisplayName)
                     && !string.Equals(u.DisplayName, u.Identity, StringComparison.OrdinalIgnoreCase))
            .Select(u => new { identity = u.Identity, displayName = u.DisplayName }));
}
```

Two deliberate authorization decisions, worth repeating in a sibling:

- **Any authenticated user may read it.** Non-admin pages (run history, diagnostics the user can
  see, etc.) also show people; gating the map to admins would leave everyone else staring at raw
  identities. The payload is names and nothing else — no groups, no disabled flags.
- **It only returns identities whose name actually differs.** Identity-as-its-own-name rows are
  noise; the fallback in the client handles them for free, and the payload stays small.

Separately, the current user's own name rides on the existing "who am I" endpoint, so the shell's
identity chip needs no second call:

```csharp
return Ok(new
{
    identity,
    displayName = string.IsNullOrWhiteSpace(account?.DisplayName) ? identity : account!.DisplayName,
    groups = ...,
});
```

### 6. Client-side: one service, one rule

A root-provided Angular service holds the map and does all translation (adapt freely — the shape
matters, not the framework):

```typescript
/**
 * Render-time identity → display-name resolution: rows and logs STORE the durable identity;
 * every surface that SHOWS a person asks this service for the friendly name (falling back to
 * the identity itself). Old records thereby improve retroactively the moment a name resolves,
 * and the audit record underneath never degrades — templates pair name(x) with [title]="x"
 * so the raw identity is always one hover away.
 */
@Injectable({ providedIn: 'root' })
export class PeopleService {
  private map = signal<Map<string, string>>(new Map());
  private loaded = false;

  /** The display name for an identity, or the identity itself when none is known. */
  name(identity: string | null | undefined): string {
    this.ensureLoaded();
    if (!identity) return '—';
    return this.map().get(identity.toLowerCase()) ?? identity;
  }

  private ensureLoaded(): void {
    if (this.loaded) return;
    this.loaded = true;
    this.load();
    // Admin edits (and the one-time backfill) land live.
    this.changes.of('users').pipe(takeUntilDestroyed(this.destroyRef)).subscribe(() => this.load());
  }

  private load(): void {
    this.api.userNames().subscribe({
      next: rows => this.map.set(new Map(rows.map(r => [r.identity.toLowerCase(), r.displayName]))),
      error: () => {}, // identities simply stand until the map loads
    });
  }
}
```

Usage is one idiom, applied everywhere a person renders:

```html
<td [title]="h.requestedBy">{{ people.name(h.requestedBy) }}</td>
```

**The pairing rule is the heart of the client side: every `people.name(x)` carries
`[title]="x"`** — the friendly name is what you read, the durable identity is one hover away.
That keeps the audit trail inspectable without cluttering the screen, and it means a display-name
collision (two John Smiths) is always disambiguable in place.

Details that matter:

- **Case-insensitive keys.** The map is keyed on `identity.toLowerCase()` because SQL Server's
  default collation is case-insensitive — the stored actor stamps and the map rows may disagree
  on casing.
- **Lazy, once, then live.** The map loads on first use and re-loads on the `users` live-change
  signal, so an admin's rename (or the one-time backfill finishing) updates every open page
  without a refresh. If you have no live-change channel, load once per session — staleness here
  is cosmetic.
- **Fail to the identity.** A failed map load is swallowed; everything renders as identities,
  which is exactly the pre-feature behavior.
- **The dash for nothing.** `name(null)` renders `—`, so optional actor columns don't show blank.

### 7. Where it shows (Prospector's inventory — a porting checklist shape)

Every surface below stores/receives the identity and renders through `people.name(...)` +
`[title]` (the shell chip uses the `/me` payload instead):

| Surface | Fields resolved |
|---|---|
| Shell top bar (identity chip) | current user (`/me`'s `displayName`, identity fallback) |
| Users & Groups admin page | per-row Display name column (shows the stored value directly) |
| Query authoring lists/cards | draft `createdBy`, `lockedBy` ("editing: …"), version `publishedBy`, "base of a draft by …" |
| Query editor dialog | draft holder banner, take-over confirm, taken-over-while-away notice, undo-checkpoint tooltips, "other drafts open by …" |
| Run history (two pages) | `requestedBy` |
| Telemetry | `actor`, pinned-by tooltip |
| Diagnostics / audit log | `source` (the actor column) |
| Script-safety records | `editor`, reviewed-by tooltip |
| UI sessions (presence) | session `user` |

When porting, build the equivalent inventory first: grep the UI for every template that prints an
identity-shaped field, then convert each to the `name(x)` + `title` idiom. Missing one isn't a
bug, just an un-upgraded corner — the idiom can be adopted surface by surface.

## What *not* to do (traps we considered or hit)

- **Don't store display names in record rows.** Everything in "The design in one sentence"
  exists to avoid this; the first paragraph of this note is why.
- **Don't write display names into audit/log message text.** Messages keep naming identities;
  the UI resolves the *actor column* for display. (This also preserves the sibling
  account-removal feature's "actor-columns-only" activity guard — message text stays
  identity-stable and greppable.)
- **Don't resolve per-render against the directory.** The directory is slow, blocking, and
  sometimes unreachable; that's why the name is captured once into the database and the UI reads
  a map from *our own* API.
- **Don't let a lookup failure surface.** Any error path must land on "the identity stands."
- **Don't overwrite a hand-set name.** The equals-identity sentinel is what makes automation
  (add-time fill, backfill) safe to re-run conceptually: it only ever upgrades the sentinel.
- **Don't gate the names map behind admin rights** unless names are genuinely sensitive in your
  deployment — you'd be re-breaking every non-admin page.

## Porting checklist

1. Add nullable `DisplayName` to the user-account table; seed it with the identity on account
   creation.
2. Drop in `IIdentityNameLookup` + `WindowsIdentityNameLookup` (NuGet
   `System.DirectoryServices.AccountManagement`); register as a singleton.
3. Call the lookup at every account-creation path (and only for newly created accounts).
4. Add the one-time backfill hosted service with a config-table marker; only touch
   sentinel-valued rows; marker written after success only.
5. Add the `{identity, displayName}` map endpoint (authenticated-users policy; differs-only
   filter) and put `displayName` on the "who am I" payload.
6. Add the client `PeopleService` equivalent (lowercased keys, identity fallback, lazy load,
   refresh on user-change signal if you have one).
7. Sweep the UI: every rendered identity becomes `name(x)` paired with a tooltip/title carrying
   the raw identity.
8. Verify: (a) a fresh grant of a real domain account shows the friendly name within one list
   refresh; (b) a grant on an off-domain dev box shows the identity, no errors; (c) pre-existing
   accounts resolve after one service restart, and the backfill logs its summary line exactly
   once across restarts; (d) a hand-edited display name survives a restart; (e) old history rows
   (runs, logs) show friendly names with the identity in the tooltip.
