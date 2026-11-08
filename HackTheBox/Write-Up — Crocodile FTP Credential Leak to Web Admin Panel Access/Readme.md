# Write-Up — Crocodile: FTP Credential Leak to Web Admin Panel Access

> **Type:** Network / Web Application Penetration Test (HTB Starting Point)
> **Target:** `10.129.1.15` (`Crocodile`)
> **Tools:** Nmap, FTP (CLI client), Wappalyzer, Gobuster
> **Final Result:** Admin panel access via credentials leaked on an anonymous FTP share, chained to a brute-forceable web login

---

## 1. Introduction

The target is a beginner-tier lab machine from Hack The Box's Starting Point track, running both **FTP** and a **web server**. Unlike single-service boxes, this one demonstrates a **chained attack vector**: information disclosed on one service (FTP) is used to gain access on a completely different service (a web application admin panel).

This box illustrates two findings working together:

- **Anonymous FTP access** exposing files that were never meant to be public — in this case, what appear to be leftover credential/configuration files from setting up another service.
- **A web admin login page**, undiscoverable from the public site navigation, found only through directory brute-forcing, and accessible with the leaked credentials.

---

## 2. Reconnaissance

### 2.1 Connectivity Check

```bash
ping 10.129.1.15
```

Successful replies confirmed the VPN tunnel was up and the target was reachable.

### 2.2 Port Scan — Nmap

```bash
nmap -sV -sC 10.129.1.15
```

Result:

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-syst:
|   STAT:
| FTP server status:
|      Connected to ::ffff:10.10.16.101
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 1
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| -rw-r--r--    1 ftp      ftp            33 Jun 08  2021 allowed.userlist
|_-rw-r--r--    1 ftp      ftp            62 Apr 20  2021 allowed.userlist.passwd
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Smash - Bootstrap Business Template
```

| Port | Service | Version |
|------|---------|---------|
| 21/tcp | FTP | vsftpd 3.0.3 |
| 80/tcp | HTTP | Apache httpd 2.4.41 (Ubuntu) |

Two open ports were identified. The `-sC` script scan was immediately valuable here: it confirmed **anonymous FTP login is allowed** and even listed two files sitting in the FTP root — `allowed.userlist` and `allowed.userlist.passwd` — names strongly suggestive of a username list and a matching password list. The web server was also fingerprinted, revealing an Apache/Ubuntu stack serving a "Smash" Bootstrap business template site.

---

## 3. Vulnerability Analysis

### 3.1 Identifying the Flaw

With anonymous FTP confirmed and two suspicious files already surfaced by the Nmap script scan, the natural next step was to retrieve and read them. If they contained valid credentials, the logical follow-up would be testing those credentials anywhere authentication was required — starting with FTP itself, then the web application.

### 3.2 The Vulnerability

- **CVE:** N/A — this is a configuration/information-disclosure weakness, not a software bug
- **Type:** Anonymous FTP access exposing leftover credential material, chained with a web login lacking brute-force protection
- **Authentication:** Bypassed entirely on FTP (anonymous); satisfied on the web app using leaked, valid credentials
- **Impact:** Administrative access to a web application management panel

---

## 4. Initial Access

### 4.1 Connecting to FTP Anonymously

```bash
ftp 10.129.1.15
```

```
Name: anonymous
Password: [blank]
230 Login successful.
```

### 4.2 Enumerating and Retrieving Files

```bash
dir
```

```
allowed.userlist
allowed.userlist.passwd
```

```bash
get allowed.userlist
get allowed.userlist.passwd
exit
```

### 4.3 Reading the Retrieved Files

```bash
cat allowed.userlist
cat allowed.userlist.passwd
```

The two files together produced a small candidate username list and a matching password list.

### 4.4 Testing Credentials Against FTP Itself

```bash
ftp 10.129.1.15
```

Attempting the harvested username/password combinations for a non-anonymous FTP session returned:

```
530 This FTP server is anonymous only.
```

No elevated FTP access was possible — the credentials were meant for something else.

### 4.5 Web Enumeration

Navigating to `http://10.129.1.15` displayed a storefront-style business template site (**"Smash - Bootstrap Business Template"**, confirmed by the Nmap `http-title` field). A Wappalyzer check confirmed **PHP** as part of the stack, but no obvious attack path from the visible pages alone.

