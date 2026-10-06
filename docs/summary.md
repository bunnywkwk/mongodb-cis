# Summary — what `mongodb8_cis` does for each CIS recommendation

CIS MongoDB 8 Benchmark v2.0.0, all 23 recommendations, in benchmark order. **Why** = the benchmark's Rationale, in
short. **Role** = what the role does. **Default** = what happens without any setting.

**Type:** A = Automated, M = Manual. **Role:** 🔧 PATCH (fixes it) · 🔧? DECISION (reports; fixes only if you set the
variable) · 📋 REPORT (a person decides) · ➖ not applicable on a standalone mongod.
**Default:** ✅ on · ⚠️ off by default because it can lock users out or break clients (turn it on with your values).

| ID | L | Type | Recommendation | Why (CIS rationale) | Role | Default | Your switch / value |
|----|---|------|----------------|---------------------|------|---------|---------------------|
| **1** | | | **Installation and Patching** | | | | |
| 1.1 | 1 | M | Appropriate version/patches installed | Current, supported, patched software limits known vulnerabilities | 📋 reports the installed version; never upgrades | ✅ | — |
| **2** | | | **Authentication** | | | | |
| 2.1 | 1 | A | Authentication is configured | Without authentication anyone can reach the data and actions can't be traced to a user | 🔧 creates the admin (`root@admin`), then `security.authorization: enabled` | ⚠️ | `mongodb8_cis_rule_2_1`, `_admin_user`, `_admin_password` |
| 2.2 | 1 | A | No auth bypass via the localhost exception | The exception lets a local connection in without a username/password | 🔧 checks a user exists, then `enableLocalhostAuthBypass: false` | ⚠️ | `mongodb8_cis_rule_2_2` |
| 2.3 | 2 | A | Authentication in the sharded cluster | Cluster members must prove who they are (key or certificate) | ➖ reports N/A on a standalone | off | `mongodb8_cis_rule_2_3` |
| **3** | | | **Authorization** | | | | |
| 3.1 | 1 | M | Least privilege for database accounts | Roles that can grant any privilege belong only to administrators | 🔧? lists accounts with `dbOwner`/`userAdmin`/`userAdminAnyDatabase` in admin; revokes them from accounts you list | ✅ report | `mongodb8_cis_revoke_admin_roles: ["admin.<user>"]` |
| 3.2 | 1 | M | Role-based access control enabled and configured | Access is controlled through roles, not individual grants | 📋 shows authorization state + every user's roles | ✅ | — ([manual-remediation.md](manual-remediation.md)) |
| 3.3 | 1 | M | Non-privileged, dedicated service account | A non-root account limits what a compromised database can reach on the OS | 📋 PASS if mongod doesn't run as root (RPM default: `mongod`) | ✅ | — |
| 3.4 | 1 | M | Each role is needed and grants only necessary privileges | Roles and privileges pile up over time and are no longer needed | 🔧? lists every custom role and its actions; drops the roles you list | ✅ report | `mongodb8_cis_drop_custom_roles: ["<db>.<role>"]` |
| 3.5 | 2 | M | Review superuser/admin roles | Fewer admin accounts, less chance of unwanted privileged access | 🔧? lists users with `root`, `*AnyDatabase`, `clusterAdmin`, …; revokes them from accounts you list (never the role's own admin) | ✅ report | `mongodb8_cis_revoke_superuser_roles: ["<db>.<user>"]` |
| **4** | | | **Data Encryption** (4.3 runs first: mongod refuses TLS options without TLS, D25) | | | | |
| 4.1 | 2 | A | Legacy TLS protocols disabled | TLS 1.0 is banned by PCI DSS; newer versions are stronger | 🔧 adds `TLS1_0,TLS1_1` to `disabledProtocols` (needs TLS on) | ✅ | — |
| 4.2 | 1 | A | Weak protocols disabled | TLS 1.0 is open to BEAST; TLS 1.1 has no authenticated encryption | 🔧 same setting as 4.1 (CIS lists it twice) | ✅ | — |
| 4.3 | 1 | A | Encryption of data in transit (TLS) | Stops sniffing of cleartext traffic and man-in-the-middle attacks | 🔧 checks the PEM files exist, then `net.tls.mode: requireTLS` + key + CA | ⚠️ | `mongodb8_cis_rule_4_3`, `_tls_certificate_key_file`, `_tls_ca_file` |
| 4.4 | 2 | A | FIPS is enabled | FIPS is the standard for how data is encrypted | 🔧 `net.tls.FIPSMode: true` (needs TLS) | ⚠️ | `mongodb8_cis_rule_4_4` |
| 4.5 | 2 | M | Encryption of data at rest | Stolen disks or backups can't be read without the master key | 📋 shows encryption + key management; never enables it ([why](manual-remediation.md#45-encryption-of-data-at-rest)) | ✅ | — |
| **5** | | | **Audit Logging** | | | | |
| 5.1 | 1 | A | System activity is audited | Logs are needed to troubleshoot and to investigate incidents | 🔧 adds `auditLog` if missing (syslog); never changes an existing one | ✅ | `mongodb8_cis_audit_log: {destination: syslog}` |
| 5.2 | 2 | M | Audit filters configured properly | The audit trail must record what the organisation needs to trace incidents | 🔧? shows the filter (none = everything); writes yours | ✅ report | `mongodb8_cis_audit_filter: '<filter>'` |
| 5.3 | 2 | A | Logging captures as much as possible | `quiet` hides details needed for troubleshooting and investigations | 🔧 writes `systemLog.quiet: false` | ✅ | — |
| 5.4 | 2 | A | New entries appended to the log | Overwriting the log can destroy old entries that are still needed | 🔧 `systemLog.logAppend: true` (fresh install already compliant) | ✅ | — |
| **6** | | | **Operating System Hardening** | | | | |
| 6.1 | 1 | A | Non-default port | Automated attacks and scanners look for standard ports | 🔧 checks your port, labels it for SELinux, sets `net.port` | ⚠️ | `mongodb8_cis_rule_6_1`, `mongodb8_cis_port` |
| 6.2 | 2 | M | OS resource limits set | Limits stop one process from consuming too many system resources | 🔧? compares systemd limits with CIS values; writes a drop-in | ✅ report | `mongodb8_cis_fix_resource_limits: true` (+ `_resource_limits`) |
| 6.3 | 2 | M | Server-side scripting disabled if not needed | Unneeded scripting adds risk from insecure code | 🔧? shows `javascriptEnabled`; disables it on your word | ✅ report | `mongodb8_cis_javascript_needed: false` |
| **7** | | | **File Permissions** | | | | |
| 7.1 | 1 | M | Appropriate key file permissions | Protected keys and certificates prevent unauthorized access | 🔧? PASS/FAIL per key/cert/CA file (0600, owner mongod); fixes them | ✅ report | `mongodb8_cis_fix_key_file_permissions: true` |
| 7.2 | 1 | M | Appropriate database file permissions | Restricts who can read the database files | 🔧? PASS/FAIL on dbPath (0770, owner mongod; RPM ships 0755); fixes it | ✅ report | `mongodb8_cis_fix_db_path_permissions: true` |

## Totals

| | Count | Rules |
|---|---|---|
| Recommendations | 23 | Level 1: 13 · Level 2: 10 |
| 🔧 Role fixes (Automated) | 10 | 2.1, 2.2, 4.1, 4.2, 4.3, 4.4, 5.1, 5.3, 5.4, 6.1 (5 of them ⚠️ off by default) |
| 🔧? Fixes on your decision (Manual) | 8 | 3.1, 3.4, 3.5, 5.2, 6.2, 6.3, 7.1, 7.2 |
| 📋 Report only (a person decides) | 4 | 1.1, 3.2, 3.3, 4.5 ([automation-decisions.md](automation-decisions.md)) |
| ➖ Not applicable (standalone) | 1 | 2.3 |

**Fully compliant run:** Level 2 on, the ⚠️ rules on with their values, the 🔧? variables set, and the 📋 items reviewed
by a person. Proof: [compliance-test.md](compliance-test.md). Example settings: `sysconfig/group_vars/mongodb.yml` in the
test project.

## Status (2026-10-06, branch `main`)

| Item | State |
|------|-------|
| All 23 rules | implemented; lint (production) and `--syntax-check` clean |
| Latest change | 3.4 drop list and 3.5 revoke list (with the admin guard): **accepted by the user** (D30) |
| VM-tested | **compliance test passed on RHEL 8, 9 and 10, Level 2** (2026-10-06; [test-results.md](test-results.md)). Not yet on a real mongod: the 3.1/3.4/3.5 revoke and drop lists with entries |
| Next | compliance test per VM ([compliance-test.md](compliance-test.md)), opt-in tests in [session-handoff.md](session-handoff.md) section 3 |

