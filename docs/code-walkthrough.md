# Code walkthrough — `mongodb_cis`

What each file does and why, briefly. Deeper reasons: `Dn` in [design-decisions.md](design-decisions.md).
**Written by:** the user types `tasks/` and `handlers/`; Claude writes the other files.

| # | File | By | Status |
|---|------|----|--------|
| 1 | `meta/main.yml` | user (fixed by Claude) | done |
| 2 | `.yamllint`, `.ansible-lint`, `.gitignore` | user | done |
| 3 | `defaults/main.yml` | Claude | done |
| 4 | `vars/main.yml` | Claude | done |
| 5 | `tasks/main.yml`, `tasks/prelim.yml` | user | done |
| 6 | `handlers/main.yml` | user | done |
| 7 | `tasks/install.yml` (+ prelim hook) | user | done, tested on RHEL 8 + 10 |
| 8 | `tasks/section_5.yml`: 5.1–5.4 (Audit Logging) | user | done |
| 9 | `tasks/section_1.yml` (1.1), `tasks/section_7.yml` (7.1, 7.2) + prelim service read | user | in progress |
| 10 | `tasks/section_6.yml` (6.1, 6.2, 6.3) | user | in progress |
| 11+ | Sections 2, 4, 3; README | user / Claude | — |

---

## 1. `meta/main.yml`: role metadata (no effect on hosts)

| Key | Does | Why |
|-----|------|-----|
| `author`, `description`, `license: MIT` | Who, what, licence | Required by lint for standalone roles; MIT as in Lockdown |
| `role_name` / `namespace: bunnywkwk` | Full name `bunnywkwk.mongodb_cis` | Namespace = owner (GitHub user), not the role |
| `min_ansible_version: "2.16.1"` | Oldest ansible-core allowed | RHEL 8 env floor; same as Lockdown (D15). Quoted = string |
| `platforms: EL 8/9/10` | Supported OS | Verified identical packaging (D1); versions must be quoted |
| `galaxy_tags` | Search keywords | Lowercase letters/digits only (`meta-no-tags`) |
| `collections: [community.mongodb]` | Declares the collection the role uses | For `mongodb_shell` (D13). Installed on the control node via `requirements.yml`, nothing on the servers |
| `dependencies: []` (top level) | Other **roles** to run first: none | Roles ≠ collections |

Fixed during the build: `dependencies` was indented under `galaxy_info` (in YAML, indentation = parent key); missing `---`; namespace was the role name.

### `requirements.yml` (Claude)

| Key | Does | Why |
|-----|------|-----|
| `community.mongodb` `>=1.8.0,<2.0.0` | Collection users install **on the control node**: `ansible-galaxy collection install -r requirements.yml` | Provides `mongodb_shell` (D13). A role can't install its own collections: Ansible needs them before the play is read. The range blocks a surprise 2.0 major |

## 2. Lint config and `.gitignore` (no effect on hosts)

| File / key | Does | Why |
|------------|------|-----|
| `.yamllint` `extends: default` + overrides | YAML style rules | Lockdown settings, trimmed |
| `line-length: disable` | No 80-char limit | CIS titles are long; Lockdown does the same |
| `octal-values` forbid | Rejects unquoted `mode: 0644` | Unquoted octal can set wrong permissions; write `"0644"` |
| `truthy` `true/false` only | No `yes/on` | CLAUDE.md style |
| `braces`/`brackets` `max-spaces-inside: 1` | Allows `{{ var }}` / `[ a ]` spacing | Required by ansible-lint (warned without it); Lockdown setting |
| `.ansible-lint` `profile: production` | Strictest rule set | CLAUDE.md validation standard; no skipped rules |
| `.gitignore` `.ansible/` | Keeps lint cache out of git | Machine-generated |

Run lint **inside** `mongodb_cis/`: yamllint reads `.yamllint` from the current directory.

## 3. `defaults/main.yml`: the user's control panel

All user-facing variables, at Ansible's lowest precedence, so `group_vars`/`host_vars` override them without editing tasks.
Rule parameters (port, TLS paths, admin user) are added next to their toggle when that rule is built.

