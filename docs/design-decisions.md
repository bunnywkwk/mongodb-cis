# Design decisions — `mongodb8_cis` (rebuild, branch `mongodb8-cis-rebuild`)

Why the role is built the way it is. Every decision has **Decision**, **Why** and **Evidence**. This file is new for
the rebuild (started 2026-10-03). The previous version on `main` is the history; IDs (D1, D2, ...) are **kept the same**
so that comments in `vars/main.yml`, `build-guide.md` and `CLAUDE.md` still point to the right entry. Where the
rebuild differs from `main`, the entry says **Changed in the rebuild**.

Scope: **MongoDB Enterprise 8.0, standalone `mongod`, RHEL 8, 9 and 10 (x86_64)**, CIS MongoDB 8 Benchmark v2.0.0.

Related: [reading-config.md](reading-config.md) (how config values are read and written),
[simplicity-review.md](simplicity-review.md) (how each line is justified).

## Sources

| # | Source |
|---|--------|
| S1 | MongoDB docs, *Install MongoDB Enterprise Edition on Red Hat or CentOS* (v8.0): https://www.mongodb.com/docs/v8.0/tutorial/install-mongodb-enterprise-on-red-hat/ |
| S2 | Enterprise repo metadata `https://repo.mongodb.com/yum/redhat/<8\|9\|10>/mongodb-enterprise/8.0/<arch>/repodata/` and keys at `https://pgp.mongodb.com/` |
| S3 | `mongodb-enterprise-server-8.0.32-1.el8/.el9/.el10` RPMs from S2, inspected with `rpm -qp --scripts`, `rpm -qplv`, `rpm2cpio` (2026-10-01) |
| S4 | MongoDB v8.0 docs: auditing, FIPS, encryption at rest, parameters, configuration options (`https://www.mongodb.com/docs/v8.0/...`) |
| S5 | CIS MongoDB 8 Benchmark v2.0.0 PDF (`cis-pdf/`, local only, never committed) |
| S6 | CIS MongoDB 8 Benchmark v2.0.0 certification spreadsheet (`cis-spreadsheet/`, local only) |
| S7 | ansible-core support matrix: https://docs.ansible.com/ansible/latest/reference_appendices/release_and_maintenance.html |
| S8 | EPEL repo metadata (pymongo versions per RHEL) |
| S9 | MongoDB SELinux policy: https://github.com/mongodb/mongodb-selinux |
| R1 | RFC 9293 (TCP), section 3.1: source/destination port fields are 16 bits |
| R2 | RFC 6335, section 6: port ranges — System Ports 0–1023, User Ports 1024–49151, Dynamic 49152–65535 |
| R3 | Linux kernel `net.ipv4.ip_unprivileged_port_start` (default `1024`; `sysctl` on the AlmaLinux 10 workstation, 2026-10-03) |
| L1 | ansible-lockdown `RHEL9-CIS` `tasks/section_5/cis_5.4.2.x.yml` rule 5.4.2.2: AUDIT then PATCH in one block |
| L2 | ansible-lockdown `RHEL9-CIS` `tasks/section_1/cis_1.1.1.x.yml` rule 1.1.1.1: declarative module, no AUDIT step |
| L3 | ansible-lockdown `RHEL9-CIS` `defaults/main/main.yml` (`disruption_high`, section and level switches) |
| L4 | ansible-lockdown `RHEL9-CIS`: level variables are used only by the Goss audit templates, levels are chosen by tags |

`platform-notes.md` (per-OS facts with commands) is not restored on this branch yet; the S-sources above are defined
here until it is.

---

## D0. The rebuild: retyped by hand, batch by batch (new)

- **Decision:** branch `mongodb8-cis-rebuild` rebuilds the role from `docs/build-guide.md`. The user types `tasks/`
  and `handlers/`; everything else is written by the assistant. `main` stays the tested reference.
- **Why:** the user must understand and defend every block. Typing it, then reviewing it with lint and a comparison
  against the guide, does that. The rebuild also brings in D13a (pymongo modules) and D20 revised (site variables).
- **Evidence:** user direction 2026-10-02; CLAUDE.md "Collaboration mode".

## D1. One role for RHEL 8, 9 and 10, x86_64 only

