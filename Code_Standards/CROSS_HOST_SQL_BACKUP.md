# Cross-Host SQL Backup — How It Works and How to Reproduce It

A reference for backing up a SQL Server database **to a path on a different machine than the SQL
engine** — written for the Facility Dashboard Platform's implementation, but deliberately general:
the technique applies to any service that needs a remote engine's `.bak` to land on its own disk.

The reference implementation is inlined in §7 of this document (sources:
`src/Dashboards.Host/Services/RemoteBackupSupport.cs` and `BackupService.RunBackupAsync` — the
inline copies below are complete enough to port without opening the repository).

---

## 1. The fundamental constraint

`BACKUP DATABASE ... TO DISK = '<path>'` is executed **inside the SQL Server engine process, on
the engine's host, under the engine's service account**. The client that issues the T-SQL never
touches the file. Three consequences:

1. **Drive letters resolve on the SQL host.** `D:\backups` means the SQL machine's D: — if that's
   a DVD drive or doesn't exist, you get *"Cannot open backup device ... operating system error
   21 (The device is not ready)"* or error 3 (path not found), regardless of what exists on the
   client machine.
2. **Permissions are the engine account's**, not yours and not the client service's. Opening up a
   folder's ACLs on the *client* machine changes nothing.
3. **Mapped drive letters never work** — drive mappings belong to interactive logon sessions;
   services don't have them on either side.

The only way a remote engine can write to another machine's disk is through a **UNC share** on
that machine, with the engine's network identity granted write access.

## 2. Who the engine *is* on the network

The grant target depends on how the SQL service logs on:

| SQL service logon | Authenticates over the network as | Grant this |
| --- | --- | --- |
| `LocalSystem`, `NT Service\MSSQLSERVER` (virtual account), `NetworkService` | The SQL host's **domain computer account** | `DOMAIN\SQLHOST$` |
| Domain service account | Itself | `DOMAIN\svc-sql` |
| Group managed service account (gMSA — required for failover clusters) | Itself | `DOMAIN\sqlcluster$` |
| Any account on a **workgroup** (non-domain) host | Anonymous — effectively nothing | *Not workable; don't try* |

Notes:
- For a **failover cluster instance**, `SERVERPROPERTY('MachineName')` returns the *virtual
  network name*, but the SMB traffic originates from the **physical node**
  (`SERVERPROPERTY('ComputerNamePhysicalNetBIOS')`) — when granting a computer account, grant the
  physical node's (or all nodes', for failover coverage).
- The service account can be read from `sys.dm_server_services` (needs VIEW SERVER STATE);
  when that's not grantable, the computer-account fallback covers the LocalSystem/virtual cases.
- Domain computer accounts are members of *Authenticated Users*, so a share open to Authenticated
  Users would also work — but a single-principal ACL is the right posture and costs nothing.

## 3. The pattern, end to end

What the platform automates (and what you'd do by hand elsewhere):

1. **Detect whether the engine is off-box.**
   ```sql
   SELECT SERVERPROPERTY('MachineName'),
          SERVERPROPERTY('ComputerNamePhysicalNetBIOS'),
          DEFAULT_DOMAIN();
   ```
   Compare against the local machine name (LocalDB and `localhost` are local; a cluster's virtual
   name correctly classifies as remote). Local engine → skip everything below and write directly.

2. **Make sure SMB can serve.** The Windows **Server** service (`LanmanServer`) must be running on
   the receiving host.

3. **Open the door — narrowly.** Inbound firewall rule on the receiving host: TCP 445, **remote
   address scoped to the SQL host's resolved IPs only**, Domain profile only, with a recognizable
   name. Confirm-or-create on every run (and refresh the address scope — clusters fail over,
   machines re-IP); persistent rather than ephemeral, because rules deleted "after the run" become
   debris when a run dies mid-flight. A host rule cannot fix switch/firewall segmentation between
   production cells — if the network blocks 445, that has to be opened by the network team.
   ```
   netsh advfirewall firewall add rule name="<name>" dir=in action=allow ^
         protocol=TCP localport=445 profile=domain enable=yes remoteip=<sql-host-ips>
   ```

4. **Create a hidden, single-principal staging share** on the receiving host (requires local
   admin — LocalSystem qualifies):
   ```
   net share fd-backup$="<staging-folder>" /GRANT:"DOMAIN\SQLHOST$",CHANGE
   icacls "<staging-folder>" /grant "DOMAIN\SQLHOST$":(OI)(CI)M
   ```
   The `$` suffix hides it from casual browsing. Delete any leftover share of the same name first
   (a crashed prior run may have stranded one).

5. **Run the backup to the UNC**, with progress:
   ```sql
   BACKUP DATABASE [FacilityDashboards]
     TO DISK = '\\RECEIVINGHOST\fd-backup$\FacilityDashboards.bak'
     WITH INIT, COPY_ONLY, STATS = 10;
   ```
   - `COPY_ONLY` keeps this out of the DBA's differential/log backup chain — important wherever a
     DBA team manages the real backup regimen (clusters especially).
   - `STATS = 10` emits percent-complete informational messages; a client can observe them via
     the connection's InfoMessage event (or poll `sys.dm_exec_requests.percent_complete` from a
     second connection for long backups).