| Variable(s) | Default | Does | Why |
|-------------|---------|------|-----|
| `mongodb_cis_install` | `false` | Add the MongoDB repo + install if missing | Opt-in (D12); set `true` for empty VMs |
| `mongodb_cis_version` | `"8.0"` | Release series in the repo URL | One variable, not one repo per version. Quoted: `8.10` unquoted would become `8.1` |
| `mongodb_cis_level_1` / `_level_2` | `true` / `false` | Gate Level 1 / Level 2 rules | Real gates, not Goss-only; for Level 2 set both `true` (D18) |
| `mongodb_cis_disruption_high` | `false` | Allows 2.1, 2.2, 4.3, 6.1 | These can lock users out or break clients (D6, Lockdown) |
| `mongodb_cis_section1..7` | `true` | Turn a whole section on/off | Lockdown convention |
| `mongodb_cis_rule_<id>` | `true` | One switch per CIS rule, commented with exact title, level, type | Skip a rule without editing tasks |
| `mongodb_cis_rule_2_3` | `false` | Sharded-cluster rule | Role targets standalone |
| `mongodb_cis_port` | `""` | Site port for 6.1 (1024–65535, ≠ 27017) | CIS gives no value (*"Organisation Defined port"*); empty → 6.1 stops with a message (D11) |
| `mongodb_cis_javascript_needed` | `true` | Site decision for 6.3: `false` → disable server-side JS | Manual rule, one clear fix (D20); CIS warns it blocks all server-side scripts |
| `mongodb_cis_fix_db_path_permissions` | `false` | Site decision: also fix 7.2 (dbPath → 0770, owner mongod) | 7.2 is Manual → report by default; fix only when the site decides (D20) |

A rule runs only if **section + level + rule toggle** (+ `disruption_high` for the 4 risky rules) are all `true`.
Manual and Enterprise-only rules stay `true`: they only report (D4, D9).

**Sections vs levels:**
- A section is the topic.
- A level is the strictness of each rule, so one section mixes both (3.4 = L1, 3.5 = L2).
- CIS definitions:
  - Level 1: *"practical and prudent"*, doesn't *"inhibit the utility … beyond acceptable means"*.
  - Level 2: *"extends"* Level 1, for *"security is paramount"* / *"defense in depth"*, and may cost functionality (e.g. 6.3 blocks server-side scripts).

## 4. `vars/main.yml`: internal constants

High precedence; **not** for users to change (they use `defaults/`). Values from platform-notes S1/S3.

| Variable(s) | Value | Why |
|-------------|-------|-----|
| `mongodb_cis_supported_os_majors`, `_supported_arch` | `8/9/10`, `x86_64` | Only what was verified (D1); prelim asserts against them |
| `mongodb_cis_repo_baseurl`, `_repo_gpgkey` | Built from OS major + `mongodb_cis_version` | One definition for all OS/versions (S1) |
| `mongodb_cis_package`, `_server_package`, `_service`, `_conf_path` | `mongodb-org`, `mongodb-org-server`, `mongod`, `/etc/mongod.conf` | Names from the RPM (S3); server package = "is MongoDB installed?" |
| `mongodb_cis_default_db_path`, `_log_path`, `_user` | `/var/lib/mongo`, `/var/log/mongodb/mongod.log`, `mongod` | RPM defaults, **fallback only**: live values come from `mongod.conf` (D8) |
| `mongodb_cis_service_user` | systemd `User=` of `mongod` (read in prelim), fallback `mongod` | The real service account for 7.x/3.3. CIS text says `mongodb` (Debian); RHEL uses `mongod` (D8) |
| `mongodb_cis_listen_port`, `_listen_host` | `net.port` (default 27017); first `bindIp` entry, `0.0.0.0` → `127.0.0.1` | Where the handler checks mongod is back. From the live config, so it follows a 6.1 port change. Tested on 2.16 and 2.20 |

## 5. `tasks/main.yml` + `tasks/prelim.yml`: entry point and discovery (read-only)

`main.yml` imports `prelim.yml` with tag `always`, so every run (even `--tags rule_5.4`) does discovery first.
Section files are added to `main.yml` as they are written. Every prelim task is **AUDIT**: nothing on the host changes.

| Task | Does | Why |
|------|------|-----|
| Gather minimal facts if missing | `setup: gather_subset: min` only when facts are absent | Role still works if the play set `gather_facts: false` |
| Check ansible-core version | Assert `>= 2.16.1` | Project floor = Lockdown's (D15) |
| Check supported OS | Assert RedHat family, major in 8/9/10, x86_64 | Fail fast with a clear message (CLAUDE.md) |
| Check ansible-core for RHEL 8 | Assert `< 2.17` on RHEL 8 | `dnf` breaks on RHEL 8 with 2.17+ (D15) |
| Install MongoDB when requested | `import_tasks: install.yml` if `mongodb_cis_install \| bool` (section 7) | Runs **before** detection, so detection sees the result; no duplicated detection code |
| Gather installed packages | `package_facts` (rpm) | Detect, don't assume (CLAUDE.md app roles) |
| Detect MongoDB server package | `discovered_mongodb_installed`, `discovered_mongodb_version` | `discovered_*` = Lockdown naming for audit results |
| Stop when not installed | Report + `meta: end_host` whenever the server package is missing | Clean skip, not a failure (CLAUDE.md app roles). Also covers `--check` with install on: the install is only simulated, so there is no `mongod.conf` to read. Only this host stops |
| Read + parse `mongod.conf` | `slurp` → `b64decode` → `from_yaml` → `discovered_mongod_conf` | The shared AUDIT all config rules compare against (D7, D14). `slurp` is read-only and works in `--check` |
| Show what was found | Version, config path, dbPath, log path | CLAUDE.md: audit reports version, config path; paths from live config with RPM fallback (D8) |

