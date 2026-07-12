# EF Core — Conventions & Patterns (go-by)

Field-proven EF Core 8 conventions from the Facility Dashboard Platform, written to be **lifted into
the next EF Core project**. Each is small, load-bearing, and was chosen for a concrete reason — not a
generic EF tutorial. Working examples live in `src/Dashboards.DataAccess/` and
`src/Dashboards.Host/Program.cs`.

Stack here: **EF Core 8 + SQL Server, code-first**. No Dapper; the only raw ADO.NET is DB-lifecycle
tooling that EF can't express (see §7).

---

## 1. UTC `DateTime` everywhere (the value converter)

**Problem.** SQL Server `datetime2` carries no kind, so EF materializes every `DateTime` as
`Kind=Unspecified`. A later `.ToUniversalTime()` / `.ToLocalTime()` then silently shifts it by the
host's UTC offset — a bug that only shows up on a machine whose clock isn't UTC.

**Fix.** Stamp `Kind=Utc` on every `DateTime` as it's read, applied globally via `ConfigureConventions`
so no per-property annotation is needed (covers nullable `DateTime?` too).

The converter — `UtcDateTimeValueConverter.cs`:

```csharp
using Microsoft.EntityFrameworkCore.Storage.ValueConversion;

/// <summary>
/// Stamps DateTimeKind.Utc onto every DateTime read from the database. Write side is passthrough —
/// store UTC (DateTime.UtcNow). All timestamps in this schema are UTC by convention.
/// </summary>
public class UtcDateTimeValueConverter : ValueConverter<DateTime, DateTime>
{
    public UtcDateTimeValueConverter()
        : base(v => v, v => new DateTime(v.Ticks, DateTimeKind.Utc))
    {
    }
}
```

Wire it in the `DbContext` by **overriding `ConfigureConventions`**:

```csharp
protected override void ConfigureConventions(ModelConfigurationBuilder configurationBuilder)
{
    configurationBuilder
        .Properties<DateTime>()
        .HaveConversion<UtcDateTimeValueConverter>();
}
```

- **Read side is the point** (Kind stamped); **write side is passthrough** — you must store UTC
  (`DateTime.UtcNow`), which the code does throughout.
- `.Properties<DateTime>()` at the convention level hits **all** `DateTime` properties, nullable
  included, with zero per-entity wiring.
- **Caveat:** it assumes *every* timestamp column is UTC. If a genuinely non-UTC `DateTime` is ever
  introduced, drop the blanket convention for that property and use a per-property converter instead.
- Name columns/properties `…Utc` (e.g. `CreatedAtUtc`, `LastSeenUtc`) so the convention's assumption
  is self-documenting at every call site.

### 1a. The serialization boundary — the *other* half

The EF converter fixes the value **in memory**; it does nothing about how a `DateTime` crosses the
wire. `System.Text.Json` writes a `Kind=Unspecified` value with **no offset and no `Z`**, so a browser
(`new Date(...)`, Angular's `date` pipe) reads it as **local** time — every user-presented timestamp is
off by the viewer's UTC offset. Pair the EF converter with a JSON converter that forces the `Z`, and
register it on **every** serialization boundary — MVC responses **and** the SignalR protocol:

```csharp
public sealed class UtcDateTimeConverter : JsonConverter<DateTime>
{
    public override DateTime Read(ref Utf8JsonReader r, Type t, JsonSerializerOptions o)
    {
        var v = r.GetDateTime();
        return v.Kind == DateTimeKind.Utc ? v
             : v.Kind == DateTimeKind.Local ? v.ToUniversalTime()
             : DateTime.SpecifyKind(v, DateTimeKind.Utc);
    }
    public override void Write(Utf8JsonWriter w, DateTime v, JsonSerializerOptions o)
        => w.WriteStringValue(v.Kind == DateTimeKind.Utc ? v
             : v.Kind == DateTimeKind.Local ? v.ToUniversalTime()
             : DateTime.SpecifyKind(v, DateTimeKind.Utc));   // writes ISO-8601 with 'Z'
}
```

```csharp
builder.Services.AddControllers()
    .AddJsonOptions(o => o.JsonSerializerOptions.Converters.Add(new UtcDateTimeConverter()));
builder.Services.AddSignalR()
    .AddJsonProtocol(o => o.PayloadSerializerOptions.Converters.Add(new UtcDateTimeConverter()));
```

Think of it as a matched pair: **EF converter** = "everything read from the DB is UTC-kinded";
**JSON converter** = "everything sent to a client is UTC with a `Z`". Miss the second and timestamps
render wrong in the browser even though they're correct in the database.

### 1b. The modern alternative

For a **new** schema, `DateTimeOffset` mapped to SQL `datetimeoffset` sidesteps the whole class of
problem — the offset travels with the value, so there's no kind to lose and no boundary to patch. The
converter pair above is the right fix when you're on `DateTime`/`datetime2` (existing schema, or a
deliberate "store UTC, no offset" choice).

## 2. Short-lived contexts via `IDbContextFactory`, not a scoped `DbContext`

The app registers a **factory**, not a scoped context:

```csharp
builder.Services.AddDbContextFactory<DashboardsDbContext>(o => o.UseSqlServer(connectionString));
```

and every controller/service creates a context per unit of work and disposes it:

```csharp
await using var db = await _dbFactory.CreateDbContextAsync(ct);
```