6. **Finalize locally.** The share *is* a local folder on the receiving host, so the `.bak` is
   already on the right disk — move it (`File.Move`, instant on the same volume) into the final
   backup set, then `net share fd-backup$ /DELETE`.

## 4. Failure modes worth knowing in advance

| Symptom | Actual cause |
| --- | --- |
| *OS error 21 (device is not ready)* | The engine resolved a drive letter on **its** host to removable/absent media (classic: D: is the SQL host's DVD drive). |
| *OS error 3 (path not found)* | Path doesn't exist on the SQL host; or UNC share name typo. |
| *OS error 5 (access denied)* | The engine's network identity isn't granted on the share/NTFS — check the table in §2; remember it's the **physical node's** computer account for clusters running virtual accounts. |
| *OS error 53/64/1231 (network path not found / network name deleted)* | SMB blocked between hosts (445), Server service not running on the receiver, or name resolution failure. |
| Backup works, then breaks after cluster failover | The firewall scope or computer-account grant named the old physical node; re-scope/grant (the platform refreshes the firewall scope each run; grants to a gMSA avoid the issue entirely). |
| *OS error 1396 (Logon Failure: the target account name is incorrect)* | **Kerberos**, not permissions — the failure happens before the share ACL is consulted. See §4.1. |

### 4.1 Diagnosing OS error 1396 (Kerberos) — a field-tested walkthrough

When the engine opens `\\HOST\share`, Windows requests a Kerberos ticket for `cifs/HOST`.
Error 1396 means the machine that answered at that name could not decrypt the ticket — the
name, *as resolved from the SQL host*, doesn't lead to the computer account AD believes owns
it. Causes, in field-likelihood order: stale DNS (the name resolves to a recycled IP now held
by a different machine), a cloned/renamed VM with a duplicate or mismatched computer
account/SPN, a broken machine-account secure channel (snapshot rollbacks), or a **stale cached
ticket** in the engine's logon session left over from any of the above after the underlying
problem has healed.

Work through these *from the SQL host* unless noted:

1. `nslookup <receiver-host>` — every returned address must appear in `ipconfig` **on the
   receiver**. Mismatch = stale DNS; fix with `ipconfig /registerdns` on the receiver.
2. On the receiver, from an **elevated** prompt: `nltest /sc_verify:<DOMAIN>`. Note that this
   returns `ERROR_ACCESS_DENIED` from a non-elevated prompt even when the channel is healthy —
   elevate before believing a failure. Repair in place with PowerShell
   `Test-ComputerSecureChannel -Repair -Credential DOMAIN\admin` (no unjoin/rejoin needed).
3. `setspn -Q HOST/<receiver-host>` from any domain box — expect entries owned by the
   receiver's computer account. (`cifs/...` returning "No such SPN found" is **normal**: CIFS
   is served by the machine's `HOST/` SPN; only query `cifs/` explicitly to detect a *rogue*
   registration.) Duplicates: `setspn -X`, remove with `setspn -D`.
4. The decisive scope test: `dir \\<receiver-host>\admin$` as an interactive domain user on
   the SQL host. If this lists (or merely access-denies), host-to-host Kerberos is healthy
   **now**, and the failure is confined to the engine's logon session — almost certainly a
   stale cached ticket. Tickets cache per logon session (~10 h); a fresh interactive logon
   doesn't share the engine's cache, which is exactly the asymmetry observed.