- **Decision:**
  - One role; the only OS-dependent values are the repo `baseurl` (`distribution_major_version`) and the Python
    packages for pymongo (D13a).
  - prelim asserts RedHat family, major in `mongodb8_cis_supported_os_majors`, and `architecture == x86_64`.
- **Why:**
  - An OS branch needs a real difference; the RPMs ship the same `mongod.conf`, unit, user and paths on el8/9/10 (S3).
  - **Architecture:** RHEL also runs on `aarch64`, `ppc64le` and `s390x`, and MongoDB ships packages for some of
    them, but the role was tested only on x86_64. The assert does not claim other architectures fail; it stops
    **before anything changes** instead of risking a half-applied hardening on an untested platform (fail fast).
- **Evidence:** S3. S2 metadata, `mongodb-enterprise-server` versions listed (read 2026-10-03):

  | | x86_64 | aarch64 | s390x | ppc64le |
  |---|---|---|---|---|
  | RHEL 8 | 29 | 29 | 29 | 29 |
  | RHEL 9 | 29 | 29 | 24 | 24 |
  | RHEL 10 | 4 | 4 | 0 | 0 |

- **To add an architecture:** make `mongodb8_cis_supported_arch` a list, test on that architecture (venv packages,
  SELinux build, CPU requirements), switch the assert to `in`.

## D2. AUDIT → PATCH (Lockdown standard)

- **Decision:** when a rule must read state before deciding, it is a `block`: a read-only AUDIT step stores the
  state in `discovered_<what>`, and the PATCH step runs only when that state shows drift.
- **Why:** a compliant host is never touched (no rewrite, no restart); the second run is `changed=0`.
- **Evidence:** L1.

## D3. No AUDIT step when the module audits itself

- **Decision:** rules done with declarative modules (`file`, `dnf`, `systemd_service`, `seport`) have no separate
  AUDIT step.
- **Why:** these modules compare and change only on drift; an extra read would duplicate them.
- **Evidence:** L2. Example: 7.2's PATCH `file` task needs no `when:` on the `stat` result.

## D4. Manual rules report by default

- **Decision:** Manual rules (1.1, 3.1–3.5, 4.5, 5.2, 6.2, 6.3, 7.1, 7.2) are `"<ID> | AUDIT | ..."` blocks tagged
  `audit`, printing a verdict with `debug`: `PASS`/`FAIL` when CIS gives the expected value, `REVIEW` when a human
  must judge, `NOT APPLICABLE` when the thing doesn't exist. They change nothing unless a site variable says so (D20).
- **Why:** CIS marks them Manual because the right state depends on the environment (S5, Assessment Status).
- **Evidence:** S5; L1 rule 5.4.2.3 (audit-only).

## D5. Tags on the rule block

- **Decision:** `level1`/`level2`, `automated`/`manual`, `patch`/`audit`, `rule_<id>` and a topic tag sit on the
  rule block. The optional PATCH inside a Manual rule adds its own `patch` tag. prelim is tagged `always`.
- **Why:** `--tags rule_7.2` or `--skip-tags rule_6.1` selects a whole rule; prelim must always run.
- **Evidence:** L1.

## D6. Disruptive rules: their own toggle defaults to `false`

- **Decision:** 2.1, 2.2, 4.3, 4.4, 6.1 default `false`, each with a `# WARNING:` in `defaults/main.yml`. No global
  `disruption_high` switch.
- **Why:** they can lock users out or break clients (auth, TLS, FIPS, port). CLAUDE.md principle 3. A global switch
  is not CIS, Lockdown uses it inconsistently (L3), and it hides which rules it covers.

## D7. How `/etc/mongod.conf` is changed

- **Decision:** prelim parses the file once into `discovered_mongod_conf`. A config rule writes
  `discovered_mongod_conf | combine(<its keys>, recursive=true) | to_nice_yaml` with `copy` (`root:root 0644`,
  `backup: true`), notifies `Restart mongod`, then updates `discovered_mongod_conf` with `set_fact`.
- **Why:**
  - Nested YAML; `lineinfile` can't edit nested keys safely. Merging keeps every site setting.
  - `recursive=true`: without it `combine` replaces a whole section (tested: `net.port` lost when adding `net.tls`).
  - `set_fact` after the write: the parsed config is a snapshot; without it the next rule writes from the old copy
    and erases the previous rule's change.
