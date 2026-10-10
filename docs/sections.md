# What each section does — quick reference (`main`)

One page per CIS section: the goal, what each rule does, what you can set, and what the **variables inside the
task files** mean (the `vars:` at the top of a rule). How `mongod.conf` values are read and written:
[reading-config.md](reading-config.md). Why things are built this way: [design-decisions.md](design-decisions.md) (`Dn`).

Legend for "Fresh install": result on a new MongoDB 8.0 install with the role's defaults.

## Shared values every section uses (set in `prelim.yml` and `vars/main.yml`)

| Variable | What it holds | Where it comes from |
|----------|---------------|---------------------|
| `discovered_mongod_conf` | `mongod.conf` as a dictionary; updated after every config write (D7) | prelim: `slurp` + `from_yaml` |
| `discovered_mongod_running_conf` | the config mongod is **running** with; never updated in the run (D24) | prelim: copy of the above |
| `discovered_mongod_service` | systemd properties of `mongod` (`User`, `MainPID`, `Limit*`) | prelim: `systemd_service` (read-only) |
| `discovered_mongodb_version` | installed server version, e.g. `8.0.32` | prelim: `package_facts` |
| `mongodb8_cis_service_user` | the user mongod runs as (`User=`), fallback `mongod` | `vars/main.yml` (D8) |
| `mongodb8_cis_tls_enabled` | `true` when `net.tls.mode` is anything but `disabled` | `vars/main.yml` |
| `mongodb8_cis_shell_host` / `_port` / `_auth` / `_tls` | how the role's `mongosh` reaches mongod (from the **running** config) | `vars/main.yml` (D24) |
| `mongodb8_cis_conf_path` | `/etc/mongod.conf` | `vars/main.yml` |

Every rule's `when:` starts with its toggle and its level (`mongodb8_cis_rule_<id>`, `mongodb8_cis_level_1/2`).

---

## Section 1 — Installation and Patching

**Goal:** run a supported, patched MongoDB release.

| Rule | Type | Role does | You can set | Fresh install |
|------|------|-----------|-------------|---------------|
| **1.1** (L1) | Manual, report | Prints the installed version and where to check for patches (mongodb.com/alerts). Never upgrades | — | 👤 REVIEW |

- Why no upgrade: an upgrade restarts the database and needs a change window (D29).

---

## Section 2 — Authentication

**Goal:** nobody reaches the data without logging in.