### 4.6 Directory Brute-Forcing

```bash
gobuster dir --url http://10.129.1.15 --wordlist [wordlist_path] -x php,html
```

This surfaced `/login.php` — an administrative login page not linked from the public site navigation.

### 4.7 Foothold Confirmation

Navigating to `http://10.129.1.15/login.php` and testing the small set of harvested username/password combinations succeeded, granting access to a **Server Manager admin panel**.

### 4.8 Flag / Evidence

The admin panel exposed the lab's flag, retrievable from within the authenticated interface, and submitted to the Starting Point page to mark the machine as complete.

> **Severity: High.** Full administrative access to a web management panel, achieved by chaining an anonymous file share disclosure with credential reuse — no exploit code required at any stage.

---

## 5. Full Attack Chain

```
[ Recon: Nmap ]
       │
       ▼
[ FTP (21) anonymous login allowed + HTTP (80) Apache/PHP ]
       │
       ▼
[ ftp anonymous → allowed.userlist / allowed.userlist.passwd retrieved ]
       │
       ▼
[ Credentials tested against FTP itself → rejected (anonymous-only) ]
       │
       ▼
[ Web recon: Wappalyzer confirms PHP, no visible attack path ]
       │
       ▼
[ Gobuster directory brute-force → /login.php discovered ]
       │
       ▼
[ Harvested credentials tested on /login.php → SUCCESS ]
       │
       ▼
[ Server Manager admin panel accessed → flag retrieved ]
```

**Lesson:** Neither service was vulnerable on its own in a classic "exploit" sense. The compromise came entirely from **information leaking across service boundaries** — credentials meant for the web application were left sitting on an anonymously readable FTP share.

---

## 6. Remediation Summary

| # | Finding | Severity | Remediation |
|---|---------|----------|--------------|
| 1 | Anonymous FTP login enabled | **High** | Disable anonymous FTP access unless there is an explicit, isolated public-file-drop use case; audit any anonymous share regularly for unintended content. |
| 2 | Credential/userlist files stored in a publicly readable FTP directory | **Critical** | Never store credentials, password lists, or configuration secrets in a location reachable by anonymous or low-privilege accounts. Use a secrets manager, and restrict file permissions to the specific service account that needs them. |
| 3 | Web admin login page (`/login.php`) discoverable via simple directory brute-force, with no apparent rate limiting | **High** | Enforce account lockout / exponential backoff on login attempts; consider moving administrative interfaces off the public-facing vhost or placing them behind a VPN/IP allowlist. |
| 4 | Credential reuse between service-provisioning material and the live web application | **Medium** | Use unique, randomly generated credentials per service/account; rotate any credentials that may have been used during initial setup before the system goes live. |
| 5 | Directory/file names on FTP directly hinted at their sensitive contents | **Low** | Avoid descriptive naming for sensitive files (`allowed.userlist.passwd`) that telegraphs their purpose to anyone browsing a share. |

---

## 7. Conclusion

This box is a compact illustration of a principle that shows up constantly in real engagements:

1. **The most damaging findings are often chains, not single bugs.** Neither the FTP misconfiguration nor the web login was independently "game over" — together, they were.
2. **Anonymous file shares are a recurring source of credential leakage.** Enumeration of every reachable service, however innocuous it looks, regularly pays off far more than searching for a flashy exploit.

The fix, as with most findings of this class, is operational hygiene across service boundaries: no credentials on open shares, unique credentials per service, and basic protections on every login form — administrative or not.