- **Trade-off:** comments and key order of the original file are lost on the first write (`backup: true` keeps it).
- **No `validate:`:** `mongod` has no config dry-run; the handler restarts and waits for the port, so a bad config
  fails the play at once.
- **Evidence:** [reading-config.md](reading-config.md) (debug-only tests, ansible-core 2.16.19 and 2.20.7).

## D8. Paths and user come from the live config

- **Decision:** `dbPath`, log path and service user are read from `mongod.conf` and the systemd unit (`status.User`).
  `vars/main.yml` holds the RPM values only as fallbacks: `/var/lib/mongo`, `/var/log/mongodb/mongod.log`, `mongod`.
- **Why:** the MongoDB install page says user `mongodb` and `/var/lib/mongodb`; the RHEL RPM uses `mongod` and
  `/var/lib/mongo` (S1 vs S3). Sites may move `dbPath`.
- **Accepted limit:** mongod's own built-in `dbPath` default is `/data/db` (S4); `/var/lib/mongo` only applies
  because the RPM's `mongod.conf` sets it. A config with `dbPath` deleted would make the fallback point to the
  wrong directory. Not handled: rare, and mongod normally won't start that way.

## D9. Enterprise-only rules are real rules

- **Decision:** 4.4 FIPS, 4.5 encryption at rest, 5.1 `auditLog`, 5.2 audit filter are implemented, not reported
  as gaps.
- **Why:** the role targets Enterprise (D12); S5 marks these as Enterprise features.

## D10. Rule 5.3 applied as written

- **Decision:** `systemLog.quiet: false`. A fresh install is already compliant, so the PATCH only runs if a site
  set `quiet: true`.
- **Evidence:** S5 5.3, S4 default.

## D11. Rule 6.1: non-default port, range check, SELinux label

- **Decision:**
  - Disruptive (D6). The port is a site value, `mongodb8_cis_port`, with no default from the role.
  - An assert stops the rule unless the port is **1024–65535 and not 27017**.
  - With SELinux enabled and a port the base policy doesn't already label (27017–27019, 28017–28019), the port is
    labelled `mongod_port_t` with `community.general.seport` before the restart.
