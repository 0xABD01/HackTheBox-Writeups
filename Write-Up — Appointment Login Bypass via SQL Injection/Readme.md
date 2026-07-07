# Write-Up — Appointment: Login Bypass via SQL Injection

> **Type:** Web Application Penetration Test (HTB Starting Point)
> **Target:** `10.129.80.100` (`Appointment`)
> **Tools:** Nmap, Gobuster, Web Browser
> **Final Result:** Authentication bypass on a login form via classic SQL Injection

---

## 1. Introduction

The target is a beginner-tier lab machine from Hack The Box's Starting Point track, running a web application with a login form backed by an SQL database. The objective was to enumerate the web service, attempt standard access routes, and ultimately bypass authentication to retrieve a flag.

This box illustrates one of the most enduring findings in web application security:

- **Unparameterized SQL queries** built directly from user-supplied form input.
- **No input validation or sanitization** on special characters (`'`, `#`) that have syntactic meaning in SQL.
- A resulting **authentication bypass** that requires no password guessing at all — just a query the developer never anticipated.

---

## 2. Reconnaissance

### 2.1 Connectivity Check

```bash
ping 10.129.80.100
```

Successful replies confirmed the VPN tunnel was up and the target was reachable.

### 2.2 Port Scan — Nmap

```bash
nmap -sC -sV 10.129.80.100
```

Result:

| Port | Service | Version |
|------|---------|---------|
| 80/tcp | HTTP | Apache httpd 2.4.38 |

Only a single port was found open. `-sV` confirmed the exact Apache version; `-sC` ran default enumeration scripts. A quick check of public vulnerability databases for Apache 2.4.38 turned up nothing directly exploitable — the version itself was not the way in.

### 2.3 Web Enumeration

Navigating to `http://10.129.80.100` in a browser presented a **login form**, with no publicly linked pages beyond it.

### 2.4 Directory Brute-Forcing (Optional Recon)

```bash
gobuster dir --url http://10.129.80.100 --wordlist /usr/share/wordlists/[wordlist_name]
```

Result: only default/standard directories were returned (no non-standard files or hidden endpoints of interest). This step confirmed there was no alternate entry point being missed before focusing on the login form itself.

---

## 3. Vulnerability Analysis

### 3.1 Identifying the Flaw

With no useful hidden directories and no default credentials working, the login form itself became the focus. Since credential guessing (`admin:admin`, `guest:guest`, `root:root`, etc.) failed and brute-forcing risked detection/lockout, the form was tested for **SQL Injection** — a natural next step for any login form backed by a SQL query built from raw user input.

### 3.2 The Vulnerability

- **CVE:** N/A — this is an application-logic/input-validation flaw, not a versioned software bug
- **Type:** SQL Injection (authentication bypass via query manipulation)
- **Authentication:** Present as a login form, but bypassable without any valid password
- **Impact:** Full authentication bypass, gaining access to the authenticated/restricted area of the application

**Root cause (as reconstructed from typical backend logic for this class of vulnerability):**

```php
$sql = "SELECT * FROM users WHERE username='$username' AND password='$password'";
```

Because `$username` and `$password` are inserted directly into the query with no escaping or parameterization, any special character submitted through the form — such as a single quote (`'`) or a comment marker (`#`) — is interpreted as part of the SQL syntax rather than as literal data.

---

## 4. Initial Access

### 4.1 Attempting Default Credentials

```
admin:admin
guest:guest
user:user
root:root
administrator:password
```

All failed — no default credential combination was valid.

### 4.2 Crafting the SQL Injection Payload

Submitted into the login form:

```
Username: admin'#
Password: [anything]
```

This closes the intended string literal early with `'` and comments out the remainder of the query — including the password check — with `#`.

Resulting effective query:

```sql
SELECT * FROM users WHERE username='admin'#' AND password='...'
```

Everything after `#` is treated as a comment, so the query reduces to checking only whether a user named `admin` exists — which it does — satisfying the application's "exactly one row returned" login condition without ever validating a password.

### 4.3 Foothold Confirmation

Submitting the form redirected past the login page into the authenticated area of the application, confirming the bypass succeeded.

### 4.4 Flag / Evidence

The authenticated page displayed the lab's flag value directly, which was submitted to the Starting Point page to mark the machine as complete.

> **Severity: Critical.** Complete authentication bypass requiring no valid credentials, achievable with a single crafted input string and no specialized tooling.

---

## 5. Full Attack Chain

```
[ Recon: Nmap ]
       │
       ▼
[ Apache 2.4.38 on port 80 — no known CVE ]
       │
       ▼
[ Web login form identified as sole application surface ]
       │
       ▼
[ Gobuster directory brute-force — no useful hidden paths ]
       │
       ▼
[ Default credential attempts — all failed ]
       │
       ▼
[ SQL Injection payload: admin'# ]
       │
       ▼
[ Query manipulated — password check commented out ]
       │
       ▼
[ Authenticated as admin — flag retrieved ]
```

**Lesson:** No password was ever known or cracked. A single unescaped quote and a comment character were enough to rewrite the application's intended logic entirely.

---

## 6. Remediation Summary

| # | Finding | Severity | Remediation |
|---|---------|----------|--------------|
| 1 | SQL query built via direct string concatenation of user input | **Critical** | Use **parameterized queries / prepared statements** (e.g., PDO with bound parameters in PHP) so user input is always treated as data, never as SQL syntax. |
| 2 | No input validation or sanitization on login fields | **High** | Validate and sanitize all user input server-side; reject or escape characters with special meaning (`'`, `"`, `#`, `--`) where they are not expected. |
| 3 | No apparent rate-limiting or lockout on the login form | **Medium** | Implement account lockout / exponential backoff after repeated failed login attempts, and log/alert on anomalous authentication patterns. |
| 4 | No Web Application Firewall (WAF) observed in front of the application | **Medium** | Deploy a WAF with SQL Injection detection rules as a defense-in-depth layer — not a substitute for fixing the underlying query, but a mitigating control. |
| 5 | Detailed internal error/query behavior potentially inferable from application responses | **Low** | Ensure generic error messages are returned to users; do not expose query structure, database errors, or stack traces that could aid further exploitation. |

---

## 7. Conclusion

This box is a compact illustration of a principle that remains at or near the top of the OWASP Top 10 year after year:

1. **User input must never be trusted to build a query directly.** The entire compromise stemmed from one unescaped string being concatenated into SQL.
2. **Enumeration discipline still matters even when the final vector is well-known.** Checking directories and default credentials first ensured no simpler path was missed before committing to the SQL Injection approach.

The fix is not exotic: parameterized queries have been standard practice for over a decade. The persistence of this vulnerability class is a reminder that "well-known" does not mean "eliminated."
