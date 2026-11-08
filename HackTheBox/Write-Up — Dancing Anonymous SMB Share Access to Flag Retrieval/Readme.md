# Write-Up — Dancing: Anonymous SMB Share Access to Flag Retrieval

> **Type:** Network / Service Penetration Test (HTB Starting Point)
> **Target:** `10.129.1.12` (`Dancing`)
> **Tools:** Nmap, smbclient
> **Final Result:** Unauthenticated file access via a misconfigured, blank-credential SMB share

---

## 1. Introduction

The target is a beginner-tier lab machine from Hack The Box's Starting Point track, running a Windows host with **SMB (Server Message Block)** exposed. The objective was to enumerate the host, identify accessible shares, and retrieve a flag file.

This box illustrates one of the most common SMB-specific findings:

- **Administrative shares** (`ADMIN$`, `C$`) present but correctly locked down.
- A **custom, human-created share** (`WorkShares`) left accessible with blank credentials — a classic case of a manually configured resource bypassing the security posture applied elsewhere.

No exploit code is required; the vulnerability is a share-permission misconfiguration reachable with a standard SMB client.

---

## 2. Reconnaissance

### 2.1 Connectivity Check

```bash
ping 10.129.1.12
```

Successful replies confirmed the VPN tunnel was up and the target was reachable.

### 2.2 Port Scan — Nmap

```bash
nmap -sV 10.129.1.12
```

Result:

```
PORT     STATE SERVICE       VERSION
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds?
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
Service Info: OS: Windows
```

| Port | Service | Version |
|------|---------|---------|
| 135/tcp | MSRPC | Microsoft Windows RPC |
| 139/tcp | NetBIOS-SSN | Microsoft Windows netbios-ssn |
| 445/tcp | SMB (microsoft-ds) | unidentified by version probe |
| 5985/tcp | WinRM (HTTPAPI) | Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP) |

Analysis: The combination of **139/445 (SMB/NetBIOS)** confirms this is a Windows host offering file/share services, exactly as expected — SMB operates at the Application/Presentation layer and typically rides over NetBIOS (NBT), which is why both appear together. **5985 (WinRM)** is also notable as a potential remote-management foothold if valid credentials are later obtained, but the immediate, credential-free path is SMB enumeration.

---

## 3. Vulnerability Analysis

### 3.1 Identifying the Flaw

Rather than searching for an SMB version-specific exploit (e.g., EternalBlue-class bugs), the service was enumerated for accessible shares first, since the version probe for port 445 returned no confident banner. Shares were tested for **guest/anonymous authentication** — a common misconfiguration where a share is created ad hoc by an administrator without applying the same access controls as the built-in administrative shares.

### 3.2 The Vulnerability

- **CVE:** N/A — this is a configuration weakness, not a software bug
- **Type:** Anonymous/blank-credential SMB share access misconfiguration
- **Authentication:** Present on `ADMIN$`/`C$` (properly denied); absent/bypassable on the custom `WorkShares` share
- **Impact:** Read and download access to files stored on the exposed share, without valid domain or local credentials

---

## 4. Initial Access

### 4.1 Enumerating Available Shares

```bash
smbclient -L 10.129.1.12
```

Prompted for a password against the local machine's username (SMB requires a username field even for anonymous attempts); left blank and pressed Enter.

Four shares were listed:

| Share | Type | Notes |
|-------|------|-------|
| `ADMIN$` | Administrative | Hidden share for remote admin access to the system volume |
| `C$` | Administrative | Root of the `C:\` volume |
| `IPC$` | Inter-process | Named-pipe communication, not browsable as files |
| `WorkShares` | Custom | Non-default, manually created share |

### 4.2 Testing Each Share

```bash
smbclient \\\\10.129.1.12\\ADMIN$
```
Result: `NT_STATUS_ACCESS_DENIED`

```bash
smbclient \\\\10.129.1.12\\C$
```
Result: `NT_STATUS_ACCESS_DENIED`

```bash
smbclient \\\\10.129.1.12\\WorkShares
```
Result: **Login succeeded with a blank password.**

### 4.3 Foothold Confirmation

The prompt changed to `smb: \>`, confirming an interactive session on the `WorkShares` share.

```bash
ls
```

```
Amy.J
James.P
```

### 4.4 Flag / Evidence

Navigating into `Amy.J`:

```bash
get worknotes.txt
```

Navigating into `James.P`:

```bash
get flag.txt
```

```bash
exit
```

Reading the retrieved files locally:

```bash
cat flag.txt
```

returned the lab's hash-value flag, submitted to the Starting Point page to mark the machine as complete. `worknotes.txt` contained internal notes hinting at other services/lateral-movement opportunities — not required for this stage, but a reminder that user-created shares often leak more than intended.

> **Severity: High.** Unauthenticated read/download access to a share containing user directories and internal notes, on a host that also exposes WinRM — a chained risk if any harvested notes had contained credentials.

---

## 5. Full Attack Chain

```
[ Recon: Nmap ]
       │
       ▼
[ SMB (445), NetBIOS (139), WinRM (5985) identified ]
       │
       ▼
[ smbclient -L → four shares enumerated ]
       │
       ▼
[ ADMIN$ → denied ]
       │
       ▼
[ C$ → denied ]
       │
       ▼
[ WorkShares → blank-credential login SUCCESS ]
       │
       ▼
[ Amy.J/worknotes.txt and James.P/flag.txt retrieved ]
```

**Lesson:** The built-in administrative shares were correctly locked down — the weak point was a custom share created outside that baseline configuration. Human-made resources are consistently the ones that drift from policy.

---

## 6. Remediation Summary

| # | Finding | Severity | Remediation |
|---|---------|----------|--------------|
| 1 | `WorkShares` custom SMB share accessible with blank/guest credentials | **High** | Explicitly disable guest/anonymous access on all custom shares (`Set-SmbServerConfiguration -EnableSMB1Protocol/AllowInsecureGuestAuth $false` and share-level ACLs). Require authenticated, least-privilege access per user/group. |
| 2 | Internal notes (`worknotes.txt`) stored in a broadly accessible location | **Medium** | Store operational/internal notes in access-controlled systems (wikis, ticketing) rather than open file shares; audit shares periodically for sensitive content. |
| 3 | SMB service version not confidently fingerprinted, increasing unknowns during assessment | **Low** | Ensure banner/version information is available to authorized scanning tools for internal patch-audit purposes, while still minimizing external exposure. |
| 4 | WinRM (5985) exposed alongside SMB, widening the impact of any credentials found on shares | **Medium** | Restrict WinRM to management network segments/jump hosts only; do not expose it on the same broad network path as general file-sharing services. |

---

## 7. Conclusion

This box is a compact illustration of a principle that scales directly to production Windows environments:

1. **Default hardening doesn't cover manually created resources.** `ADMIN$` and `C$` were properly locked down, but `WorkShares` — created for a practical business reason — was not brought up to the same standard.
2. **Anonymous/guest SMB access remains a fast, reliable win for attackers.** No exploit, no credential cracking — just testing the built-in "try nothing" path first.

The fix, as with most findings of this class, is governance: every share, however casually created, needs to inherit the same access-control baseline as the system's default shares.
