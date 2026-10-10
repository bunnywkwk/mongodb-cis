# CIS requirements matrix — what we must deliver, and how we prove it

Our **target**, taken from the benchmark (CIS MongoDB 8 Benchmark v2.0.0: PDF + certification spreadsheet, columns
*Audit Procedure*, *Remediation Procedure*, *Default Value*). Every one of the 23 recommendations in the benchmark's
*Summary Table* has one row here. This is the checklist: a rule is **done** only when its row is built **and** its
"Verify" check gives the expected result on RHEL 8, 9 and 10.

How to read a row:
- **CIS checks / passes when:** what the benchmark's audit looks at, and the value that means "Set Correctly: Yes".
- **Role:**
  - **PATCH** fixes it when it's wrong.
  - **REPORT** only prints PASS / FAIL / REVIEW (Manual rules, or things the role must not decide).
  - **DECISION** reports, and fixes only if the site sets a decision variable (3.1, 5.2, 6.2, 6.3, 7.1, 7.2; D20).
  - "off" means the rule's toggle defaults to `false` (risky).
- **Verify:** run on the VM after the role, to confirm the result yourself (not trusting the role's own output).

## How to prove compliance

Run the role, rerun it (`changed=0`), then check every rule on the host with its CIS Audit command:
[compliance-test.md](compliance-test.md). How to pick a level and set site values: [README](../README.md).

## How each row was derived from the PDF

Every recommendation has the same parts; each one maps to one piece of the role.

| PDF part | Becomes |
|----------|---------|
| Title + (Automated / Manual) | Task name `"<ID> \| PATCH/AUDIT \| <title>"`. Automated → role may fix; Manual → report only |
| Profile Applicability | `when: mongodb8_cis_level_1/2` + tag `level1/2` |
| **Audit** | The AUDIT step: what is read and the **pass value** (the rule's `when:`) |
| **Remediation** | The PATCH steps, **in the same order** (e.g. 2.1: create admin user → `authorization: enabled` → restart) |
| Impact | Risky (lockout, broken clients) → toggle off by default + `# WARNING:` |
| Default Value | Expected result on a fresh install |
| Additional Information | Special cases: Enterprise-only, version notes |

Not implemented on purpose:
- Advice with no setting behind it (e.g. 2.1 Rationale's password complexity: MongoDB has no such check).
- Windows/Ubuntu commands (the same check is done on RHEL).
- Anything not in Audit/Remediation.

"Organisation defined" values become variables the user fills in.

---

## Section 1: Installation and Patching

| ID | Lvl / type | CIS checks → passes when | Role | Default | Verify on the VM |
|----|-----------|--------------------------|------|---------|------------------|
| 1.1 | L1 Manual | `mongod --version` / `db.version()` → the version is the latest patch the organisation can run (check mongodb.com/alerts) | REPORT: installed version + where to compare. Never upgrades | on | `mongod --version`; `dnf list --showduplicates mongodb-enterprise-server \| tail -1` (latest 8.0.x in the repo) |

## Section 2: Authentication

| ID | Lvl / type | CIS checks → passes when | Role | Default | Verify on the VM |
|----|-----------|--------------------------|------|---------|------------------|
| 2.1 | L1 Auto | `grep authorization /etc/mongod.conf` → `authorization: enabled`. Remediation: create the user administrator (CIS example: `role: "root"` on `admin`) **first**, then enable auth, then restart | PATCH: needs `admin_user`/`admin_password` → creates the admin user if missing → `security.authorization: enabled` | **off** | `grep -A2 '^security' /etc/mongod.conf`; `mongosh … --eval 'db.adminCommand({listDatabases:1})'` **without** login must fail |
| 2.2 | L1 Auto | `grep enableLocalhostAuthBypass` → `false` (default `true`) | PATCH: refuses if no DB user exists (lockout guard) → `setParameter.enableLocalhostAuthBypass: false` | **off** | `grep -A2 '^setParameter' /etc/mongod.conf` |
| 2.3 | L2 Auto | Sharded cluster: `clusterAuthMode: x509` (keyFile only for development) | REPORT: values, or "not applicable" on standalone; cluster members: PATCH x509 (D31) | off | `grep -E 'clusterAuthMode\|keyFile' /etc/mongod.conf` |

## Section 3: Authorization (all Manual: a person decides who needs what)

| ID | Lvl / type | CIS checks → passes when | Role | Default | Verify on the VM |
|----|-----------|--------------------------|------|---------|------------------|
| 3.1 | L1 Manual | `db.system.users.find` with `dbOwner`, `userAdmin`, `userAdminAnyDatabase` in `admin` → none (CIS fix: drop them) | DECISION: lists them (PASS if none); revokes the 3 roles from accounts named in `mongodb8_cis_revoke_admin_roles` | on | `mongosh … --eval 'db.getSiblingDB("admin").system.users.find({},{user:1,roles:1})'` |
| 3.2 | L1 Manual | `db.getUser()` / `getRole()` → each user has only appropriate roles | REPORT: authorization state + every user with roles; optional `mongodb8_cis_users` (D31) | on | same as above |
| 3.3 | L1 Manual | `ps -ef \| grep mongod` → runs as a dedicated non-root user | REPORT: unit `User=` and process owner, PASS if not root | on | `ps -o user= -C mongod` → `mongod` |
| 3.4 | L1 Manual | `rolesInfo … showPrivileges` → only needed roles/privileges | REPORT: every user-defined role and its actions | on | `mongosh … --eval 'db.getSiblingDB("admin").getRoles({showPrivileges:true})'` |
| 3.5 | L2 Manual | `rolesInfo` for superuser/admin roles → only people who need them hold them | REPORT: users with `root`, `dbOwner`, `userAdmin*`, `*AnyDatabase`, `clusterAdmin`, `hostManager` | on | same as 3.1. The 2.1 admin appears here, as expected: CIS's own 2.1 remediation creates it with `role: "root"` |

## Section 4: Data Encryption

| ID | Lvl / type | CIS checks → passes when | Role | Default | Verify on the VM |
|----|-----------|--------------------------|------|---------|------------------|
| 4.3 | L1 Auto | `net.tls.mode` → `requireTLS`, with `certificateKeyFile` and `CAFile` | PATCH: checks both PEM files exist → sets mode + files. Runs **first** in Section 4 (mongod refuses TLS options without TLS) | **off** | `grep -A6 'tls:' /etc/mongod.conf`; plain `mongosh --port <p>` must fail |
| 4.1 | L2 Auto | `disabledProtocols` → includes `TLS1_0,TLS1_1` | PATCH: adds them (keeps others). TLS off → prints FAIL | on (L2) | `grep disabledProtocols /etc/mongod.conf` |
| 4.2 | L1 Auto | Same key as 4.1 (CIS lists it twice, L1 and L2) | PATCH: same as 4.1 | on | same as 4.1 |
| 4.4 | L2 Auto | `net.tls.FIPSMode: true`; the log shows FIPS mode activated | PATCH: sets it (needs TLS on) | **off** | `grep FIPSMode /etc/mongod.conf`; `grep -i 'fips' /var/log/mongodb/mongod.log \| tail -2` |
| 4.5 | L2 Manual | `security.enableEncryption: true` + key file / KMIP (CIS recommends KMIP) | REPORT: on/off + key management; enabling it by hand (D34) ([manual-remediation.md](manual-remediation.md) 4.5) | on (L2) | `grep -A4 '^security' /etc/mongod.conf` |

## Section 5: Audit Logging

| ID | Lvl / type | CIS checks → passes when | Role | Default | Verify on the VM |
|----|-----------|--------------------------|------|---------|------------------|
| 5.1 | L1 Auto | `auditLog.destination` is set (syslog, console or file) | PATCH: adds `auditLog` if missing (`syslog` by default; site can pick `file` + JSON/BSON). Never replaces an existing one | on | `grep -A3 '^auditLog' /etc/mongod.conf`; `journalctl -t mongod -n 5` (syslog) |
| 5.2 | L2 Manual | `auditLog.filter` matches the organisation's requirements | DECISION: shows the filter (none = everything is audited); writes `mongodb8_cis_audit_filter` if set | on (L2) | `grep -A6 '^auditLog' /etc/mongod.conf` |
| 5.3 | L2 Auto | `systemLog.quiet` → `false` | PATCH: writes `quiet: false` unless already explicitly `false` (fresh install: added) | on (L2) | `grep quiet /etc/mongod.conf` → `quiet: false` |
| 5.4 | L2 Auto | `systemLog.logAppend` → `true` | PATCH: sets `true` if not (fresh install: already compliant) | on (L2) | `grep logAppend /etc/mongod.conf` |

## Section 6: Operating System Hardening

| ID | Lvl / type | CIS checks → passes when | Role | Default | Verify on the VM |
|----|-----------|--------------------------|------|---------|------------------|
| 6.1 | L1 Auto | `net.port` → not `27017` (value is "organisation defined") | PATCH: checks `mongodb8_cis_port` (1024–65535, not 27017) → SELinux label `mongod_port_t` → sets `net.port` | **off** | `grep port /etc/mongod.conf`; `ss -tlnp \| grep mongod`; `semanage port -l \| grep mongod_port_t` |
| 6.2 | L2 Manual | `/proc/<pid>/limits` → f, t, v, m unlimited; n, u 64000 | DECISION: the six limits from systemd vs `mongodb8_cis_resource_limits` (CIS values); writes a systemd drop-in with `mongodb8_cis_fix_resource_limits: true` | on (L2) | `cat /proc/$(pidof mongod)/limits`; `systemctl cat mongod` (drop-in) |
| 6.3 | L2 Manual | `security.javascriptEnabled` → `false` **if not needed** | DECISION: report; fix only with `mongodb8_cis_javascript_needed: false` | on (L2) | `grep javascriptEnabled /etc/mongod.conf` |

## Section 7: File Permissions

| ID | Lvl / type | CIS checks → passes when | Role | Default | Verify on the VM |
|----|-----------|--------------------------|------|---------|------------------|
| 7.1 | L1 Manual | `ls -l` of `keyFile`, PEM key, `CAFile` → key files `600`, owned by the mongo user | DECISION: mode/owner of each, or "not applicable"; sets `0600` + owner with `mongodb8_cis_fix_key_file_permissions: true` | on | `ls -l /etc/pki/mongodb/` |
| 7.2 | L1 Manual | `stat -c %a <dbPath>` → `770`, owner/group = mongo user | DECISION: PASS/FAIL (RPM ships `755` → FAIL); fix with `mongodb8_cis_fix_db_path_permissions: true` | on | `stat -c '%a %U:%G' /var/lib/mongo` |

---

## Coverage summary

| | Count | Rules |
|---|---|---|
| Recommendations in the benchmark | **23** | 13 Level 1, 10 Level 2 |
| Automated, role fixes them (PATCH) | 11 | 2.1, 2.2, 2.3 (cluster members), 4.1, 4.2, 4.3, 4.4, 5.1, 5.3, 5.4, 6.1 |
| Manual, role reports | 3 | 1.1, 3.3, 4.5 (why: D29; hand fixes: [manual-remediation.md](manual-remediation.md)) |
| Manual, report + optional site decision | 9 | 3.1, 3.2, 3.4, 3.5, 5.2, 6.2, 6.3, 7.1, 7.2 |
| Off by default (risky) | 6 | 2.1, 2.2, 2.3, 4.3, 4.4, 6.1 |
| **Not covered** | **0** | |

**What the role does NOT do, on purpose** (and why it's still CIS-correct):
- **Fixing Manual rules without a site decision** (changing roles, encryption at rest): CIS says a person decides. The role gives that person the facts; hand fixes are in [manual-remediation.md](manual-remediation.md), the reasoning in D20/D28.
- **Creating certificates, choosing the port, inventing passwords:** CIS says "organisation defined". The role only uses the values you give.
- **Upgrading MongoDB:** 1.1 is Manual, and a hardening run must never upgrade a database by surprise.