| Rule | Type | Role does | You can set | Fresh install |
|------|------|-----------|-------------|---------------|
| **2.1** (L1) | Automated, **off** | Creates the admin (`root@admin`, as in CIS's remediation) **first**, then `security.authorization: enabled`, restart | `mongodb8_cis_rule_2_1: true`, `mongodb8_cis_admin_user`, `mongodb8_cis_admin_password` (Vault) | ❌ auth disabled |
| **2.2** (L1) | Automated, **off** | Checks at least one user exists (lockout guard), then `setParameter.enableLocalhostAuthBypass: false`, restart | `mongodb8_cis_rule_2_2: true` | ❌ bypass on (default `true`) |
| **2.3** (L2) | Automated, PATCH, off | `NOT APPLICABLE` on a standalone; on a replica set / shard / config server member: `security.clusterAuthMode: x509` + `net.tls.clusterFile` (needs 4.3) | `mongodb8_cis_rule_2_3`, `mongodb8_cis_cluster_file` | ➖ / 🔧 |

**Variables in the code**

| Rule | Variable / check | Meaning |
|------|------------------|---------|
| 2.1 | `mongodb8_cis_2_1_settings` | the keys to merge: `{security: {authorization: enabled}}` |
| 2.1 | `when: ...['security']['authorization'] \| default('disabled') != 'enabled'` | run only while auth is off |
| 2.1 | `discovered_2_1_admin_user` | `mongosh` answer to `getUser(<admin>) !== null`: the text `true`/`false`; the user is created only when not `true` |
| 2.2 | `mongodb8_cis_2_2_settings` | `{setParameter: {enableLocalhostAuthBypass: false}}` |
| 2.2 | `when: ... \| default(true) \| bool` | missing = MongoDB default `true`; `\| bool` because the value comes from a human-written file (D19) |
| 2.2 | `discovered_2_2_users` | number of users in `admin.system.users` (all users live there) |

- Order matters: create the user **before** turning on auth (CIS 2.1); 2.2 refuses to run with zero users (D26).
- `no_log: true` hides the password in the user-creation task.

---

## Section 3 — Authorization

**Goal:** each account has only the roles it needs. All five are **Manual**: a person decides who needs what.

| Rule | Type | Role does | You can set | Fresh install |
|------|------|-----------|-------------|---------------|
| **3.1** (L1) | Manual + optional fix | Runs CIS's own query: accounts with `dbOwner`, `userAdmin` or `userAdminAnyDatabase` **in admin**. PASS if none; revokes those roles from accounts you list | `mongodb8_cis_revoke_admin_roles: ["admin.badadmin"]` | ✅ PASS (no users) |
| **3.2** (L1) | Manual, report + site decision | Authorization state + every user with its roles; `mongodb8_cis_users` creates missing accounts and adds missing roles (never removes) | `mongodb8_cis_users` | 👤 REVIEW |
| **3.3** (L1) | Manual, report | PASS if mongod's unit `User=` and the running process are not root | — | ✅ PASS (`mongod`) |
| **3.4** (L1) | Manual, report + opt-in | Every **user-defined** role and its actions (built-in roles are fixed by MongoDB); drops the roles in `mongodb8_cis_drop_custom_roles` | — | 👤 0 roles |
| **3.5** (L2) | Manual, report + opt-in | Users holding a superuser/admin role; revokes them from accounts in `mongodb8_cis_revoke_superuser_roles` (never the role's own admin) | — | 👤 REVIEW |

**Variables in the code**

| Rule | Variable | Meaning |
|------|----------|---------|
| 3.1 | `mongodb8_cis_3_1_roles` | the three roles CIS names: `[dbOwner, userAdmin, userAdminAnyDatabase]` |
| 3.1 | `discovered_3_1_users` | result of CIS's query on `admin.system.users`; each item has `_id` (`<db>.<user>`) and `roles` |
| 3.1 | revoke `when:` | the account is in your list **and** still holds one of the three roles on `admin` → rerun skips it |
| 3.2 | `discovered_3_2_users` | every user (`user`, `db`, `roles`) |
| 3.3 | `mongodb8_cis_3_3_unit_user` | `User=` from systemd; empty means systemd runs it as **root** |
| 3.3 | `mongodb8_cis_3_3_pid` / `_owner` | mongod's process ID, and the owner of `/proc/<PID>` (= who it really runs as) |
| 3.4 | `discovered_3_4_roles` | documents of `admin.system.roles` (where MongoDB stores all custom roles) |
| 3.5 | `mongodb8_cis_3_5_roles` | `root` + the roles CIS lists: `dbOwner`, `userAdmin`, `userAdminAnyDatabase`, `readWriteAnyDatabase`, `dbAdminAnyDatabase`, `clusterAdmin`, `hostManager` |

- The 2.1 admin shows up in 3.2 and 3.5 with `root`: expected, CIS's own 2.1 remediation creates it that way.
- `main` reads users with `mongodb_shell` + CIS's queries on `admin.system.users`, so **every** user is seen (D13).
- 3.4/3.5 options: D30; 3.2 option: D31 (only adds; an empty list changes nothing). Why 3.3 has no fix: [design-decisions.md](design-decisions.md) D28/D29; by hand: [manual-remediation.md](manual-remediation.md).

---

## Section 4 — Data Encryption

**Goal:** data is encrypted on the network (TLS) and, ideally, on disk.

| Rule | Type | Role does | You can set | Fresh install |
|------|------|-----------|-------------|---------------|
| **4.3** (L1) | Automated, on | Copies the two files from `_src` to `/etc/pki/mongodb/`, keeping their names (owner mongod, 0600), then `net.tls.mode: requireTLS` + `certificateKeyFile` + `CAFile`; `NOT APPLIED` until both `_src` are set. **Runs first** in Section 4 (D25) | `mongodb8_cis_tls_certificate_key_src`, `mongodb8_cis_tls_ca_src` | ❌ TLS off |
| **4.1** (L2) | Automated | Adds `TLS1_0,TLS1_1` to `net.tls.disabledProtocols` (keeps others). TLS off → FAIL message, no write | — | ❌ FAIL (TLS off) |
| **4.2** (L1) | Automated | Same key as 4.1 (CIS lists it at both levels) | — | ❌ FAIL (TLS off) |
| **4.4** (L2) | Automated, **off** | `net.tls.FIPSMode: true` (needs TLS) | `mongodb8_cis_rule_4_4: true` | ❌ |
| **4.5** (L2) | Manual, report | PASS if `security.enableEncryption`; shows KMIP or keyfile. Enabling it is done by hand (D34) | — | 👤 REVIEW (off) |

**Variables in the code**

| Rule | Variable | Meaning |
|------|----------|---------|
| 4.3 | `mongodb8_cis_4_3_key_file` / `_ca_file` | your variable if set, else the value already in `mongod.conf` |
| 4.3 | `discovered_4_3_files` | `stat` of both files; the assert stops if either is missing |
| 4.1/4.2 | `mongodb8_cis_4_1_disabled` / `_4_2_disabled` | `disabledProtocols` as a list: split on `,`, trim spaces, drop empty items. `"TLS1_0, TLS1_1"` → `[TLS1_0, TLS1_1]` |
| 4.1/4.2 | `when: not (['TLS1_0','TLS1_1'] is subset(...))` | run only if one of the two is missing |
| 4.4 | `mongodb8_cis_4_4_settings` | `{net: {tls: {FIPSMode: true}}}` |

- mongod refuses TLS options without a TLS mode, so 4.1/4.2/4.4 only write when `mongodb8_cis_tls_enabled` (D25).
- 4.5 is report only (D34): existing data can't be encrypted in place, a lost key = unreadable data (D29).

---

## Section 5 — Audit Logging

**Goal:** MongoDB records **what happens** and **who did what**.

| Log | Records | Where (role default) | Read it with |
|-----|---------|----------------------|--------------|
| 📓 Normal log (`systemLog`) | startup, errors, connections, slow queries | `/var/log/mongodb/mongod.log` | `sudo tail -f /var/log/mongodb/mongod.log` |
| 🔒 Audit log (`auditLog`, Enterprise) | logins, user/role changes, dropped collections, failed access | syslog (journal) | `sudo journalctl -t mongod -f` |

| Rule | Type | Role does | You can set | Fresh install |
|------|------|-----------|-------------|---------------|
| **5.1** (L1) | Automated | Adds `auditLog` **only if missing** (an existing one is never changed) | `mongodb8_cis_audit_log` (default `{destination: syslog}`; for a file add `format` + `path`) | ✅ added (changed) |
| **5.2** (L2) | Manual + optional fix | Shows the filter (none = everything audited); writes yours if set and auditing is on | `mongodb8_cis_audit_filter: '{ atype: { $in: [ "authenticate" ] } }'` | 👤 no filter |
| **5.3** (L2) | Automated | Writes `systemLog.quiet: false` unless it is already explicitly `false`, so CIS's `grep quiet` finds it | — | ✅ added (changed) |
| **5.4** (L2) | Automated | `systemLog.logAppend: true` if not set | — | ✅ PASS (RPM sets it) |

**Variables in the code**

| Rule | Variable | Meaning |
|------|----------|---------|
| 5.1 | `when: ...['auditLog']['destination'] \| default('') \| length == 0` | run only when there is **no** audit destination yet |
| 5.1 | `mongodb8_cis_audit_log` (defaults) | the `auditLog:` block, written as-is. CIS's four options: `{destination: syslog}` (default), `{destination: console}`, `{destination: file, format: JSON, path: /var/log/mongodb/auditLog.json}`, same with `BSON` |
| 5.1 | `mongodb8_cis_5_1_settings` | `{auditLog: <your block>}`, merged into `mongod.conf` like 5.3/5.4 |
| 5.1 | assert | stops on a typo: destination not `syslog`/`console`/`file`, or `file` without `format` (`JSON`/`BSON`) and `path` |
| 5.2 | `when:` of the PATCH | a filter is set **and** auditing is on **and** the current filter differs |
| 5.3 | `mongodb8_cis_5_3_settings` + `when: ... \| default(true) is not false` | `{systemLog: {quiet: false}}`; missing counts as "not written yet", so a fresh install gets the explicit line (D10) |
| 5.4 | `mongodb8_cis_5_4_settings` + `when: ... is not true` | `{systemLog: {logAppend: true}}` |

- `syslog` by default: the OS rotates and can forward it; a file isn't rotated by mongod (D21).
- A filter only records **fewer** events, never more.

---

## Section 6 — Operating System Hardening

**Goal:** limit what an attacker can find and what mongod can use.

| Rule | Type | Role does | You can set | Fresh install |
|------|------|-----------|-------------|---------------|
| **6.1** (L1) | Automated, **off** | Checks the port (1024–65535, not 27017), labels it `mongod_port_t` when SELinux is on, sets `net.port`, restart | `mongodb8_cis_rule_6_1: true`, `mongodb8_cis_port: 27100` | ❌ 27017 |
| **6.2** (L2) | Manual + optional fix | PASS/REVIEW per limit vs CIS; writes a systemd drop-in if they differ | `mongodb8_cis_fix_resource_limits: true`, `mongodb8_cis_resource_limits` | ✅ PASS (RPM unit) |
| **6.3** (L2) | Manual + optional fix | Shows `javascriptEnabled`; disables server-side JS if you say it isn't needed | `mongodb8_cis_javascript_needed: false` | 👤 on (default `true`) |

**Variables in the code**

| Rule | Variable | Meaning |
|------|----------|---------|
| 6.1 | `mongodb8_cis_port \| int` | `\| int` turns text into a number, so `"27017"` can't pass as "not 27017" ([reading-config.md](reading-config.md) 7) |
| 6.1 | `mongodb8_cis_selinux_default_ports` (`vars/`) | 27017–27019, 28017–28019: already `mongod_port_t`, no label needed |
| 6.2 | `mongodb8_cis_resource_limits` (defaults) | CIS values: f/t/v/m = `infinity`, n/u = `64000`, as systemd names (`LimitFSIZE`, `LimitCPU`, `LimitAS`, `LimitNOFILE`, `LimitRSS`, `LimitNPROC`) |
| 6.2 | `mongodb8_cis_6_2_drift` | `true` when what systemd applies now differs from that dict |
| 6.2 | `mongodb8_cis_limits_dropin` (`vars/`) | `/etc/systemd/system/mongod.service.d/mongodb8_cis-limits.conf`; the RPM's unit is never edited |
| 6.3 | `mongodb8_cis_6_3_settings` | `{security: {javascriptEnabled: false}}` |
| 6.3 | `when: not mongodb8_cis_javascript_needed` + `... \| default(true) is not false` | the site doesn't need JS **and** it isn't off yet (missing = MongoDB default `true`) |

- On RHEL 8/9 mongod is confined (`mongod_t`) and may only bind `mongod_port_t` ports; on RHEL 10 it is unconfined
  ([test-results.md](test-results.md), D11). 6.1 labels a new port `mongod_port_t` when SELinux is enabled.
- The restart handler runs `daemon_reload` so systemd reads the 6.2 drop-in.
- Server-side JavaScript (`$where`, `mapReduce`, `$function`, `$accumulator`) is deprecated in MongoDB 8.0.

---

## Section 7 — File Permissions

**Goal:** only the mongod account can read the **secrets** (key/certificate files) and the **data** (dbPath).

| Rule | Type | Role does | You can set | Fresh install |
|------|------|-----------|-------------|---------------|
| **7.1** (L1) | Manual + optional fix | PASS/FAIL per `keyFile`, TLS key and CA file (`0600`/`0400`, owner **and** group = service user), or NOT APPLICABLE | `mongodb8_cis_fix_key_file_permissions: true` | ➖ no files yet |
| **7.2** (L1) | Manual + optional fix | PASS/FAIL for the dbPath directory (`0770`, owner/group = service user) | `mongodb8_cis_fix_db_path_permissions: true` | ❌ FAIL (RPM: `0755`) |

**Variables in the code**

| Rule | Variable | Meaning |
|------|----------|---------|
| 7.1 | `mongodb8_cis_7_1_files` | `keyFile`, `certificateKeyFile`, `CAFile` from `mongod.conf`; `select` drops the empty ones |
| 7.1 | `discovered_7_1_stat` | `stat` of each file: `mode`, `pw_name` (owner), `gr_name` (group), `exists` |
| 7.2 | `mongodb8_cis_7_2_db_path` | `storage.dbPath`, or the RPM's `/var/lib/mongo` if not set (D8) |
| 7.2 | `discovered_db_path` | `stat` of that directory |

- CIS's commands use Ubuntu's user (`mongodb`) and path (`/var/lib/mongodb`); RHEL uses `mongod` and `/var/lib/mongo`.
- 7.1 never creates a missing file; 7.2 changes the directory only (no `-R`). No restart needed.
- 7.1 becomes real once 4.3 adds TLS files.