Checked on the draft: lint `production` passes; `--syntax-check` passes on ansible-core 2.20 and 2.16.

**`| bool` on every user switch:**
- `-e mongodb_cis_install=true` passes the string `"true"`.
- ansible-core 2.20 fails the run: *"Conditionals must have a boolean result"* (seen in the first install test).
- 2.16 treats the string `"false"` as true, so it would install anyway.
- `| bool` makes both versions behave the same. The rule applies to all toggles in rules too.

## 6. `handlers/main.yml`: restart mongod safely

Handlers run once at the end of the play, only if a task `notify:`s them. Both handlers `listen: Restart mongod`,
so a PATCH writes `notify: Restart mongod` and gets both, in order.

| Handler | Does | Why |
|---------|------|-----|
| Restart mongod service | `systemd_service: state: restarted` | mongod reads `mongod.conf` only at start (D7). `systemd_service` is idempotent and exists in 2.16 |
| Wait for mongod to accept connections | `wait_for` on the listen host/port, 60 s | `mongod.service` is not a notify-type unit, so "restarted" does not mean "ready". A bad config fails here, right after the change that caused it (D7: no `validate:` for mongod) |

`listen:` instead of one handler notifying another: both steps share one trigger name, which is simpler to read and to notify.

## 7. `tasks/install.yml`: optional install (only when `mongodb_cis_install: true`)

Imported by prelim. Follows the official install steps (platform-notes S1). Every module is declarative, so there's **no AUDIT step** (D3): a second run changes nothing.

| Task | Does | Why |
|------|------|-----|
| Import signing key | `rpm_key` from `server-<version>.asc` | Explicit trust in MongoDB's key before any package is installed; packages are verified (`gpgcheck`) |
| Add repository | `yum_repository` `mongodb-org-<version>` | Same repo id/URL as S1, so a host set up by hand gets the identical file; URL built from OS major (D1) |
| Install packages | `dnf` `mongodb-org`, `state: present` | `present`, not `latest`: hardening never upgrades by surprise (lint `package-latest`). Needs the 2.16 env on RHEL 8 (D15) |
| Start and enable mongod | `systemd_service` started + enabled | A fresh install is not started; prelim then reads the created `mongod.conf` |