5. Fix for the cached ticket: restart the SQL Server service (discards its session's ticket
   cache), or surgically `klist sessions` → `klist -li <LUID-of-engine-session> purge` from an
   elevated prompt. Then re-run the backup.

This exact sequence resolved the first live two-host deployment: DNS, SPNs, and the secure
channel all checked healthy; the interactive `admin$` test passed; a stale ticket in the
engine's session (after earlier VM snapshot activity) was the culprit, and a service restart
fixed it.

## 5. How the platform decides (decision tree)

```
SQL engine local (incl. LocalDB)      → engine writes .bak straight into the backup set folder
Backup target is already a UNC        → engine writes the UNC directly
                                         (grant the share to the engine's identity per §2 — manual, one-time)
SQL engine remote + target local path → automated: LanmanServer check → scoped 445 rule →
                                         hidden share granted to one principal → BACKUP WITH STATS →
                                         local move into the set → share removed
```

The grant principal is auto-detected (§2), overridable at **Config → SQL backup grant account**
for sites where the detection isn't permitted or the topology is unusual. Every share and
firewall action is recorded in the platform's diagnostic log.

## 6. Operational prerequisites checklist (for a new site)

- [ ] SQL host is domain-joined (workgroup hosts cannot authenticate to the share).
- [ ] Network path SQL host → dashboard host permits TCP 445 (network ACLs, not just host firewalls).
- [ ] Dashboard service runs with local admin rights (LocalSystem default) — needed to create
      shares and firewall rules.
- [ ] For clusters: know the instance's service account (gMSA/domain account) — set it as the
      grant override, or grant will fall back to the physical node's computer account and must be
      revisited if the auto-detect query is not permitted.
- [ ] The backup target volume has space for: database `.bak` + full media tree × retention count.
- [ ] Decide whether the DBA team's backup regimen makes the database portion redundant — if so,
      a future "external" backup mode (media + manifest only) is the cleaner fit; see the design
      discussion in the session notes.

---

## 7. Reference implementation (C#, inlined)

Complete working code, copied from the platform so this document stands alone. Dependencies:
`Microsoft.Data.SqlClient`, `System.ServiceProcess.ServiceController`, and Windows (uses
`net.exe`, `netsh.exe`, and NTFS ACL APIs). The `DiagnosticsService` calls are the platform's
audit log — substitute your own logging.

### 7.1 Orchestration — the decision tree inside the backup run

```csharp
// Inside the backup routine. `connection` is an open SqlConnection to the target database;
// `targetPath` is the operator-configured backup destination; `backupDir` is this run's
// timestamped set folder beneath it.
var location = await RemoteBackupSupport.DetectAsync(connection, ct);

if (location.IsLocal || targetPath.StartsWith("\\\\"))
{
    // Engine on this host, or target already a UNC: the engine writes the file directly.
    var bakPath = Path.Combine(backupDir, databaseName + ".bak");
    await ExecuteEngineBackupAsync(connection, databaseName, bakPath, ct);
}
else
{
    // Remote engine + local target: stage via a temporary single-principal share.
    var principal = string.IsNullOrWhiteSpace(configuredGrantAccount)
        ? location.AutoGrantPrincipal          // DOMAIN\SQLHOST$ or the engine's domain account/gMSA
        : configuredGrantAccount.Trim();       // operator override

    RemoteBackupSupport.EnsureServerService();
    await RemoteBackupSupport.EnsureFirewallRuleAsync(location.PhysicalHost, diagnostics, ct);

    var stagingPath = Path.Combine(targetPath, ".fd-backup-staging");
    using var share = await RemoteBackupSupport.CreateShareAsync(stagingPath, principal, diagnostics);

    var uncBakPath = share.UncPath + "\\" + databaseName + ".bak";
    await ExecuteEngineBackupAsync(connection, databaseName, uncBakPath, ct);

    // The share maps to local disk: finalize with a local move, no second copy.
    File.Move(Path.Combine(stagingPath, databaseName + ".bak"),
              Path.Combine(backupDir, databaseName + ".bak"), overwrite: true);
}   // disposing `share` removes it
```

### 7.2 Running the backup with progress

