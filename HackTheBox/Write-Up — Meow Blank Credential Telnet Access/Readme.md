# Write-Up — Meow: Blank Credential Telnet Access

> **Type:** Network / Service Penetration Test (HTB Starting Point)
> **Target:** `10.129.251.148` (`Meow`)
> **Tools:** Nmap, Telnet
> **Final Result:** Root-level shell access via a blank-password root account on an exposed Telnet service

---

## 1. Introduction

The target is a beginner-tier lab machine from Hack The Box's Starting Point track. The objective was to enumerate the host, identify any exposed services, and gain access to a flag file — the standard "root the box" goal for this style of lab.

This box illustrates one of the most common (and most preventable) findings in real-world assessments:

- An **obsolete, cleartext remote-management protocol** left exposed to the network.
- An **administrative account with no password set at all**.

Neither requires custom tooling or exploit development — just patience and a short list of default usernames.

---

## 2. Reconnaissance

### 2.1 Connectivity Check

Before scanning, basic reachability was confirmed:

```bash
ping 10.129.251.148
```

Four successful replies confirmed the VPN/lab connection was stable, and the target was reachable.

### 2.2 Port Scan — Nmap

```bash
nmap -sV 10.129.251.148
```

Result:

| Port | Service | Notes |
|------|---------|-------|
| 23/tcp | Telnet | Presents a "Hack The Box" login banner |

Only a single open port was identified. `-sV` was used specifically to fingerprint the service, since some hosts run non-standard services on standard ports — in this case, the port matched its expected default service.

Observation: **Telnet** is a legacy, unencrypted remote-administration protocol. Its presence alone is a finding — all traffic, including credentials, is sent in plaintext. With no other ports open, this was the only viable path forward.

---

## 3. Vulnerability Analysis

### 3.1 Attack Surface

With a single exposed service and no other footholds available, the logical next step was to attempt authentication rather than search for a software vulnerability. Telnet daemons are frequently misconfigured with:

- Default vendor credentials left unchanged.
- Administrative accounts created without a password, for "convenience" during setup or troubleshooting.

### 3.2 Candidate Usernames

Rather than brute-forcing a full wordlist, a short manual list of high-probability administrative usernames was tried first, with an empty password:

```
admin
administrator
root
```

- **Type:** Blank/weak credential misconfiguration
- **Authentication:** Present, but trivially bypassed
- **Impact:** Full interactive shell access on the host

---

## 4. Initial Access

### 4.1 Connecting to the Service

```bash
telnet 10.129.251.148
```

This returned the Hack The Box login banner and prompted for a username.

### 4.2 Credential Attempts

The first two candidates (`admin`, `administrator`) failed. The third attempt succeeded:

```
login: root
password: [blank — Enter pressed]
```

```
Welcome to Meow
$
```

Foothold obtained as **`root`** — no password was set on the account at all.

### 4.3 Post-Login Enumeration

```bash
ls
```

```
flag.txt
```

### 4.4 Flag Retrieval

```bash
cat flag.txt
```

The command returned the lab's hash-value flag, submitted to the Starting Point page to mark the machine as complete.

> **Severity: Critical.** Unauthenticated (blank-password) administrative access to a cleartext remote-management service, with root-level privileges granted immediately on login — no privilege escalation required.

---

## 5. Full Attack Chain

```
[ Recon: Nmap ]
       │
       ▼
[ Only Telnet (23/tcp) open ]
       │
       ▼
[ No other services to explore — auth is the only path ]
       │
       ▼
[ Manual test of common admin usernames ]
       │  → admin: fail
       │  → administrator: fail
       │  → root: SUCCESS (blank password)
       ▼
[ Interactive shell as root ]
       │
       ▼
[ ls → flag.txt located ]
       │
       ▼
[ cat flag.txt → flag captured ]
```

**Lesson:** No exploit code was needed. The entire chain was: find the one open port, recognize it as a legacy protocol prone to weak configuration, and try the most obvious credentials first. This mirrors a large share of real low-hanging-fruit findings in external assessments.

---

## 6. Remediation Summary

| # | Finding | Severity | Remediation |
|---|---------|----------|--------------|
| 1 | Telnet (23/tcp) exposed to the network | **High** | Disable Telnet entirely. Replace with SSH for remote administration, which provides encryption and stronger authentication options. |
| 2 | `root` account with no password configured | **Critical** | Enforce a password policy at account creation — no account, especially a privileged one, should ever be created without a credential. Use configuration management to audit for blank/default passwords. |
| 3 | Root-equivalent account directly reachable over remote login | **High** | Disable direct remote login for `root`. Require authentication as a standard user followed by `sudo`/privilege elevation, with logging. |
| 4 | No account lockout / throttling observed on the login service | **Medium** | Implement lockout or rate-limiting after repeated failed login attempts (e.g., `fail2ban`) to slow down credential-guessing attempts. |

---

## 7. Conclusion

This box is a compact illustration of a principle that scales all the way up to enterprise environments:

1. **Legacy protocols are still deployed.** Telnet has had a stronger, encrypted replacement (SSH) for over two decades, yet it still turns up in the wild — often on forgotten management interfaces or IoT-adjacent devices.
2. **Weak or absent credentials remain the single most common way in.** No CVE, no memory corruption, no custom payload — just an administrative account nobody bothered to protect.

The fix here, as with most findings of this class, is operational discipline: retire outdated services, and never let an account exist without a credential.
