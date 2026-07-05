# Write-Up — Fawn: Anonymous FTP Access to Flag Retrieval

> **Type:** Network / Service Penetration Test (HTB Starting Point)
> **Target:** `10.129.77.38` (`Fawn`)
> **Tools:** Nmap, FTP (CLI client)
> **Final Result:** Unauthenticated file access via a misconfigured Anonymous FTP account

---

## 1. Introduction

The target is a beginner-tier lab machine from Hack The Box's Starting Point track, running a single exposed **FTP** service. The objective was to enumerate the host, identify the exposed service, and retrieve a flag file.

This box illustrates one of the most common FTP-specific findings:

- **Anonymous authentication left enabled**, allowing any client to log in without valid credentials.
- **No transport encryption** (plain FTP, not FTPS/SFTP), meaning any traffic — including file contents — would be readable to a Man-in-the-Middle.

No exploit code is required; the vulnerability is a service-level misconfiguration reachable with a stock FTP client.

---

## 2. Reconnaissance

### 2.1 Connectivity Check

```bash
ping 10.129.77.38
```

Successful replies confirmed the VPN tunnel was up and the target was reachable.

### 2.2 Port Scan — Nmap

```bash
nmap -sV 10.129.77.38
```

Result:

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
Service Info: OS: Unix
```

| Port | Service | Version |
|------|---------|---------|
| 21/tcp | FTP | vsftpd 3.0.3 |

Only a single port was found open (999 others reset/closed), making FTP the sole path forward. The `-sV` (version detection) switch was used specifically to identify the FTP daemon version, since a known-vulnerable version could point toward a software exploit rather than a configuration issue.

**Note on vsftpd 3.0.3:** this version postdates the infamous backdoored 2.3.4 release (CVE-2011-2523) and is not itself known to carry a pre-auth RCE. It was treated as a stable, unexploitable-by-CVE build for this engagement — attention was instead directed toward configuration weaknesses, which is exactly what turned up.

Analysis: Plain FTP on port 21, with no companion SSH tunnel or TLS wrapper visible, is inherently a red flag — the protocol transmits credentials and file data in cleartext by design.

---

## 3. Vulnerability Analysis

### 3.1 Identifying the Flaw

Rather than searching for a version-specific exploit, the FTP service was tested for a very common misconfiguration: **anonymous login**. Many FTP daemons ship with, or are manually configured to allow, an `anonymous` account that accepts any password (traditionally an email address, but effectively disregarded by the server).

### 3.2 The Vulnerability

- **CVE:** N/A — this is a configuration weakness, not a software bug
- **Type:** Anonymous/unauthenticated access misconfiguration
- **Authentication:** Present, but trivially bypassed via the `anonymous` account
- **Impact:** Read access to files stored on the FTP service, without valid credentials

---

## 4. Initial Access

### 4.1 Connecting to the Service

```bash
ftp 10.129.77.38
```

At the `Name` prompt:

```
Name: anonymous
Password: [anything]
```

Login succeeded — the server ignored the password entirely.

### 4.2 Foothold Confirmation

```bash
help
```

returned the list of available FTP commands, confirming an interactive session was established with standard navigation commands (`ls`, `cd`, `get`) available.

```bash
ls
```

```
flag.txt
```

### 4.3 Flag / Evidence

```bash
get flag.txt
```

The file was downloaded to the local working directory. After exiting the FTP session:

```bash
cat flag.txt
```

returned the lab's hash-value flag, submitted to the Starting Point page to mark the machine as complete.

> **Severity: Medium.** No authentication bypass of a privileged account occurred, but unauthenticated read access to server-hosted files over an unencrypted channel is a meaningful exposure — especially if sensitive files are ever placed on the share.

---

## 5. Full Attack Chain

```
[ Recon: Nmap ]
       │
       ▼
[ Only FTP (21/tcp) open ]
       │
       ▼
[ Test for Anonymous login misconfiguration ]
       │  → anonymous / [any password]: SUCCESS
       ▼
[ Interactive FTP session established ]
       │
       ▼
[ ls → flag.txt located ]
       │
       ▼
[ get flag.txt → downloaded and read locally ]
```

**Lesson:** No credentials were guessed or cracked — the service handed out access by design, because the Anonymous account was never disabled. This is one of the fastest wins an attacker can find during external enumeration.

---

## 6. Remediation Summary

| # | Finding | Severity | Remediation |
|---|---------|----------|--------------|
| 1 | Anonymous FTP login enabled | **High** | Disable the anonymous account in the FTP daemon configuration unless there is an explicit business need for public, unauthenticated file drops (and even then, restrict it to a dedicated, isolated directory with no sensitive content). |
| 2 | FTP service transmits data in cleartext | **Medium** | Migrate to FTPS (FTP over SSL/TLS) or SFTP (FTP tunneled over SSH) so credentials and file contents are encrypted in transit. |
| 3 | Sensitive/flag-equivalent files stored in a world-readable share | **Medium** | Apply least-privilege file permissions on any FTP root directory; never store credentials, configuration backups, or sensitive data in a location reachable by anonymous or low-privilege accounts. |
| 4 | No visible access logging/alerting on anonymous logins | **Low** | Enable and monitor FTP access logs; alert on anonymous session activity, which is often a strong indicator of reconnaissance or automated scanning. |

---

## 7. Conclusion

This box is a compact illustration of a principle that scales directly to production environments:

1. **"Requires authentication" is not the same as "is secure."** A login prompt can still hand out access for free if a default or convenience account like `anonymous` is left enabled.
2. **Unencrypted legacy protocols compound the risk.** Even where anonymous access is intentional (e.g., a public file drop), running it over plaintext FTP instead of FTPS/SFTP exposes every session to interception.

The fix, as with most findings of this class, is a five-minute configuration change — not a patch, not an exploit mitigation, just turning a default off.