Why: much of the work runs in **`BackgroundService`s** (schedulers, monitors) and SignalR hubs, which
are singletons and can't hold a scoped `DbContext`. A factory gives each operation a fresh, correctly-
scoped, thread-safe context (a `DbContext` is not thread-safe, and long-lived contexts accumulate
tracked entities). Controllers use the same pattern for consistency. Inject
`IDbContextFactory<TContext>` anywhere, including singletons.

## 3. Schema-version gate — verify, never auto-migrate

The service **refuses to start** if the database schema is behind the binaries, rather than silently
`Database.Migrate()`-ing a production DB. A `SchemaInfo.CurrentVersion` constant in the domain is the
version the binaries require; a `SchemaVersions` table records what's applied; a startup gate compares
them:

```csharp
var check = await SchemaVersionGuard.CheckAsync(db);
if (!check.IsCompatible)
{
    Log.Fatal("SCHEMA GATE: {Error}", check.Error);   // exact reason: DB is at N, binaries need M
    await Log.CloseAndFlushAsync();
    Environment.Exit(2);
}
```

- **Migrations are forward-only / strictly monotonic** — never renumber or reorder a shipped version.
- The service never migrates; a **DBA-runnable idempotent SQL script** (generated from the migrations,
  §4) or a small `dbtool apply` command advances the schema as an explicit, backed-up step.
- Payoff: an operator who installs new binaries against an un-migrated DB gets a clear diagnostic in
  the log, not a half-migrated database or a silent auto-change.

## 4. Migration workflow (the gotchas)

The exact recipe, because two steps aren't obvious:

1. Edit the entity + its `DbContext` config.
2. Add the migration **using the DataAccess project as BOTH `--project` and `--startup-project`**:
   ```
   dotnet ef migrations add <Name> --project src/<X>.DataAccess --startup-project src/<X>.DataAccess
   ```
   The web/host project usually **lacks the `Microsoft.EntityFrameworkCore.Design` reference**, so it
   can't be the startup project; give the DataAccess project a small
   `IDesignTimeDbContextFactory<TContext>` so it can stand alone.
3. **EF does not write the version row** — hand-add it to the migration's `Up()` (and the delete to
   `Down()`), mirroring an existing migration:
   ```csharp
   migrationBuilder.InsertData("SchemaVersions",
       new[] { "Id", "AppliedAtUtc", "Description", "Version" },
       new object[] { N, new DateTime(yyyy, mm, dd, 0, 0, 0, DateTimeKind.Utc), "…", N });
   // Down(): migrationBuilder.DeleteData("SchemaVersions", "Id", N);
   ```
4. Bump `SchemaInfo.CurrentVersion` to `N`.
5. Regenerate the idempotent SQL for DBAs:
   ```
   dotnet ef migrations script --idempotent --project src/<X>.DataAccess --startup-project src/<X>.DataAccess --output deploy/sql/create-or-migrate.sql
   ```
6. Apply to any dev/preview DB explicitly (the service won't boot against a behind DB):
   ```
   dotnet ef database update --project src/<X>.DataAccess --startup-project src/<X>.DataAccess --connection "Server=(localdb)\MSSQLLocalDB;Database=<Db>;Trusted_Connection=True;"
   ```

## 5. Store app-shaped JSON as `nvarchar(max)` strings, not EF owned-JSON

Config blobs, widget bindings, view-model snapshots, etc. are stored as **plain JSON strings** and
parsed with `System.Text.Json` in the application layer — not mapped as EF owned entities / JSON
columns. Reasons: the shapes are versioned and evolve independently of the relational schema (a new
optional field needs no migration), they're passed through to other layers as-is, and the app already
owns their (de)serialization. Reach for EF's owned-types/JSON mapping only when you need to **query
into** the JSON from SQL — which this workload never does.

## 6. Soft delete + filtered unique indexes

- Mutable entities carry an `IsDeleted` (or `DecommissionedAtUtc`) flag and are filtered in queries
  (`Where(x => !x.IsDeleted)`) rather than hard-deleted — history, references, and audit survive.
- Uniqueness that should apply only to *live* rows uses a **filtered unique index**, so a re-created
  entity doesn't collide with a soft-deleted one:
  ```csharp
  e.HasIndex(x => x.Token).IsUnique().HasFilter("[DecommissionedAtUtc] IS NULL");
  ```

## 7. EF for the app; raw ADO.NET only for DB-lifecycle tooling

All application data access is EF Core LINQ through the `DbContext` — **no `FromSqlRaw` /
`ExecuteSqlRaw`** in app code. The only direct `SqlConnection` usage is out-of-band tooling that EF has
no model for: connecting to `master` to DROP/CREATE/RESTORE the database (a standalone db tool), and an
installer connection-test probe. Keeping raw ADO out of the app (and confined to lifecycle tools) means
one query model to reason about; it's EF everywhere that matters.

## 8. Testing against LocalDB with per-fixture databases

Dev/test use `(localdb)\MSSQLLocalDB`. Test fixtures **create a uniquely-named database per fixture**
(`db.Database.Migrate()` on setup), run against real SQL Server semantics, and drop it on teardown — so
suites are isolated and exercise the actual provider (filtered indexes, collation, `datetime2`) rather
than the InMemory provider, which silently diverges from SQL Server on exactly those points.