**Check mode:** not supported for a first install (the repo is only simulated, so dnf can't find the package). Install with a real run. Accepted limitation (T-M5).

## 8. Rule 5.4: first AUDIT → PATCH (`tasks/section_5.yml` + hook in `tasks/main.yml`)

CIS 5.4 (L2, Automated): *Ensure that new entries are appended to the end of the log file*. Target: `systemLog.logAppend: true`.

**Where is the AUDIT?**
- **Read:** prelim's shared read of `mongod.conf` → `discovered_mongod_conf` (D7). CIS audits this the same way: `grep logAppend /etc/mongod.conf` (D14).
- **Compare:** the block's `when:` checks `logAppend` against `true`.
- **PATCH:** runs only on drift.
- Compliant host → block skipped → file untouched, no restart.

| Part | Does | Why |
|------|------|-----|
| `main.yml` hook | Imports `section_5.yml` when `mongodb_cis_section5 \| bool` | Section switch (Lockdown); `\| bool` (D19) |
| Block `when:` rule toggle + level | `mongodb_cis_rule_5_4 \| bool`, `mongodb_cis_level_2 \| bool` | 5.4 is Level 2 (D18) |
| Block `when:` drift check | `logAppend \| default(false) is not true` | **The AUDIT decision.** A missing key counts as drift: mongod's built-in default is `false` (S4) |
| `tags` | `level2, automated, patch, rule_5.4, logging` | On the block (D5): `--tags rule_5.4` / `--skip-tags rule_5.4` |
| `vars: mongodb_cis_5_4_settings` | The keys this rule owns | Written once, used by both tasks |
| Task 1 `copy` | Writes current config + 5.4 keys (`combine(..., recursive=true)`), root:root 0644 as shipped, `backup: true` | Keeps every site setting; nested YAML can't be `lineinfile`d (D7). Comments are lost on the first write; the backup keeps the original |
| Task 1 `notify: Restart mongod` | Restart + port check at the end | mongod reads the config only at start (section 6) |
| Task 2 `set_fact` | Updates `discovered_mongod_conf` in memory | The next rule builds on the new config, not the old one |

Tested on the draft: the merge keeps `net`, `storage`, `processManagement` and flips only `logAppend` (2.16 + 2.20); lint `production` passes.

### Rules 5.1–5.3 (same file, **above** 5.4: benchmark order)

| Rule | Type | Does | Why |
|------|------|------|-----|
| 5.1 (L1, Automated) | `AUDIT`, `debug` | Reports "not applicable: auditLog is Enterprise-only" with the installed version | Community can't do it; report the gap, don't hide it (D9). Tagged `audit` even though CIS says Automated: nothing can be patched |
| 5.2 (L2, Manual) | `AUDIT`, `debug` | Same report for audit filters | Needs auditLog (Enterprise) (D9); Manual rules never patch (D4) |
| 5.3 (L2, Automated) | `PATCH`, same pattern as 5.4 | Drift = `systemLog.quiet` is `true`; fix = `quiet: false` | Missing key = default `false` = compliant (S4), so a fresh install is skipped. Applied on Community despite CIS's "Enterprise only" note (deviation D10) |

`--tags audit` runs only the report rules (5.1, 5.2) plus prelim: a read-only report.

## 9. Sections 1 and 7 (+ one prelim task)

**Prelim addition:** `systemd_service` with **no `state`** only reads the unit (`changed=false`, safe in `--check`), registered as `discovered_mongod_service`. It gives `mongodb_cis_service_user` (from `User=`) and, later, 6.2's limits. The report shows "Service user".

| Rule | Type | Does | Why (CIS text) |
|------|------|------|----------------|
| 1.1 (L1, Manual) | AUDIT | Prints the installed version + where to compare (latest 8.0.x, mongodb.com/alerts) | CIS audit = check the version; remediation = upgrade. Never automatic in hardening (D20) |
| 7.1 (L1, Manual) | AUDIT | Collects `security.keyFile`, `net.tls.certificateKeyFile`, `net.tls.CAFile` (`select` drops empty ones), `stat` each, reports mode/owner; "not applicable" if none | CIS audit greps the same three settings and runs `ls -l`. Standalone without TLS/keyFile has none |
| 7.2 (L1, Manual) | AUDIT (+ opt-in PATCH) | `stat` of dbPath → PASS/FAIL vs `0770` owner/group = service user. If `mongodb_cis_fix_db_path_permissions` → `file:` sets it | CIS remediation: `chmod 770`, `chown` to the mongo user. **RPM ships 0755 → fresh install FAILS**. `file:` is declarative, so no extra audit (D3) |

Tested with a fake config on localhost (2.16 + 2.20): 7.1 N/A, 7.1 per-file incl. a missing file, 7.2 FAIL on 0755. Lint `production` passes.

## 10. Section 6: OS Hardening (6.1, 6.2, 6.3)

| Rule | Type | Does | Why |
|------|------|------|-----|
| 6.1 (L1, Automated) | PATCH, gated by `disruption_high` | Assert `mongodb_cis_port` is set, 1024–65535, ≠ 27017 → if `net.port` differs, merge it into `mongod.conf`, restart, wait on the **new** port | CIS: any non-default port, site-defined. Risky: breaks clients (D6, D11). Ports <1024 need root; mongod runs as `mongod` |
| 6.2 (L2, Manual) | AUDIT | Prints the six limits of the running unit (`Limit*` from prelim's systemd read) next to the CIS values | CIS lists f/t/v/m unlimited, n/u 64000. The shipped unit already matches (platform-notes) |
| 6.3 (L2, Manual) | AUDIT + opt-in PATCH | Reports `security.javascriptEnabled` (default `true`) and the site decision. If `mongodb_cis_javascript_needed: false` → sets it to `false` | "If not needed" is the site's call (D20). CIS impact: blocks all server-side scripts |

**Port inline, not in `vars:`** (T-M6): on 2.16 a templated value in `vars:` becomes text (`port: '27018'`). Computed values go inside `combine({...})`; literal `true`/`false` in `vars:` are fine.

Tested on localhost with the RPM `mongod.conf` (2.16 + 2.20): writes `port: 27018` and `javascriptEnabled: false`, keeps `bindIp`/`dbPath`, one restart, run 2 `changed=0`; missing port → clear stop.