```csharp
/// Runs BACKUP DATABASE with progress (STATS) streamed to the log; wraps engine errors with context.
private async Task ExecuteEngineBackupAsync(SqlConnection connection, string databaseName, string bakPath, CancellationToken ct)
{
    SqlInfoMessageEventHandler? onMessage = (_, e) =>
    {
        var text = e.Message;
        if (text.Contains("percent", StringComparison.OrdinalIgnoreCase))
        {
            _logger.LogInformation("Database backup progress: {Message}", text.Trim());
        }
    };
    connection.InfoMessage += onMessage;
    try
    {
        await using var command = connection.CreateCommand();
        command.CommandText = "BACKUP DATABASE [" + databaseName + "] TO DISK = @path WITH INIT, COPY_ONLY, STATS = 10";
        command.CommandTimeout = 600;
        command.Parameters.Add(new SqlParameter("@path", bakPath));
        await command.ExecuteNonQueryAsync(ct);
    }
    catch (SqlException ex)
    {
        // BACKUP DATABASE is executed by the SQL Server ENGINE on its own host under its own
        // service account — a path that works for this service can still be invalid there.
        throw new InvalidOperationException(
            ex.Message +
            " — Note: SQL Server itself writes the database backup, so the target must exist on the SQL Server's host " +
            "and be writable by the SQL Server service account; drive letters refer to the SQL host's own drives.", ex);
    }
    finally
    {
        connection.InfoMessage -= onMessage;
    }
}
```

### 7.3 The mechanics class — detection, firewall, share lifecycle

