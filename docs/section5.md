# Section 5 — Audit Logging

**Goal:** MongoDB keeps a record of **what happens** and **who did what**, so problems and security incidents can be traced.

## Two logs, two rules each

| Log | What it records | Where (role default) | Read it with |
|-----|-----------------|----------------------|--------------|
| 📓 **Normal log** (`systemLog`) | How the server runs: startup, errors, connections, slow queries | File `/var/log/mongodb/mongod.log` | `sudo tail -f /var/log/mongodb/mongod.log` |
| 🔒 **Audit log** (`auditLog`, Enterprise) | Who did what: logins, user/role changes, dropped collections, failed access | syslog (system journal) | `sudo journalctl -t mongod -f` |

| Rule | Log | Goal | Role does | You can set |
|------|-----|------|-----------|-------------|
| **5.1** (L1) | 🔒 audit | **Turn it on** | Adds `auditLog: {destination: syslog}` if missing | `mongodb8_cis_audit_destination`: `syslog` / `console` / `file` (+ `_format`, `_path`) |
| **5.2** (L2, Manual) | 🔒 audit | Keep only the events the organisation needs | Reports the filter; writes it if you set one | `mongodb8_cis_audit_filter`, e.g. `'{ atype: { $in: [ "authenticate", "createUser" ] } }'` |
| **5.3** (L2) | 📓 normal | **Don't hide details** | Sets `systemLog.quiet: false` if someone set it `true` | — |
| **5.4** (L2) | 📓 normal | **Keep old lines** after a restart | Sets `systemLog.logAppend: true` if missing | — |

## Good to know

- **No filter = everything auditable is recorded.** A filter only keeps **fewer** events, never more.
- Not audited by default: successful normal reads/writes (MongoDB's `auditAuthorizationSuccess`; not a CIS requirement).
- `quiet: true` would drop connection, authentication and replication messages from `mongod.log`. That's why CIS wants `false`.
- **Fresh install:** 5.3 and 5.4 already pass (MongoDB's shipped config); 5.1 adds the audit log; 5.2 only reports.
- Each change rewrites `/etc/mongod.conf` (old copy kept as `mongod.conf.<…>~`) and restarts mongod once.

Source: CIS MongoDB 8 Benchmark v2.0.0, recommendations 5.1–5.4.
