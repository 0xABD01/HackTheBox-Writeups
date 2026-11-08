# Write-Up — Redeemer: Unauthenticated Redis Access to Flag Retrieval

> **Type:** Network / Service Penetration Test (HTB Starting Point)
> **Target:** `10.129.79.54` (`Redeemer`)
> **Tools:** Nmap, redis-cli
> **Final Result:** Unauthenticated database access via a misconfigured, no-auth Redis instance

---

## 1. Introduction

The target is a beginner-tier lab machine from Hack The Box's Starting Point track, running a single exposed **Redis** in-memory database service. The objective was to enumerate the host, connect to the database without credentials, and retrieve a flag stored as a key-value pair.

This box illustrates one of the most common NoSQL/cache-layer findings:

- **No authentication configured** on a database service (`requirepass` left unset), allowing any network client to connect and issue full read/write commands.
- **Sensitive data left directly retrievable** via a simple `keys *` / `get` enumeration, requiring no exploit code at all.

---

## 2. Reconnaissance

### 2.1 Connectivity Check

```bash
ping 10.129.79.54
```

A couple of successful replies were enough to confirm the VPN tunnel was up and the target reachable — the ping was interrupted early since connection quality, not a full round-trip log, was the only thing needed at this stage.

### 2.2 Port Scan — Nmap

```bash
nmap -sV 10.129.79.54
```

Result:

| Port | Service | Version |
|------|---------|---------|
| 6379/tcp | Redis | [version as reported by scan] |

The `-sV` switch was used to confirm the service identity behind the port. Only a single port was found open, making Redis the sole path forward — and immediately notable, since Redis has **no authentication enabled by default** unless explicitly configured with a password.

---

## 3. Vulnerability Analysis

### 3.1 Identifying the Flaw

Rather than searching for a Redis version-specific CVE, the service was tested for the most common Redis-specific misconfiguration: **no `requirepass` set**, meaning any client that can reach port 6379 has full, unauthenticated access to the database — including the ability to read all stored keys.

### 3.2 The Vulnerability

- **CVE:** N/A — this is a configuration weakness, not a software bug
- **Type:** Missing authentication on a network-exposed database service
- **Authentication:** None required
- **Impact:** Full read (and typically write) access to all key-value data stored on the server

---

## 4. Initial Access

### 4.1 Installing the Client

```bash
sudo apt install redis-tools
```

Confirmed with `redis-cli --help` that the utility was installed and available.

### 4.2 Connecting to the Service

```bash
redis-cli -h 10.129.79.54
```

The connection succeeded immediately, dropping into an interactive Redis prompt with no password requested — confirming the server has no authentication configured.

### 4.3 Server Enumeration

```bash
info
```

Under the **Keyspace** section, a single logical database was found:

```
db0:keys=[N],expires=[N],avg_ttl=[N]
```

confirming database index `0` was the one to explore.

### 4.4 Selecting the Database and Listing Keys

```bash
select 0
keys *
```

This returned the list of all keys stored in the database, including one relevant to the lab objective.

### 4.5 Flag / Evidence

```bash
get [flag_key_name]
```

This returned the lab's hash-value flag directly from the in-memory store, submitted to the Starting Point page to mark the machine as complete.

> **Severity: Critical.** Full unauthenticated access to a database service, with no distinction between read and write operations — an attacker in this position could also modify or delete data, or in more complex configurations leverage Redis for further exploitation (e.g., writing SSH keys or cron entries via `CONFIG SET`/`SAVE` abuse).

---

## 5. Full Attack Chain

```
[ Recon: Nmap ]
       │
       ▼
[ Only Redis (6379) open ]
       │
       ▼
[ redis-cli -h [TARGET_IP] → connects with no authentication ]
       │
       ▼
[ info → confirms single database (index 0) ]
       │
       ▼
[ select 0 → keys * → flag key identified ]
       │
       ▼
[ get [key] → flag retrieved ]
```

**Lesson:** No exploit was written or run. The entire compromise consisted of connecting to a database that was never configured to ask for a password in the first place.

---

## 6. Remediation Summary

| # | Finding | Severity | Remediation |
|---|---------|----------|--------------|
| 1 | Redis instance with no authentication (`requirepass` unset) | **Critical** | Set a strong password via `requirepass` in `redis.conf`, or better, use Redis ACLs (Redis 6+) to define least-privilege users per application. |
| 2 | Redis exposed directly to the network rather than bound to localhost | **High** | Bind Redis to `127.0.0.1` or an internal-only interface (`bind` directive) unless remote access is explicitly required; if required, restrict via firewall rules to known application servers only. |
| 3 | No TLS on the Redis connection | **Medium** | Enable Redis's built-in TLS support (Redis 6+) or place the service behind a `stunnel`/VPN tunnel if remote connectivity is unavoidable. |
| 4 | Potentially dangerous commands (`CONFIG`, `FLUSHALL`, `SAVE`) left enabled and reachable | **Medium** | Use `rename-command` in `redis.conf` to disable or rename high-risk administrative commands not needed by the application. |

---

## 7. Conclusion

This box is a compact illustration of a principle that applies broadly to modern application stacks:

1. **"Internal-only" services still need authentication.** Redis is often deployed as a supporting cache layer and assumed to be safely tucked away — but if the network path is reachable, an unauthenticated instance is a fully open database.
2. **Default configurations are rarely secure by default.** Redis does not require a password out of the box; it is an opt-in control that must be deliberately configured.

The fix, as with most findings of this class, is a one-line configuration change (`requirepass`) combined with basic network segmentation — no patch, no code fix, just closing a door that was never locked.