```csharp
using System.Diagnostics;
using System.Net;
using System.Security.AccessControl;
using System.Security.Principal;
using Microsoft.Data.SqlClient;

/// <summary>
/// Cross-host SQL backup support. BACKUP DATABASE is executed by the SQL engine on ITS host, so a
/// remote engine cannot write to this host's drive letters. When the engine is off-box and the
/// backup target is a local path, this class automates the standard workaround: a temporary
/// hidden SMB share on this host, ACL'd to exactly the principal the engine authenticates as
/// (its domain service account/gMSA, or the SQL host's computer account when it runs as
/// LocalSystem / a virtual account), a scoped firewall allowance for SMB from the SQL host,
/// the backup to the UNC, and a local move into the final backup set.
/// </summary>
public static class RemoteBackupSupport
{
    public const string ShareName = "fd-backup$";
    public const string FirewallRuleName = "Facility Dashboards - SQL backup (SMB-In)";

    public class SqlLocation
    {
        public bool IsLocal { get; set; }
        /// <summary>The physical node currently running the instance (cluster-aware).</summary>
        public string PhysicalHost { get; set; } = string.Empty;
        /// <summary>The instance's reported machine name (a cluster's virtual network name).</summary>
        public string MachineName { get; set; } = string.Empty;
        /// <summary>The principal the engine presents over the network, best-effort detected.</summary>
        public string AutoGrantPrincipal { get; set; } = string.Empty;
    }

    /// <summary>Asks the server where it runs and how it authenticates remotely.</summary>
    public static async Task<SqlLocation> DetectAsync(SqlConnection connection, CancellationToken ct)
    {
        var location = new SqlLocation();

        if (connection.DataSource.StartsWith("(localdb)", StringComparison.OrdinalIgnoreCase))
        {
            location.IsLocal = true;
            location.PhysicalHost = Environment.MachineName;
            location.MachineName = Environment.MachineName;
            return location;
        }

        await using (var command = connection.CreateCommand())
        {
            command.CommandText =
                "SELECT CAST(SERVERPROPERTY('MachineName') AS nvarchar(128)), " +
                "CAST(SERVERPROPERTY('ComputerNamePhysicalNetBIOS') AS nvarchar(128)), " +
                "DEFAULT_DOMAIN()";
            await using var reader = await command.ExecuteReaderAsync(ct);
            await reader.ReadAsync(ct);
            location.MachineName = reader.IsDBNull(0) ? string.Empty : reader.GetString(0);
            location.PhysicalHost = reader.IsDBNull(1) ? location.MachineName : reader.GetString(1);
            var domain = reader.IsDBNull(2) ? string.Empty : reader.GetString(2);

            // LocalSystem / NT Service\ / NetworkService authenticate over the network as the SQL
            // HOST'S COMPUTER ACCOUNT — the physical node's, which is what touches the share.
            location.AutoGrantPrincipal = domain + "\\" + location.PhysicalHost + "$";
        }

        // A domain service account or gMSA (mandatory for clusters) authenticates as itself.
        // Reading it needs VIEW SERVER STATE; fall back silently to the computer account.
        try
        {
            await using var command = connection.CreateCommand();
            command.CommandText =
                "SELECT TOP 1 service_account FROM sys.dm_server_services WHERE servicename LIKE 'SQL Server (%'";
            var account = (await command.ExecuteScalarAsync(ct)) as string;
            if (!string.IsNullOrEmpty(account) && account.Contains('\\'))
            {
                var authority = account.Split('\\')[0].ToUpperInvariant();
                if (authority != "NT AUTHORITY" && authority != "NT SERVICE" && authority != "BUILTIN")
                {
                    location.AutoGrantPrincipal = account;
                }
            }
        }
        catch (SqlException)
        {
            // No VIEW SERVER STATE — the computer-account fallback stands.
        }

        var local = Environment.MachineName;
        location.IsLocal =
            string.Equals(location.PhysicalHost, local, StringComparison.OrdinalIgnoreCase) ||
            string.Equals(location.MachineName, local, StringComparison.OrdinalIgnoreCase);
        return location;
    }

    /// <summary>The SMB server must be running for any share to exist.</summary>
    public static void EnsureServerService()
    {
        try
        {
            using var controller = new System.ServiceProcess.ServiceController("LanmanServer");
            if (controller.Status != System.ServiceProcess.ServiceControllerStatus.Running)
            {
                controller.Start();
                controller.WaitForStatus(System.ServiceProcess.ServiceControllerStatus.Running, TimeSpan.FromSeconds(30));
            }
        }
        catch (Exception ex)
        {
            throw new InvalidOperationException(
                "The Windows 'Server' service (LanmanServer) is not running and could not be started — " +
                "SMB shares are impossible without it: " + ex.Message, ex);
        }
    }

    /// <summary>
    /// Confirms (or creates/updates) the inbound SMB rule, scoped to the SQL host's addresses on
    /// the Domain profile only — never a blanket 445 opening. Persistent by design: ephemeral
    /// rules leave debris when a run dies; a scoped named rule is auditable and idempotent.
    /// A host rule cannot fix network-level segmentation; failures downstream say so.
    /// </summary>
    public static async Task EnsureFirewallRuleAsync(string sqlPhysicalHost, DiagnosticsService diagnostics, CancellationToken ct)
    {
        string remoteIps;
        try
        {
            var addresses = await Dns.GetHostAddressesAsync(sqlPhysicalHost, ct);
            remoteIps = string.Join(",", addresses
                .Where(a => a.AddressFamily == System.Net.Sockets.AddressFamily.InterNetwork ||
                            a.AddressFamily == System.Net.Sockets.AddressFamily.InterNetworkV6)
                .Select(a => a.ToString()));
        }
        catch (Exception ex)
        {
            throw new InvalidOperationException(
                "Could not resolve the SQL host '" + sqlPhysicalHost + "' to scope the firewall rule: " + ex.Message, ex);
        }
        if (remoteIps.Length == 0)
        {
            throw new InvalidOperationException("The SQL host '" + sqlPhysicalHost + "' resolved to no usable addresses.");
        }

        var exists = RunNetsh("advfirewall firewall show rule name=\"" + FirewallRuleName + "\"").ExitCode == 0;
        if (!exists)
        {
            var add = RunNetsh("advfirewall firewall add rule name=\"" + FirewallRuleName + "\" dir=in action=allow " +
                               "protocol=TCP localport=445 profile=domain enable=yes remoteip=" + remoteIps);
            if (add.ExitCode != 0)
            {
                throw new InvalidOperationException(
                    "The scoped SMB firewall rule could not be created (is the service running with administrative rights?): " + add.Output);
            }
            await diagnostics.LogAsync(DiagnosticLevel.Warning, "backup",
                "Created inbound firewall rule '" + FirewallRuleName + "' for TCP 445 from " + remoteIps + " (domain profile).");
        }
        else
        {
            // Keep the scope current if the SQL host moved (cluster failover, re-IP).
            var set = RunNetsh("advfirewall firewall set rule name=\"" + FirewallRuleName + "\" new remoteip=" + remoteIps + " enable=yes");
            if (set.ExitCode != 0)
            {
                throw new InvalidOperationException("The SMB firewall rule exists but could not be updated: " + set.Output);
            }
        }
    }

    /// <summary>
    /// Creates the hidden staging share ACL'd to exactly one principal. Dispose removes it.
    /// </summary>
    public static async Task<ShareHandle> CreateShareAsync(string stagingPath, string grantPrincipal, DiagnosticsService diagnostics)
    {
        Directory.CreateDirectory(stagingPath);

        // NTFS: give the principal Modify on the staging folder (best effort — the share grant is
        // the gate; on some topologies the account resolves only via the share layer).
        try
        {
            var info = new DirectoryInfo(stagingPath);
            var security = info.GetAccessControl();
            security.AddAccessRule(new FileSystemAccessRule(
                new NTAccount(grantPrincipal),
                FileSystemRights.Modify,
                InheritanceFlags.ContainerInherit | InheritanceFlags.ObjectInherit,
                PropagationFlags.None,
                AccessControlType.Allow));
            info.SetAccessControl(security);
        }
        catch (IdentityNotMappedException)
        {
            await diagnostics.LogAsync(DiagnosticLevel.Warning, "backup",
                "NTFS grant for '" + grantPrincipal + "' could not be applied (identity not resolvable from this host); relying on the share-level grant.");
        }

        // Remove any leftover share from a crashed prior run, then create fresh.
        RunNet("share " + ShareName + " /DELETE /Y");
        var create = RunNet("share " + ShareName + "=\"" + stagingPath + "\" \"/GRANT:" + grantPrincipal + ",CHANGE\" /REMARK:\"Facility Dashboards temporary backup staging\"");
        if (create.ExitCode != 0)
        {
            throw new InvalidOperationException(
                "The temporary backup share could not be created (service must run with administrative rights; " +
                "grant principal was '" + grantPrincipal + "'): " + create.Output);
        }

        await diagnostics.LogAsync(DiagnosticLevel.Warning, "backup",
            "Created temporary hidden share \\\\" + Environment.MachineName + "\\" + ShareName +
            " -> " + stagingPath + " granted to '" + grantPrincipal + "'.");
        return new ShareHandle(diagnostics);
    }

    public class ShareHandle : IDisposable
    {
        private readonly DiagnosticsService _diagnostics;
        private bool _disposed;

        public ShareHandle(DiagnosticsService diagnostics)
        {
            _diagnostics = diagnostics;
        }

        public string UncPath
        {
            get { return "\\\\" + Environment.MachineName + "\\" + ShareName; }
        }

        public void Dispose()
        {
            if (_disposed)
            {
                return;
            }
            _disposed = true;
            var delete = RunNet("share " + ShareName + " /DELETE /Y");
            _diagnostics.LogAsync(DiagnosticLevel.Warning, "backup",
                delete.ExitCode == 0
                    ? "Removed temporary backup share " + UncPath + "."
                    : "Temporary backup share " + UncPath + " could not be removed: " + delete.Output).GetAwaiter().GetResult();
        }
    }

    private static (int ExitCode, string Output) RunNetsh(string arguments)
    {
        return RunProcess("netsh.exe", arguments);
    }

    private static (int ExitCode, string Output) RunNet(string arguments)
    {
        return RunProcess("net.exe", arguments);
    }

    private static (int ExitCode, string Output) RunProcess(string fileName, string arguments)
    {
        var psi = new ProcessStartInfo
        {
            FileName = Path.Combine(Environment.SystemDirectory, fileName),
            Arguments = arguments,
            UseShellExecute = false,
            RedirectStandardOutput = true,
            RedirectStandardError = true,
            CreateNoWindow = true
        };
        using var process = Process.Start(psi)!;
        var output = process.StandardOutput.ReadToEnd() + process.StandardError.ReadToEnd();
        process.WaitForExit(30000);
        return (process.ExitCode, output.Trim());
    }
}
```

**Porting notes.** The platform-specific pieces are trivial to swap: `DiagnosticsService.LogAsync`
is an audit log (replace with your logger); the share/rule names are constants; the staging folder
is placed inside the target volume so the final `File.Move` is an instant same-volume rename. If
your service might run without administrative rights, check for that up front — share and firewall
creation both require it — and fail with guidance rather than letting `net.exe` produce an
"Access is denied" five steps in.