- **Why the range** (the range is the role's guard; CIS itself only says *"$Orginasation Defined port"*, S5 6.1):

  | Limit | Reason | Evidence |
  |-------|--------|----------|
  | ≤ 65535 | A TCP port is a 16-bit number; 65536 and above don't exist | R1 |
  | ≥ 1024 | Ports below 1024 are privileged on Linux: only root or `CAP_NET_BIND_SERVICE` may bind them. mongod runs as the non-root `mongod` user (which CIS 3.3 requires), so it would fail to start | R3 |
  | ≥ 1024 | 0–1023 are IANA System Ports for well-known services (22, 80, 443, ...) | R2 |
  | ≠ 27017 | The CIS rule: *"ensure that the port number is not 27017"* | S5 6.1 Audit |

  The assert runs before `mongod.conf` is written, so a bad value never takes the database down.
- **Why the label:** on RHEL 8/9 the base policy confines `mongod` as `mongod_t`, which may only bind
  `mongod_port_t`; without the label mongod won't start under enforcing. On RHEL 10 it is harmless.
- **Firewall:** site policy (`bindIp` is `127.0.0.1` by default).
- **Evidence (2026-10-03):** [test-results.md](test-results.md) "SELinux baseline": RHEL 8/9 `mongod_t` + MongoDB file
  rules, RHEL 10 `unconfined_service_t`, no file rules, data labelled `var_lib_t`; port label present on all three.

## D12. Install is opt-in, from the Enterprise repo

- **Decision:** `mongodb8_cis_install: false`. When `true`: import the key, add `mongodb-enterprise-<version>` with
  `yum_repository`, install `mongodb-enterprise`, start and enable `mongod`.
- **Why:** the role's job is hardening; installing is a deliberate choice. The repo URL uses `$basearch` and the OS
  major, so one definition fits every target.
- **Evidence:** S1, S2. The key name changes at 9.0 (`server-8.0.asc` → `server-9.asc`), handled in `vars/main.yml`.

## D13. Superseded by D13a

`main` used `community.mongodb.mongodb_shell` only, to avoid installing anything on the DB servers.
**Changed in the rebuild:** replaced by D13a.

## D13a. Database access with the pymongo modules, pymongo in a venv on the server (new)

- **Decision:**
  - `tasks/pymongo.yml` (not a CIS rule) installs Python 3.12 (RHEL 8/9: `python3.12`, RHEL 10: system `python3`)
    and `pymongo>=4.9,<5` into `/opt/mongodb8_cis/venv`, only when Section 2 or 3 is on.
  - Sections 2 and 3 run with `ansible_python_interpreter` set to the venv's Python.
  - Modules: `mongodb_user` (2.1), `mongodb_info` (2.2, 3.1, 3.2, 3.5), `mongodb_shell` (3.4 privileges).
- **Why:**
  - User direction 2026-10-02: make full use of `community.mongodb`, even if pymongo must be installed on the target.
  - Real modules give structured results (no parsing of shell output) and idempotence (`mongodb_user` with
    `update_password: on_create` → rerun `changed=0`).
  - A venv keeps pip packages away from the RPM-managed system Python. RHEL 8's system Python is 3.6, too old for
    pymongo 4.9+ (needs 3.9+); EPEL's pymongo is old or missing (S8).
- **Cost (accepted):** Python 3.12 + pip + PyPI access (or a mirror) on each DB server.
- **Accepted gap:** `mongodb_info` does not list users of databases that hold no data.
- **Evidence:** tested on a real mongod 8.0.32 Enterprise with ansible-core 2.16.19 and 2.20.7 (2026-10-02).

## D14. CIS audit procedures map to the parsed config

- **Decision:** CIS audits for 2.1, 2.2, 4.2, 4.3, 5.1, 5.3, 5.4, 6.1 are `grep` on `mongod.conf` (S6); the role
  checks the same keys on the parsed dictionary. 6.2 reads systemd `Limit*` properties (`LimitFSIZE`, `LimitCPU`,
  `LimitAS`, `LimitNOFILE`, `LimitRSS`, `LimitNPROC`) from prelim's `systemd_service` result.
- **Why:** a parsed lookup can't be fooled by a commented-out key or a key in the wrong section. systemd's values
  already include drop-ins.

## D15. One control node: ansible-core 2.16 for RHEL 8, 9 and 10

- **Decision:** run from ansible-core `>=2.16.1,<2.17`; `min_ansible_version: "2.16.1"`; prelim asserts `>= 2.16.1`.
- **Why:** 2.17+ drops target Python 3.6 (RHEL 8's system Python, needed by `dnf`). 2.16 supports 3.6–3.12 (S7).
- **Known limit:** 2.16 is upstream EOL (July 2025), accepted.

## D16. Test plan

Default install (`--check`: mix of pass/fail), non-hardened (all fail), hardened (second run `changed=0`), per
RHEL 8, 9, 10 (S6 certification columns). **Not yet run on this branch:** batches 2–5 untested on VMs.

## D17. SELinux policy module: optional extra, off by default

- **Decision:** `mongodb8_cis_selinux_policy: false`. When on: build MongoDB's module (S9, copied in `files/selinux/`)
  on RHEL 9/10, report the base policy on RHEL 8, label non-default paths and port, restart.
- **Why:** not a CIS recommendation (S5 has no SELinux rule), so it is opt-in and documented as an extra. Built on
  the host because a module must match the base policy of that RHEL release (S1).
- **License:** `files/selinux/*` are MongoDB's GPL-2.0 files in an MIT role; headers kept.

## D18. Levels gate rules (Level 2 = L1 + L2)

- **Decision:** `mongodb8_cis_level_1: true`, `mongodb8_cis_level_2: false`. Every rule has `when:` on its level
  variable, plus `level1`/`level2` tags.
- **Why:** in Lockdown the level variables only feed Goss (L4), so setting one to `false` changes nothing, which
  misleads. S5: Level 2 *"extends"* Level 1. Default is the CIS baseline (Level 1).

| Level | Rules |
|-------|-------|
| 1 (13) | 1.1, 2.1, 2.2, 3.1, 3.2, 3.3, 3.4, 4.2, 4.3, 5.1, 6.1, 7.1, 7.2 |
| 2 (10) | 2.3, 3.5, 4.1, 4.4, 4.5, 5.2, 5.3, 5.4, 6.2, 6.3 |

## D19. Switches without `| bool`

- **Decision:** switches are used as-is in `when:`. `| bool` only for values read from `mongod.conf` (2.2
  `enableLocalhostAuthBypass`, 4.5 `enableEncryption`).
- **Why:** users set switches as unquoted YAML booleans in `group_vars`, or on the CLI as JSON
  (`-e '{"mongodb8_cis_rule_6_1": true}'`). Text values (`-e x=false`) fail on 2.19+ and silently run on 2.16, so
  they are documented as not supported. Matches Lockdown (0 uses of `| bool` in `RHEL9-CIS/tasks/`).

## D20. Manual rules: optional site variable where CIS gives one concrete fix

- **Decision:** a Manual rule gets a site variable next to its toggle **only when CIS's remediation is one concrete
  setting whose value the site decides**. Empty/false (default) = report only; set = AUDIT → PATCH.

  | Rule | Site variable | Default | When set |
  |------|---------------|---------|----------|
  | 3.1 | `mongodb8_cis_revoke_admin_roles` | `[]` | revokes `dbOwner`/`userAdmin`/`userAdminAnyDatabase` in admin from the **listed** accounts only (`<db>.<user>`); the account and its other roles stay |
  | 5.2 | `mongodb8_cis_audit_filter` | `""` | writes `auditLog.filter` (only when auditing is on) |
  | 6.3 | `mongodb8_cis_javascript_needed` | `true` | `false` → `security.javascriptEnabled: false` |
  | 6.2 | `mongodb8_cis_fix_resource_limits` + `mongodb8_cis_resource_limits` (CIS values) | `false` | systemd drop-in with the limits, restart (only on drift) |
  | 7.1 | `mongodb8_cis_fix_key_file_permissions` | `false` | existing keyFile / TLS key / CA → `0600`, owner mongod |
  | 7.2 | `mongodb8_cis_fix_db_path_permissions` | `false` | dbPath → `0770`, owner mongod |

  Report-only: 1.1 (never auto-upgrade), 3.2–3.5 (users and roles are a people decision), 4.5 (key management design).
- **Why:** the value is the site's decision, not the role's guess (CLAUDE.md principle 5). A Manual rule that
  only ever reports leaves the site to fix by hand what the role could apply safely once told.
- **Changed in the rebuild** (user direction 2026-10-02): 5.2 and 7.1 gained site variables; 6.2 on 2026-10-03; 3.1 on 2026-10-04.
- **3.1 details:** the site names accounts after reading the report, so the role never decides who loses a role. It
  revokes only the three roles CIS names, scoped to admin (`revokeRolesFromUser` via `mongodb_shell`), not the account
  (CIS: *"drop them"* is read as the roles). `mongodb_user` was not used: it rewrites the whole role list and clears the
  user's `authenticationRestrictions` (`updateUser` with `[]`, `mongodb_user.py` `user_add`). Runs only when the listed
  account still holds a flagged role, so a rerun is `changed=0`.
- **6.2 details:** the expected values are a variable (default = CIS 6.2 Remediation: f/t/v/m unlimited, n/u 64000)
  because CIS says *"Every deployment may have unique requirements"*. The fix is a drop-in
  (`/etc/systemd/system/mongod.service.d/mongodb8_cis-limits.conf`), never an edit of the RPM's unit, and runs only when
  systemd's values differ (a fresh install already matches, so it is untouched). The restart handler does
  `daemon_reload` so systemd reads the drop-in. Values are compared as text (`| string`), so `64000` and `"64000"` both work.

## D21. Rule 5.1: add `auditLog` only when missing, `syslog` by default

- **Decision:** compliant when `auditLog.destination` is set to anything; otherwise merge `destination`
  (`syslog` default; `console`/`file`), with `format`/`path` for `file`. An assert checks the values. An existing
  `auditLog` is never changed.
- **Why:** CIS lists the destinations without choosing (S5 5.1). syslog lets the OS rotate and forward events;
  overwriting a site's own `auditLog` would break their pipeline for no compliance gain.

## D22. Supported MongoDB version: 8.0, in `vars/`

- **Decision:** `mongodb8_cis_supported_versions: ["8.0"]` in `vars/main.yml`. prelim asserts the requested
  version before install and the **installed** series after detection.
- **Why:** the role implements the CIS MongoDB 8 benchmark; another series has its own benchmark.
- **Why `vars/`, not `defaults/`:** inventory and `group_vars` can override `defaults/` but not role `vars/`, so a
  user can't widen the list by accident. Extra vars (`-e`) still can: a deliberate bypass the user owns.

## D23. Role name `mongodb8_cis`

- **Decision:** role and variable prefix `mongodb8_cis`, named after the benchmark (Lockdown: `RHEL9-CIS`).
- **Why:** a later benchmark gets its own role and prefix; two roles in one play never share variables.

## D24. How the role reaches the database

- **Decision:** one `module_defaults` block for the `community.mongodb` modules around Sections 2 and 3, built from
  `discovered_mongod_running_conf` (prelim snapshot, never updated): host/port from `bindIp`/`net.port`, login only
  when `authorization: enabled`, TLS only when `requireTLS`, `tlsAllowInvalidHostnames` because the role talks to its
  own mongod by IP. prelim fails fast if auth is on and login values are empty or the password has spaces, quotes or
  backslashes.
- **Why:** edits made during the run only take effect after the restart handler, so the connection must use what
  mongod is running with now.
- **Changed in the rebuild:** covers `mongodb_info`/`mongodb_user` (D13a) as well as `mongodb_shell`.

## D25. Section 4 order: TLS (4.3) first

- **Decision:** `section_4/main.yml` imports 4.3 before 4.1, 4.2, 4.4. TLS option rules only write when
  `net.tls.mode` is enabled.
- **Why:** mongod 8.0.32 refuses to start with TLS options but no TLS mode (*"need to enable TLS via the
  sslMode/tlsMode flag when using TLS configuration parameters"*, tested 2026-10-01). A documented exception to
  "benchmark order".

## D26. Rules 2.1 and 2.2: create the admin before locking the door

- **Decision:**
  - 2.1: assert the admin values, `mongodb_user` creates the admin (`root` on `admin`, as in CIS's remediation,
    `no_log`), then `security.authorization: enabled`.
  - 2.2: `mongodb_info` counts users; assert at least one exists; then `enableLocalhostAuthBypass: false`.
  - 2.3: sharded clusters only; report, off by default.
- **Why:** S5 2.1 says create the administrator before enabling auth; S5 2.2: the localhost exception is the only
  way in without users. The assert blocks the one combination that locks everyone out.
- **Changed in the rebuild:** `mongodb_user`/`mongodb_info` instead of `mongosh` scripts (D13a).

## D27. Task layout: one folder per section, one file per rule

- **Decision:** `tasks/main.yml` → `section_<N>/main.yml` (section switch) → `cis_<N>.<M>.yml` (one per rule).
- **Why:** Lockdown layout (L1, L2); one rule = one small file to review.

## D28. Section 3 (Authorization): report only, except an opt-in revoke list for 3.1

- **Decision:** 3.2–3.5 report only; 3.1 reports and has one opt-in site variable (`mongodb8_cis_revoke_admin_roles: []`).
  Hand fixes: [manual-remediation.md](manual-remediation.md). User decision 2026-10-04/05.
- **How each rule was weighed** (same four questions for every rule):
  1. Does CIS give **one concrete fix** the role can apply? (D20)
  2. What does a **wrong** value break?
  3. What would the site have to **write** in `group_vars`?
  4. What does a **fresh install** already look like?

  | Rule | 1. Concrete fix in CIS? | 2. Wrong value breaks | 3. Site would write | 4. Fresh install | Result |
  |------|-------------------------|-----------------------|---------------------|------------------|--------|
  | 3.1 | **Yes**: names 3 roles in admin, *"then drop them"* | one listed account loses admin roles | a name: `admin.badadmin` | no users → PASS | opt-in list |
  | 3.2 | No: *"Establish roles … assign the appropriate users"*; "appropriate" is the site's call | apps lose roles; IP login limits silently wiped (E2) | user + db + password (Vault) + roles (+ restrictions) per account | no users | report only (built, then reverted: E3) |
  | 3.3 | Nothing to fix by default; FAIL fix is a repair job | mongod does not start if one root-owned file is missed | nothing useful | `User=mongod`, not root → PASS (E4) | report only |
  | 3.4 | No: `revokePrivilegesFromRole` with privileges the site picks | app can no longer run an action it needs | role + db + resource + actions per entry | 0 custom roles (E5) | report only |
  | 3.5 | No: *"Review"* superuser/admin roles | admins locked out of their own work | which admin accounts are legitimate | no users | report only |

- **Why:** automating 3.2–3.5 would turn the hardening role into an account-management tool whose input only the site's
  people can write, and whose mistakes lock out admins or break apps. The reports already give the reviewer every fact
  the CIS audit asks for. 3.1 is the exception because CIS names the exact roles and the site only names accounts.
- **Evidence:**

  | # | Source | Shows |
  |---|--------|-------|
  | E1 | CIS MongoDB 8 v2.0.0, 3.1–3.5 Remediation sections | the wording quoted above; all five are **Manual** |
  | E2 | `community.mongodb` 1.8.0 `plugins/modules/mongodb_user.py` `user_add`: `if exists or authentication_restrictions: user_dict["authenticationRestrictions"] = authentication_restrictions`, default `[]` | updating an existing user with `mongodb_user` clears its `authenticationRestrictions` unless the site repeats them |
  | E3 | Session 2026-10-04: `mongodb8_cis_db_users` list + `mongodb_user` task built, syntax-checked, then reverted by the user | the variable needed 4–5 nested fields per account: too complex for end users |
  | E4 | Lab VM (host `rhel8`), 2026-10-04: `systemctl show mongod -p User` → `User=mongod`; `getent passwd mongod` → `mongod:x:975:974:mongod:/var/lib/mongo:/bin/false`; `ps -o user -C mongod` → `mongod` | the RPM already meets 3.3 (CIS audit = process owner) |
  | E5 | MongoDB stores custom roles in `admin.system.roles`; a new install has none (**to confirm** in the rebuild test: 3.4 should report `0 user-defined role(s)`) | nothing for 3.4 to fix by default |

## Not adopted

| Not adopted | Why |
|-------------|-----|
| Lockdown's Goss audit (`setup_audit`, `run_audit`, `audit_only`) | Extra tool; AUDIT reports + `--check --diff` cover it |
| Per-OS vars files | No per-OS difference except D1/D13a values |
| Global `disruption_high` | D6 |
| `check_mode` trick for Manual rules (one `file` task as report) | Rerun shows `changed` every time, no readable verdict ([simplicity-review.md](simplicity-review.md) 4.1) |
| Section 3 fixes for 3.2–3.5 (declared users/roles via `mongodb_user`/`mongodb_role`, drop lists for 3.4/3.5) | User decision 2026-10-04: report only. A hardened database may hold custom admins on purpose; removing accounts or roles can lock out admins or break apps. **Revised 2026-10-04:** 3.1 got an opt-in revoke list (D20). A 3.2 declared-accounts list (`mongodb8_cis_db_users`, `mongodb_user`) was built the same day and reverted by the user as too complicated |
| 3.3 fix (switch a root-run mongod to a service account) | The RPM already creates `mongod` and runs the service as it, so 3.3 passes by default (CIS audit = process owner). The FAIL case means re-owning every file root created, often in unknown paths; one miss and mongod does not start. Rare and needs a person. Manual steps: [manual-remediation.md](manual-remediation.md) |
| 3.4 fix (`revokePrivilegesFromRole` from a site list) | User decision 2026-10-05: report only. The site would have to write role + db + resource + actions per entry (too much syntax), and only the app team knows which privilege is unneeded; a wrong entry breaks the app. A fresh install has no custom roles. Manual steps: [manual-remediation.md](manual-remediation.md) |
| 2.3 fix (set `clusterAuthMode` + `keyFile`/x509) | Writing the setting is easy, but it only works across a sharded cluster: one keyFile (or x509 member certs) on every member, `transitionToAuth`, rolling restart. The role targets a standalone mongod (Scope, top of this file), where 2.3 is not applicable; the rule reports N/A |
| 4.5 fix (enable encryption at rest) | User decision 2026-10-05: report only for now. MongoDB *"cannot encrypt existing data"* (dump/empty dbPath/restore), the keyfile method *"does not meet most regulatory key management guidelines"*, KMIP needs external infrastructure, and a lost key makes all data unreadable. Evidence and future options (fresh-install keyfile, KMIP): [manual-remediation.md](manual-remediation.md) 4.5 |
