# Design decisions — `mongodb8_cis`

Every decision below has three parts: **Decision**, **Why**, and **Evidence**. Source IDs (S1–S9) are defined in
[platform-notes.md](platform-notes.md#sources). Four more sources are used here:

| # | Source |
|---|--------|
| L1 | ansible-lockdown `RHEL9-CIS` (branch `devel`), `tasks/section_5/cis_5.4.2.x.yml` rule 5.4.2.2: AUDIT then PATCH in one block |
| L2 | ansible-lockdown `RHEL9-CIS`, `tasks/section_1/cis_1.1.1.x.yml` rule 1.1.1.1: declarative `lineinfile`, no AUDIT step |
| L4 | ansible-lockdown `RHEL9-CIS` (devel, cloned 2026-09-29): `grep -rn rhel9cis_level_ tasks/` finds nothing. The level vars appear only in `templates/lockdown_audit.yml.j2` and `templates/etc/ansible/compliance_facts.j2`. README, *Matching a security Level for CIS*: *"This is managed using tags"*. Tag count: 237 rules tagged level1 only, 57 level2 only, 8 both |
| L3 | ansible-lockdown `RHEL9-CIS`, `defaults/main/main.yml` (`rhel9cis_disruption_high: false`, `rhel9cis_section1`, `rhel9cis_level_1`) and `defaults/main/audit.yml` (`setup_audit`, `run_audit`, `audit_only`) |

Scope: **MongoDB Enterprise 8.0, `mongod` (standalone; 2.3 also on shard, config server and replica set members, D31), RHEL 8, 9 and 10**, CIS MongoDB 8 Benchmark v2.0.0 (S5, S6).
Chosen with the user on 2026-09-29 (RHEL 8 added the same day). Edition changed from Community to Enterprise on 2026-10-01 (D12).

> Recovery note: this file and `platform-notes.md` were rebuilt on 2026-09-29 after an accidental
> `ansible-galaxy role init --force` deleted `mongodb8_cis/`. Content is the last version from the working session.

---

## D1. One role for RHEL 8, 9 and 10, no OS branches

- **Decision:** a single `mongodb8_cis` role. The only OS-dependent value is the repo `baseurl`, built from `ansible_facts['distribution_major_version']`.
- **Why:** an OS branch is only justified by a real difference, and none was found.
- **Evidence:** S1 gives the same install steps for 8/9/10; only the `baseurl` number changes. S3: the el8, el9 and el10 RPMs ship a byte-identical `mongod.conf` and systemd unit, with the same user and paths. S2: the same 8.0.32 build exists for all three.
- **Not a task branch:** RHEL 8's OpenSSL 1.1.1 (vs 3.x) changes no config key; TLS rules are tested on each OS. RHEL 8's Python is handled by D15.

## D2. The AUDIT → PATCH pattern (Lockdown standard)

- **Decision:** when a rule needs to read state before deciding, it is a `block`:
  1. An **AUDIT** step reads the current state (read-only, `changed_when: false`) and stores it in `discovered_<what>`.
  2. A **PATCH** step runs only when the AUDIT result shows the host is non-compliant.
- **Why:** a compliant host is never touched: no file rewrite, no `mongod` restart. The second run reports `changed=0`, and the play output shows per rule whether the host was compliant (PATCH skipped) or fixed (PATCH changed).
- **Evidence:** L1. Lockdown rule 5.4.2.2 runs an AUDIT task with `register: discovered_gid0_members`, then a PATCH task with `when: discovered_gid0_members.stdout | length > 0`.

## D3. …but no AUDIT step when the module already audits itself

- **Decision:** rules implemented with declarative modules (`file`, `lineinfile`, `systemd_service`, `dnf`) have **no** separate AUDIT step.
- **Why:** these modules compare desired vs current state and only change on drift. An extra AUDIT task would duplicate that and slow the run. This is the "don't over-engineer" limit.
- **Evidence:** L2. Lockdown rule 1.1.1.1 uses `lineinfile` directly inside a PATCH block, with no AUDIT task.

## D4. Manual rules only report

- **Decision:** Manual recommendations (1.1, 3.1–3.5, 4.5, 5.2, 6.2, 6.3, 7.1, 7.2) are `"<ID> | AUDIT | ..."` rules tagged `audit`. They gather facts and print them with `debug`. They never change anything.
- **Why:** CIS marks them Manual because the right value is site-specific (e.g. which roles a user needs, whether server-side JavaScript is needed). Automating a guess could break the application.
- **Evidence:** S5 marks these as "(Manual)". L1 rule 5.4.2.3 is an AUDIT-only rule that warns but does not fix.

## D5. Tags on the rule block, not per step

- **Decision:** tags such as `level1`/`level2`, `automated`/`manual`, `patch`/`audit`, `rule_<id>` and a topic go on the rule block. Shared reads (`package_facts`, parsed `mongod.conf`) live in `prelim.yml`, tagged `always`.
- **Why:** this matches Lockdown, so `--tags rule_2.2` or `--skip-tags rule_6.1` selects a whole rule. Prelim must always run because every rule depends on it.
- **How users run it:**
  - Read-only check of everything: `--check --diff`.
  - Manual-rule report only: `--tags audit`.
- **Evidence:** L1. The tags on rule 5.4.2.2 (`level1-server`, `patch`, `rule_5.4.2.2`, ...) sit on the block; the inner AUDIT task inherits them.

## D6. Disruptive rules are off by default (their own toggles)

- **Decision:** rules that can lock out users or break clients have their rule toggle set to `false` in `defaults/main.yml`, each with a `# WARNING:` comment. The site turns each one on when it is ready:
  - 2.1 `mongodb8_cis_rule_2_1`: create the admin user, enable authorization.
  - 2.2 `mongodb8_cis_rule_2_2`: disable the localhost exception.
  - 4.3 `mongodb8_cis_rule_4_3`: require TLS.
  - 4.4 `mongodb8_cis_rule_4_4`: FIPS mode (it disables SCRAM-SHA-1 and non-FIPS ciphers, S4 FIPS page).
  - 6.1 `mongodb8_cis_rule_6_1`: change the port.
- **Why:** CLAUDE.md principle 3: anything that can lock users out needs *"a toggle, a safe default, and a `# WARNING:` comment"*, and the naming rule says rule toggles default to *"`true` (or `false` for manual/risky)"*. A fresh install fails all of these rules (platform-notes "Fresh-install state"); enabling them without an admin user, certificates or updated client connection strings breaks access.
- **Revised 2026-10-01 (user decision):** these rules used to stay `true` and were held back by one extra switch, `mongodb8_cis_disruption_high: false`, copied from Lockdown (L3). That switch is not part of CIS or any standard, Lockdown itself uses it inconsistently (RHEL 8/9/10 roles default `false`, UBUNTU22-CIS defaults `true`, Windows-2022-CIS has none), and it hid which rules it covered. It was removed; each risky rule is now visible and switched on by name.

## D7. How `/etc/mongod.conf` is changed

- **Decision:**
  - `prelim.yml` reads the file once (`slurp` + `from_yaml`) into `discovered_mongod_conf`. This is the shared AUDIT for all config rules.
  - A config rule's PATCH writes `discovered_mongod_conf | combine(<rule's keys>, recursive=true) | to_nice_yaml` with `ansible.builtin.copy` (owner `root`, mode `0644`, as shipped), then refreshes `discovered_mongod_conf`.
  - Every PATCH notifies one handler, `Restart mongod`.
- **Why:**
  - The file is nested YAML (S3), and `lineinfile` cannot safely edit nested keys.
  - Merging keeps every site setting (dbPath, bindIp, replication, ...), so the change is non-destructive.
  - A rule only writes when its AUDIT shows drift, so compliant hosts keep their file untouched.
- **Trade-off (accepted):** the first time a rule writes, the comments in `mongod.conf` are lost. `backup: true` keeps the original.
- **Rejected alternative:** templating the whole file from role variables. It would overwrite site settings the role does not own.
- **No `validate:`:** `mongod` has no config dry-run option. The handler restarts the service and waits for the port, so a bad config fails the play immediately.

## D8. Paths come from the live config, not from the docs

- **Decision:** `dbPath`, the log path and the service user are read from `mongod.conf` and the systemd unit. The user comes from prelim's read-only `systemd_service` call (`status.User`) → `mongodb8_cis_service_user`. The package defaults (`/var/lib/mongo`, `/var/log/mongodb/mongod.log`, `mongod`) are used only as fallbacks in `vars/main.yml`.
- **Why:** the MongoDB install page and the RHEL package disagree. The docs say user `mongodb` and `/var/lib/mongodb`; the RPM uses `mongod` and `/var/lib/mongo`. Sites may also move `dbPath`.
- **Evidence:** S1 vs S3 (see platform-notes, "The docs and the package disagree").

## D9. Enterprise-only rules: applied, not reported as gaps

The benchmark marks these as Enterprise features. This role targets Enterprise, so each is a normal rule:

| Rule | Enterprise feature | Evidence | Role behaviour |
|------|--------------------|----------|----------------|
| 4.4 FIPS mode (L2, Automated) | `net.tls.FIPSMode` | S4 FIPS page; S5 4.4 | Built with Section 4 |
| 4.5 Encryption at rest (L2, Manual) | encrypted storage engine (KMIP or keyfile) | S4 encryption page; S5 4.5 *"Available in MongoDB Enterprise only"* | Manual → report only (D4), built with Section 4 |
| 5.1 System activity audited (L1, Automated) | `auditLog` | S4 config options *"Available only in MongoDB Enterprise"*; S5 5.1 | PATCH when `auditLog.destination` is missing (D21) |
| 5.2 Audit filters (L2, Manual) | `auditLog.filter` | S5 5.2 *"This check is only for Enterprise editions"* | AUDIT: reports the filter (D4) |

- **Revised 2026-10-01:** while the role targeted Community, these four were reported as "not applicable". The switch to Enterprise (D12) made them real rules.

## D10. Rule 5.3 follows the benchmark

- **Decision:** apply 5.3 (`systemLog.quiet: false`) as written.
- **Why:** S5 5.3 says *"This check is only for Enterprise editions"*, and the role targets Enterprise. A fresh install is already compliant (`quiet` not set, default `false`, S3/S4), so the PATCH only runs if someone set `quiet: true`.
- **Revised 2026-10-05 (user decision, compliance test on RHEL 10):** the PATCH now also runs when `quiet` is **absent**, so
  `mongod.conf` always contains `quiet: false`. Why: CIS's audit is `cat /etc/mongod.conf | grep "quiet"` and its
  remediation says *"Set quiet: false"*; with the key absent the grep finds nothing, although the effective value is
  already `false`. Cost: one more key on a fresh install, written in the same run (and restart) as 5.1. Condition:
  `... | default(true) is not false` (same pattern as 6.3).
- **Revised 2026-10-01:** this was a deviation (5.3 applied on Community) waiting for the mentor's confirmation. With Enterprise it is no longer a deviation.

## D11. Rule 6.1 (non-default port) and SELinux

- **Decision:**
  - The rule is disruptive (D6).
  - The port is a site value, `mongodb8_cis_port`, with no default invented by the role; the rule stops with a clear message if it is not 1024–65535 or is 27017.
  - When SELinux is enabled and the port is not one the base policy already labels (27017-27019, 28017-28019), 6.1 installs `policycoreutils-python-utils` and labels the port `mongod_port_t` (`community.general.seport`) before the restart.
- **Why:** CIS gives no port number (*"$Orginasation Defined port"*, S5 6.1). On RHEL 8 and 9 the base policy already confines `mongod` as `mongod_t` (platform-notes "SELinux"), so it can only bind `mongod_port_t` ports; without the label `mongod` would not start under enforcing. On RHEL 10 `mongod` is unconfined unless the SELinux extra is on, and the label is harmless.
- **Firewall:** stays site policy (`bindIp` is `127.0.0.1` by default, so no firewall change is needed).
- **Revised 2026-10-01:** the earlier text said no port label was needed with the default SELinux state. That held for RHEL 10 only.
- **Status:** the merge and the restart wait on the new port are tested (Section 6 tests); the `seport` step needs a host with SELinux enabled: **to verify on the VMs.**

## D12. Install is opt-in

- **Decision:** `mongodb8_cis_install: false`. When `true`, the role creates the S1 Enterprise repo (`mongodb-enterprise-<version>`) with `yum_repository` and installs `mongodb-enterprise`.
- **Why:** the role's job is hardening. Installing must be a deliberate choice, and it is needed now only because the test VMs are empty.
- **Evidence:** S1 repo definition and install command. CLAUDE.md "Application roles: Install is opt-in".
- **Revised 2026-10-01:** switched from the Community repo (`repo.mongodb.org`, `mongodb-org`) to Enterprise (`repo.mongodb.com`, `mongodb-enterprise`), the edition in use. Community and Enterprise packages conflict, so a Community host must be cleaned first (platform-notes "Installation"). The key URL follows MongoDB's naming change at 9.0 (`server-8.0.asc` → `server-9.asc`), so `mongodb8_cis_version` alone still selects the repo.

## D13. Talking to the database (rules 2.1, 3.x): `community.mongodb.mongodb_shell`

- **Decision:**
  - Database reads and the 2.1 admin bootstrap use **`community.mongodb.mongodb_shell`**, with `changed_when: false` and `check_mode: false` on read-only queries. Credentials are variables with `no_log: true`.
  - The collection's **pymongo-based** modules (`mongodb_user`, `mongodb_info`, `mongodb_parameter`, ...) are **not** used.
- **Why this module:**
  - Mentor's rule: prefer a real module over `command`/`shell`.
  - `mongodb_shell` (collection 1.8.0) lists only `requirements: mongosh`. The pymongo import in its `module_utils` is optional (`try/except ImportError`).
  - `mongosh` is installed by `mongodb-org` on RHEL 8/9/10 (S1), so there's **nothing extra on the database servers**. The collection is installed on the control node only (`requirements.yml`).
  - Its code parses as Python 3.6 (static check), so it's expected to work on RHEL 8 via the 2.16 env. **To confirm with one real run.**
  - It declares `supports_check_mode=False`, so read-only queries set `check_mode: false` (they only read). It reports `changed` by default, so reads set `changed_when: false`.
- **Why not the pymongo modules** (checked 2026-09-30):
  - The collection README says: *"PyMongo - latest version supported only."* Latest pymongo = **4.18.2, `requires_python >=3.9`** (PyPI).
  - RHEL 8's **system** Python 3.6 can't run it (it would need a separate `python3.12` venv on the server). RHEL 9 EPEL = pymongo 3.10.1 (2020); RHEL 10 EPEL = 4.8.0 → neither is "latest". EPEL itself isn't in the base repos (S8).
  - **Benefit is small:** mapped against all 23 rules, the pymongo modules would only simplify 2.1's user creation (`mongodb_user`) and the 3.1–3.5 reports (`mongodb_info`). The other ~17 rules are `mongod.conf` keys, a startup-only parameter (2.2), systemd or file permissions, which the collection has no module for. Its `mongodb_config`/`mongodb_mongod` roles template the whole file (conflicts with D7).
  - **Cost is on every DB server:** a Python 3.9+ venv with pymongo, PyPI/internet access (or an internal mirror), per-task `ansible_python_interpreter`, and keeping pymongo at "latest". That adds software and supply-chain surface to the servers being hardened.
  - **Would reconsider if** the organisation already has an internal PyPI mirror and allows pip-installed software on DB servers.
- **Confirmed** by the user on 2026-09-30 after comparing all three options (`command` / `mongodb_shell` / full pymongo modules). Chosen for clean code (a module, not `command`) with **no extra software on the servers being hardened**.
  - Making them work would need EPEL or `pip install` into system Python on every DB server (mixing pip with RPM Python), and still fails on RHEL 8.
  - Running them from the control node instead needs mongod reachable over the network, but it binds `127.0.0.1` by default. Exposing it would weaken the hardening.
- **Config-file rules are unaffected:** the collection has no module to edit individual `mongod.conf` keys, so 2.2/4.x/5.x/6.1 stay `copy`/`stat` (D7). 2.2's `enableLocalhostAuthBypass` is startup-only (S4), so `mongodb_parameter` couldn't set it anyway.
- **Revised (twice):**
  1. First `community.mongodb` pymongo modules → dropped after checking EPEL (S8).
  2. Then `ansible.builtin.command: mongosh` → replaced by `mongodb_shell` on 2026-09-30 to follow the mentor's no-`command` rule.

## D13a. Revised 2026-10-02: use the collection's pymongo modules, with pymongo in a venv on each server

> **Branch `mongodb8-cis-rebuild` only.** `main` keeps D13 (`mongodb_shell`, nothing installed on the server). On
> `main`, 2.1, 2.2 and 3.1–3.5 read `admin.system.users` with CIS's own queries, so every user is seen; `mongodb_info`
> misses users of databases without data (found 2026-10-05).

- **Decision (user):**
  - Rules 2.1, 2.2, 3.1, 3.2 and 3.5 use `community.mongodb.mongodb_user` / `mongodb_info`, for more readable tasks. 2.1's user creation becomes one idempotent task (`state: present`, `update_password: on_create`).
  - The role prepares each server:
    1. installs Python 3.12 from the OS repos;
    2. creates a venv `/opt/mongodb8_cis/venv` with `pymongo>=4.9,<5`;
    3. runs the database tasks of sections 2 and 3 with `ansible_python_interpreter` = that venv.
- **Exception, 3.4 keeps `mongodb_shell`:** `mongodb_info` reads roles with `rolesInfo` + `showBuiltinRoles` but **without** `showPrivileges` (collection 1.8.0 source, `get_roles_info`). CIS 3.4's audit is `rolesInfo … showPrivileges: true`, so only `mongodb_shell` can show what CIS asks for.
- **Evidence:**
  - MongoDB driver compatibility tables (`mongodb.com/docs/drivers/compatibility`): **pymongo 4.9+ is fully compatible with MongoDB 8.0**; 4.4–4.8 partially; pymongo 4.14–4.18 support CPython 3.9–3.14.
  - EPEL pymongo is 3.10 (EL9, incompatible) and 4.8 (EL10, partial), so not usable.
  - Python 3.12: `python3.12` in EL8 and EL9 AppStream (Rocky repodata checked), system `python3` on EL10.
  - Collection compat check: requires pymongo ≥ 4 and MongoDB ≥ 4 (`mongodb_common.check_compatibility`), so 8.0 passes.
  - TLS/login options: `login_*`, `tls`, `tlsCAFile`, `tlsCertificateKeyFile`, `connection_options`. `login_password` is `no_log`.
- **Cost, accepted:**
  - Every server needs PyPI (or an internal mirror) for the first run.
  - The role installs Python 3.12 + pip packages on servers it hardens, documented in the README.
  - pymongo must stay ≥ 4.9.
- **On the rebuild branch,** supersedes D13's "no pymongo" choice. On `main`, D13 stays in force.

## D14. What the CIS spreadsheet (S6) tells us about AUDIT steps

- **Most automated audits are config-file reads.** S6 "Audit Procedure": 2.1, 2.2, 4.2, 4.3, 5.1, 5.3, 5.4 and 6.1 are all `cat /etc/mongod.conf | grep <key>`. Our shared AUDIT (D7: `slurp` + `from_yaml` → `discovered_mongod_conf`) checks the same keys, structurally. That is more reliable than `grep`, which can't tell a commented-out key or a key in the wrong section.
- **Legacy `ssl` vs `tls` names.** S6 4.2 greps under `ssl`, while 4.1/4.3 use `net.tls`. MongoDB 4.2 renamed `net.ssl.*` to `net.tls.*` and kept `ssl` as deprecated aliases. The role writes only `net.tls.*`. The AUDIT for 4.1/4.2/4.3 flags a legacy `net.ssl` block as drift, so it can't hide an old setting.
- **6.2 reads the unit's effective limits.** S6 6.2 audits `/proc/<mongod PID>/limits`. The role uses prelim's read-only `systemd_service` result (`Limit*` properties), which already merges drop-ins, so no extra task. CIS letters map to systemd as f=`LimitFSIZE`, t=`LimitCPU`, v=`LimitAS`, n=`LimitNOFILE`, **m=`LimitRSS`** (resident memory, *not* `LimitMEMLOCK`, which is `ulimit -l`), u=`LimitNPROC`. *Revised 2026-10-01: the earlier plan read `/proc/<pid>/limits`.*
- **4.1 and 4.2 check the same key.** Both require `TLS1_0,TLS1_1` in `net.tls.disabledProtocols` (S5/S6), at L2 and L1 respectively. Each rule keeps its own ID, toggle and block, so skipping one does not skip the other; whichever runs first fixes the key and the other then finds it compliant. Both only write when TLS is enabled (D25).
- **Defaults per S6 "Default Value":** authorization disabled (2.1), `enableLocalhostAuthBypass` `true` (2.2), `javascriptEnabled` enabled (6.3). This agrees with the fresh-install state in platform-notes.

## D15. One control-node environment: ansible-core 2.16 for RHEL 8, 9 and 10

- **Decision:**
  - The role is run from one ansible-core `>=2.16.1,<2.17` environment for all targets (setup: [control-node-setup.md](control-node-setup.md)).
  - The role declares `min_ansible_version: "2.16.1"` in `meta/main.yml`, uses only 2.16 features, and `prelim.yml` asserts `ansible_version.full is version('2.16.1', '>=')`.
- **Why 2.16:**
  - RHEL 8's system Python is 3.6. ansible-core 2.17+ does not support Python 3.6 on targets (S7), and a newer Python on RHEL 8 does not help, because the `dnf` bindings exist only for 3.6.
  - 2.16 supports target Python 2.7 and 3.6–3.12, so one environment covers RHEL 8 (3.6), 9 (3.9) and 10 (3.12).
  - The floor is 2.16.1, as in Lockdown (`ansible-lockdown/RHEL9-CIS` `meta/main.yml`: `min_ansible_version: 2.16.1`); `pip` installs the newest 2.16 patch anyway.
- **Revised 2026-10-01 (user decision):** the second prelim assert (`< 2.17` when the target is RHEL 8) was removed. Every run comes from the 2.16 environment, so it always skipped. The requirement is documented in the README and in control-node-setup.md instead. Remaining risk, accepted: someone running 2.17+ against RHEL 8 is not stopped by the role; with the system Python 3.6 that fails at fact gathering, but with a newer Python forced on RHEL 8 it would only fail at the first `dnf` task.
- **Known limit:** ansible-core 2.16 reached upstream end of life in July 2025 (no more security fixes). Controller Python must be 3.10–3.12.
- **Tested:** the whole role ran from 2.16.19 on the lab VMs (RHEL 9.8 and RHEL 10). **RHEL 8 still to verify.**

## D16. Test plan follows the CIS certification grid (S6 columns E–G)

S6 has three result columns per rule. Our VM test runs use the same three states on RHEL 8, 9 and 10:

| State | Expected | How we produce it |
|-------|----------|-------------------|
| Default installation | mix of Pass/Fail (see platform-notes "Fresh-install state") | fresh VM + `mongodb8_cis_install: true`, run with `--check` |
| Non-hardened | all Fail | edit `mongod.conf` to violate every automated rule, run `--check` → every PATCH reports "would change" |
| Remediated/Hardened | all Pass | normal run, then a second run → `changed=0` |

Column H ("exceptions") is where deviations from the benchmark text are recorded. None are open since the switch to Enterprise (D9, D10).

## D17. SELinux policy for `mongod`: optional extra, off by default

> **Removed 2026-10-10 (user):** the extra (`tasks/selinux.yml`, `files/selinux/`, `mongodb8_cis_selinux_policy`) is not
> part of the role for now, to keep it to what CIS asks. 6.1 still labels a new port (`seport`), which mongod needs to
> start under SELinux. The history below is kept.

- **Decision:** `mongodb8_cis_selinux_policy: false`. When `true` and SELinux is enabled, `tasks/selinux.yml` (tag `selinux_policy`, no CIS ID):
  1. Installs `policycoreutils-python-utils`, plus `selinux-policy-devel` (which brings `make`, `checkpolicy`) on RHEL 9/10.
  2. **RHEL 9/10:** copies MongoDB's module sources (S9, pinned in `files/selinux/`) to `/usr/share/mongodb8_cis/selinux/`, builds `mongodb.pp` with `/usr/share/selinux/devel/Makefile` and loads it with `semodule --priority 200`, as the upstream `Makefile` does. It rebuilds only when the sources changed or no priority-200 `mongodb` module is loaded.
  3. **RHEL 8:** reports that the base policy's `mongodb` module already confines `mongod`. MongoDB's module is not installed: it does not build there.
  4. **All:** labels a non-default `dbPath` and log directory (`community.general.sefcontext`) and a non-default port (`seport`), runs `restorecon -R -v` on the binary, unit, `dbPath` and log directory (changed only when it relabels), and reports the domain `mongod` was running in.
  5. Any change notifies the restart, so `mongod` comes back in the new domain.
- **Why build on the host:** S1 documents it that way, and a module must be built against the base policy of the release it runs on (RHEL 8, 9 and 10 ship different ones, platform-notes "SELinux"). The earlier plan (build once on the control node, copy the `.pp`) would need a build per RHEL release outside the role. Cost: `selinux-policy-devel`, `make`, `checkpolicy` on the database server, all from the RHEL repos. This is why it is opt-in.
- **Why the sources are copied, not cloned:** no `git` and no internet access needed on the server, and the policy version is fixed by the role.
- **Why optional:** the CIS MongoDB 8 benchmark has no SELinux recommendation (S5); CLAUDE.md says never present an extra control as CIS. It is documented in the README as an extra.
- **Limits:** the role labels only the paths and port it reads from `mongod.conf`. Other paths a site adds (backup dirs, a custom audit log path, certificates outside standard locations) are the site's job. MongoDB maintains the module, so each MongoDB or RHEL update needs a retest.
- **Tested 2026-10-01** in containers (test-results "SELinux extra"); loading into a running kernel, labels and `mongod_t` at runtime: **to verify on the VMs.**
- **Fixed 2026-10-01 after the first VM run (RHEL 9.8):** loading MongoDB's module removed the base module's `mongod_tmp_t`, so the restarted mongod could not unlink its old socket and aborted. The role now removes `mongodb-*.sock` right after loading the module (troubleshooting T-M7).
- **Revised:** 2026-09-29 (twice: proposed, removed as out of scope, re-added as an optional extra, user decision); 2026-10-01: built on the host per S1, RHEL 8 handled separately after the container tests.

## D18. Level 1 / Level 2 selection

**Rule levels** (S6 sheets `Level 1- MongoDB` and `Level 2 - MongoDB`; same as the PDF):

| Level | Rules |
|-------|-------|
| **Level 1** (13) | 1.1, 2.1, 2.2, 3.1, 3.2, 3.3, 3.4, 4.2, 4.3, 5.1, 6.1, 7.1, 7.2 |
| **Level 2** (10) | 2.3, 3.5, 4.1, 4.4, 4.5, 5.2, 5.3, 5.4, 6.2, 6.3 |

S5 "Profile Definitions": Level 2 *"extends the 'Level 1 - MongoDB' profile"*. So selecting Level 2 means **L1 + L2 rules**, never L2 alone.

**How Lockdown does it (L4):** level selection is by **tags** only. `rhel9cis_level_1` / `rhel9cis_level_2` (both default `true`) only feed the Goss audit and the compliance-facts file. Setting `rhel9cis_level_2: false` does **not** skip Level 2 remediation. Level 1 rules carry only `level1-*` tags and Level 2 rules only `level2-*`, so `--tags level2-server` alone runs just the L2-only rules, **without** the L1 base.

**Decision for this role:**
- **Variables really gate the rules:**
  - `mongodb8_cis_level_1: true` and `mongodb8_cis_level_2: false`.
  - Every Level 1 rule has `when: mongodb8_cis_level_1`; every Level 2 rule has `when: mongodb8_cis_level_2`. The per-rule toggle (`mongodb8_cis_rule_<id>`) still applies on top.
- **Tags kept as well,** Lockdown-style: `level1` or `level2`. The MongoDB benchmark has no server/workstation split, so there is no `-server` suffix. A full Level 2 run by tags is `--tags level1,level2`.
- **Profiles expressed with variables:**
  - Level 1 profile (default): `level_1: true`, `level_2: false`.
  - Level 2 profile: `level_1: true`, `level_2: true`.
  - `level_1: false`, `level_2: true` runs L2 rules without their base, which CIS does not define. The README documents it; there is no runtime warning (removed as unnecessary, user decision 2026-09-29).

**Why it differs from Lockdown:**
- We don't use Goss (see "Not adopted"). A level variable that changes nothing would mislead users; the exact misunderstanding L4 shows is easy to have.
- Default **Level 1 only:** S5 defines Level 1 as *"practical and prudent"* and not limiting utility, and Level 2 as for environments *"where security is paramount"*. Lockdown enables both by default; we choose the CIS baseline and make Level 2 an explicit choice.
- **Confirmed** by the user on 2026-09-29, after comparing it with pure Lockdown (tags only, no level vars): keep vars that gate rules, plus tags. Everything else matches Lockdown: section switches, rule toggles, level tags, AUDIT → PATCH (`disruption_high` was dropped on 2026-10-01, D6).

## D19. Switches must be real YAML booleans (`| bool` removed 2026-10-01)

**In plain words:**
- When you type `-e mongodb8_cis_install=false` on the command line, Ansible receives the **text** `"false"`, not the value *false*. AWX/Tower surveys do the same.
- Text that isn't empty counts as "yes", so without `| bool`, "false" can mean **true**.
- `| bool` turns the text back into a real yes/no: `"true"`/`"yes"`/`true` → true, `"false"`/`"no"`/`false` → false.
- It is **not** about `--check`; it's about where the value comes from. Values from `defaults/` or `group_vars` YAML are already real booleans, and `| bool` leaves them unchanged. So it's never harmful, and it protects every way a user can set a switch.

**Evidence 1: the real error** (2026-09-30, RHEL 10 install test, ansible-core 2.20.7, T-M4):
```
[ERROR]: Task failed: Conditional result (True) was derived from value of type 'str' at "<CLI option '-e'>". Conditionals must have a boolean result.
Origin: .../mongodb_cis/tasks/prelim.yml:39:9
39   when: mongodb_cis_install
```

**Evidence 2: the silent danger on 2.16.** A debug-only playbook on localhost, `when: mongodb8_cis_install` vs `when: mongodb8_cis_install | bool`:

| ansible-core | `-e mongodb8_cis_install=` | Without `\| bool` | With `\| bool` |
|--------------|---------------------------|--------------------|-----------------|
| 2.20.7 (RHEL 9/10 env) | `true` | ❌ run stops with the error above | ✅ runs |
| 2.20.7 | `false` | ❌ run stops with the error above | ✅ skipped |
| 2.16.x (RHEL 8 env) | `true` | ⚠️ runs, only a deprecation warning | ✅ runs |
| 2.16.x | `false` | 🚨 **runs**: printed `INSTALL WOULD RUN (value=false, type=str)` | ✅ skipped |

So without `| bool`, 2.20 breaks loudly, and 2.16 does **the opposite of what the user asked**, silently. In a hardening role, that could mean a disruptive rule running when someone explicitly set it to `false`.

**Evidence 3: where the value comes from decides everything.** Same debug-only test, `flag: false`, **no `-e`**:

| Source of `flag` | 2.20.7 without `\| bool` | 2.16.19 without `\| bool` | Either version with `\| bool` |
|------------------|---------------------------|----------------------------|--------------------------------|
| YAML (`defaults/`, `group_vars/*.yml`, `inventory.yml`) | ✅ skipped | ✅ skipped | ✅ skipped |
| INI inventory (`flag=false` in `inventory.ini`) | ❌ run stops | 🚨 **runs** (`flag=false, type=str`) | ✅ skipped |

So `| bool` changes **nothing** for real YAML booleans. It only matters when the value arrives as **text**:
- `-e key=value`
- INI inventories
- AWX/Tower surveys and extra vars
- quoted YAML (`"false"`)

**What the references say:**
- Ansible docs, *Conditionals based on variables*: *"you must apply the `| bool` filter to non-boolean variables, such as string variables with content like 'yes', 'on', '1', or 'true'."*
- ansible-core 2.19 porting guide: non-boolean conditionals now fail (*"Conditionals must have a boolean result"*). Before, truthy strings *"masked serious logic errors."*
- **Lockdown RHEL9-CIS uses `| bool` nowhere** (0 matches in `tasks/`). It expects users to give real YAML booleans. **This is a deliberate deviation from Lockdown.**

**Original decision (2026-09-30):** every user-facing switch used in a `when:` was written `<var> | bool`, because the two ansible-core generations fail in opposite ways on text values (hard stop vs silent wrong run).

**Revised 2026-10-01 (user decision): `| bool` removed from the switches**, as in Lockdown (0 uses in `RHEL9-CIS/tasks/`):
- The switches are set in `group_vars` YAML files, not with `-e`, so they arrive as real booleans; `| bool` changes nothing for them.
- **Kept only where the value comes from `mongod.conf`** (2.2 `setParameter.enableLocalhostAuthBypass`, 4.5 `security.enableEncryption`): the file is written by people and may hold `"false"` or `0`.
- Values the role computes itself (`mongodb8_cis_tls_enabled`, `_shell_auth`, `_shell_tls`, `_selinux_build`) are real booleans on both versions, also inside `{{ … if … else … }}`.
- Re-tested on 2.20.7 and 2.16.19 (`when:` without `| bool`):

| Value | 2.20.7 | 2.16.19 |
|-------|--------|---------|
| YAML `false` (`group_vars`, `defaults`) | skipped | skipped |
| computed `"{{ 'a' == 'b' }}"` | skipped | skipped |
| `-e '{"x": false}'` (JSON) | skipped | skipped |
| `"false"` (quoted YAML) or `-e x=false` | **error**, play stops | **runs** (deprecation warning only) |

**Rule for users (README, test docs):** write switches unquoted (`true` / `false`), never `"false"`. To set one on the command line, use JSON: `-e '{"mongodb8_cis_rule_6_1": true}'`, never `-e mongodb8_cis_rule_6_1=false`.

## D20. Manual rules: report by default, optional "site decision" variable where CIS gives one clear fix

- **Decision:** a Manual rule gets a site variable next to its toggle **only when CIS's remediation is one concrete
  setting whose value the site decides**. Empty/false (default) = report only; set = AUDIT → PATCH.

  | Rule | Site variable | Default | When set |
  |------|---------------|---------|----------|
  | 3.1 | `mongodb8_cis_revoke_admin_roles` | `[]` | revokes `dbOwner`/`userAdmin`/`userAdminAnyDatabase` in admin from the **listed** accounts only (`<db>.<user>`, `mongodb_shell` `revokeRolesFromUser`); the account and its other roles stay |
  | 3.4 | `mongodb8_cis_drop_custom_roles` | `[]` | drops the **listed** custom roles (`<db>.<role>`, `dropRole`); users that held one lose it (D30) |
  | 3.5 | `mongodb8_cis_revoke_superuser_roles` | `[]` | revokes every 3.5 superuser/admin role from the **listed** accounts; never the role's own admin (D30) |
  | 5.2 | `mongodb8_cis_audit_filter` | `""` | writes `auditLog.filter` (only when auditing is on) |
  | 6.2 | `mongodb8_cis_fix_resource_limits` + `mongodb8_cis_resource_limits` (CIS values) | `false` | systemd drop-in with the limits, restart (only on drift) |
  | 6.3 | `mongodb8_cis_javascript_needed` | `true` | `false` → `security.javascriptEnabled: false` |
  | 7.1 | `mongodb8_cis_fix_key_file_permissions` | `false` | existing keyFile / TLS key / CA → `0600`, owner and group mongod |
  | 7.2 | `mongodb8_cis_fix_db_path_permissions` | `false` | dbPath → `0770`, owner mongod |

  Report-only: 1.1 (never auto-upgrade), 3.2, 3.3 (D28), 4.5 (key management design, evidence in
  [manual-remediation.md](manual-remediation.md)).
- **Why:** CIS marks a rule Manual because *"the expected state can vary depending on the environment"* (S5,
  Assessment Status). A variable set by the site is the site's decision, not the role guessing (CLAUDE.md principle 5).
- **Status:** agreed 2026-09-30 (6.3, 7.2). **Revised 2026-10-05:** 3.1, 5.2, 6.2 and 7.1 gained site variables, ported
  from branch `mongodb8-cis-rebuild` (built there 2026-10-02…04). The earlier reasons for keeping them report-only no
  longer hold: 6.2 "the shipped unit is already compliant" (true on a fresh install, so the drop-in is never written
  there; it fixes drifted hosts), 7.1 "no keyFile on standalone" (4.3 adds TLS key and CA files), 5.2 "organization's
  requirements" (the filter is exactly that requirement, given by the site).
- **3.1 on `main`:** the report keeps CIS's own audit query (any listed role **and** any role in admin, so it can list
  e.g. `dbOwner@shop` + `read@admin`); the revoke only removes a role that is one of the three **and** on admin, so such
  an account is reported but nothing is revoked.
- **6.2 details:** the drop-in is `/etc/systemd/system/mongod.service.d/mongodb8_cis-limits.conf`; the restart handler
  does `daemon_reload` so systemd reads it. Values are compared as text (`| string`), so `64000` and `"64000"` both match.

## D21. Rule 5.1: add `auditLog` only when it is missing, `syslog` by default

- **Decision:**
  - AUDIT: rule 5.1 is compliant when `auditLog.destination` is set to anything. This is the S5 audit (*"confirm the auditLog.destination value is set"*), done on the parsed config (D14).
  - PATCH (only when missing): merge `auditLog` from site values, then restart (D7):
    - `mongodb8_cis_audit_log: {destination: syslog}`: the `auditLog:` block, written as-is (`syslog`, `console`, or `file` with `format` JSON/BSON and `path`).
    - **Revised 2026-10-05 (user decision):** replaces three variables (`_audit_destination`, `_audit_format`, `_audit_path`) and the computed default path (`auditLog.<format>` next to `systemLog.path`). The variable now mirrors CIS's own remediation block, and the task's `vars:` is one line like 5.3/5.4. Cost: a `file` user writes the path. Tested on ansible-core 2.16.19 and 2.20.7 (syslog, file/BSON, file without path → assert, existing `auditLog` untouched, rerun `changed=0`).
    - An assert stops the rule with a clear message on any other value.
  - An existing `auditLog` is **never changed**, even if it differs from the variables.
- **Why `syslog` as default:**
  - CIS lists syslog, console, JSON file and BSON file without picking one (S5 5.1), so the default is ours.
  - With syslog, the OS log stack (journald/rsyslog) rotates the events and can forward them to central log management (CIS Control 8.2, cited by 5.1). A file is never rotated by mongod on its own and can fill the disk.
  - S4 limit: syslog messages can be truncated without any error. Sites that need complete records choose `file`.
- **Why never change an existing `auditLog`:** any destination passes CIS, and a site that set one chose it. Overwriting it would break their log pipeline for no compliance gain, and merging over it could leave `format`/`path` behind with a different destination.
- **Not disruptive:** its toggle stays `true` (D6). Auditing locks nobody out, but it adds load and log volume and restarts `mongod` once (warning in `defaults/main.yml`).
- **Tested 2026-10-01** on the scratch copy of the shipped `mongod.conf`, ansible-core 2.20.7 and 2.16.19:
  - syslog, file/JSON and file/BSON with a custom path: run 1 `changed=1` + restart, run 2 `changed=0`.
  - An existing `auditLog` is left untouched; an invalid destination stops with the assert message.
  - The written files started the real `mongod` 8.0.32 Enterprise binary: syslog events appeared in the journal, and file mode wrote JSON audit records.

## D22. Supported MongoDB version: 8.0 (CIS MongoDB 8 Benchmark)

- **Decision:**
  - `vars/main.yml` `mongodb8_cis_supported_versions: ["8.0"]`.
  - prelim asserts twice: (A) `mongodb8_cis_version` is in the list, before any install, and (B) the **installed** series (`<major>.<minor>` of `discovered_mongodb_version`) is in the list, after detection.
  - Install stays version-driven: repo name, URL and key come from `mongodb8_cis_version` (D12).
- **Why:**
  - CLAUDE.md principle 1: CIS is the source of truth. The role implements S5, which covers MongoDB 8.x; hardening another series with it would be a guess.
  - Check (B) uses the installed version, not the variable: detect, don't assume (CLAUDE.md app roles).
- **Adding a version later:** when a matching CIS benchmark exists, compare it with S5. Same rules → add the series to the list. Different → `vars/cis_<version>.yml` per CLAUDE.md.

## D23. Role name `mongodb8_cis`: one role per CIS benchmark

- **Decision:** the role is `mongodb8_cis` (`bunnywkwk.mongodb8_cis`), and every variable uses the `mongodb8_cis_` prefix. Renamed from `mongodb_cis` on 2026-10-01 (user decision).
- **Why:**
  - Lockdown names a role after the benchmark it implements (`RHEL8-CIS`, `RHEL9-CIS`), and CLAUDE.md prefixes variables with that short name (`rhel9cis_`). This role implements the CIS MongoDB 8 Benchmark (D22), so the name says which one.
  - A later benchmark (another MongoDB major, or the Windows platform) gets its own role and prefix, so two roles in one playbook never share variables.
  - Prefix style `<role>_` (with underscores) matches the sibling role `chrome_cis` (`chrome_cis_*`).

## D24. How the role reaches the database (rules 2.x, 3.x)

- **Decision:**
  - One `module_defaults` for `community.mongodb.mongodb_shell` on the block that imports Sections 2 and 3 (`tasks/main.yml`), so every DB task stays short.
  - The connection is built from `discovered_mongod_running_conf`, a prelim snapshot of `mongod.conf` taken **before** any rule edits it. Edits only apply after the restart handler, so the snapshot is what `mongod` is really running with.
  - Host/port: first `bindIp` entry (`0.0.0.0` → `127.0.0.1`) and `net.port`.
  - Login: `mongodb8_cis_admin_user`/`_password` only when the running config has `authorization: enabled`; otherwise no login (full access, as on a fresh install).
  - TLS: only when the running mode is `requireTLS`. CA = running `net.tls.CAFile`. Client certificate = `mongodb8_cis_shell_tls_certificate_key_file`, or the server `certificateKeyFile` (it needs EKU `clientAuth`, or no EKU). `--tlsAllowInvalidHostnames` is passed because the role talks to its own `mongod` on the same host, while certificates are normally issued for the FQDN.
  - prelim fails fast when Sections 2/3 are on, authorization is on and the login values are empty.
- **Limits of `mongodb_shell` (collection 1.8.0 source, `plugins/modules/mongodb_shell.py`):**
  - `--password` is added to the command line **unquoted** and the line is split with `shlex`, so a password with spaces, quotes or backslashes breaks the call. prelim rejects those characters (tested on 2.16 and 2.20).
  - The password is a command-line argument of `mongosh` while it runs, so other local users can see it in the process list for that moment. Mitigation, if needed: mount `/proc` with `hidepid=2`. Accepted to keep D13's "no extra software on the DB servers".
  - The TLS client certificate is passed with `ssl_keyfile` (`--tlsCertificateKeyFile`); `ssl_certfile` is accepted by the module but never used.
- **Tested 2026-10-01** end to end (see test-results "Pre-VM end-to-end").

## D25. Section 4 order: TLS first

- **Decision:** `tasks/section_4/main.yml` imports **4.3 before 4.1 and 4.2** (a documented exception to CLAUDE.md "benchmark order"). 4.1, 4.2 and 4.4 only write when `net.tls.mode` is enabled; otherwise they report FAIL with the reason.
- **Why:** the real `mongod` 8.0.32 Enterprise binary refuses to start with TLS options but no TLS mode: *"need to enable TLS via the sslMode/tlsMode flag when using TLS configuration parameters"* (tested 2026-10-01). Writing `disabledProtocols` on a default install (4.2 is Level 1) would leave `mongod` unable to start at the next restart. With 4.3 first, one run enables TLS and then restricts protocols.
- **4.3 site values:** `mongodb8_cis_tls_certificate_key_file` and `mongodb8_cis_tls_ca_file`, falling back to the values already in `mongod.conf`. The PEM files must exist on the host (asserted). The rule only runs when the mode is not `requireTLS`, so existing TLS settings are never rewritten.
- **4.4 FIPS:** S4 requires TLS first; the role checks that. On the test workstation (OpenSSL 3, OS FIPS mode off) `mongod` still logged *"FIPS 140 mode activated"*, so OS FIPS mode is not asserted. S4's OpenSSL 3 list names RHEL 9 but not RHEL 10: **to verify on the RHEL 10 VM.**
- **4.5** is Manual: it reports `security.enableEncryption` and the key management in use (KMIP or keyfile).

## D26. Rules 2.1 and 2.2: create the admin user before locking the door

- **Decision (2.1):** when `security.authorization` is not `enabled` (and `mongodb8_cis_rule_2_1` is on, D6):
  1. Assert `mongodb8_cis_admin_user`/`_password` are set.
  2. AUDIT: `getUser()` on `admin`.
  3. PATCH: `createUser` with role `root` on `admin`, as in the S5 remediation, only if the user is missing (`no_log: true`; the JSON-encoded values go inside the `eval`).
  4. PATCH: `security.authorization: enabled` + restart.
- **Decision (2.2):** when `enableLocalhostAuthBypass` is not false: count users in `admin.system.users`; assert at least one exists (skipped in `--check`, where 2.1 only simulates the user); then `setParameter.enableLocalhostAuthBypass: false` + restart. The value is a YAML boolean, which the real `mongod` accepts (tested).
- **Why this order:** S5 2.1 says to create the administrator before enabling authorization, and S5 2.2 says the localhost exception is the only way in when no user exists. The assert stops the one combination that locks everyone out.
- **2.3** targets sharded clusters; the role targets standalone, so it reports (`NOT APPLICABLE` without `sharding.clusterRole`) and is off by default.

## D27. Task layout: one folder per section, one file per rule (Lockdown)

- **Decision:** `tasks/main.yml` imports `section_<N>/main.yml` (with the section switch); each `section_<N>/main.yml` imports one file per rule, `cis_<N>.<M>.yml`, named `"SECTION | <ID> | <title>"`. The rule's toggle, level and tags stay in its own file.
- **Why:** this is the Lockdown layout (L1, L2: `RHEL9-CIS/tasks/section_5/main.yml` imports `cis_5.1.x.yml` ...). Lockdown groups by sub-heading (`5.3.1.x`); the MongoDB benchmark has no sub-headings, so each rule is its own file. A rule is found and reviewed in one small file.
- **Done 2026-10-01** (user request). Verified: the split files, joined in order, load to exactly the same YAML as the old `section_<N>.yml` files.

## D28. Section 3 (Authorization): report only, except an opt-in revoke list for 3.1

> **Revised 2026-10-06 by D30:** 3.4 and 3.5 also got opt-in name lists. 3.2 and 3.3 stay report only.

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

## D29. Coverage: every rule is implemented; 7 stay report-only on purpose

> **Revised 2026-10-10 by D31:** 2.3, 3.2 and 4.5 moved to opt-in; **2 only report: 1.1, 3.3.**
>
> **Revised 2026-10-06 by D30:** 3.4 and 3.5 moved to opt-in. Now 18 can change the host (10 Automated + 8 Manual with an
> opt-in variable) and **5 only report: 1.1, 2.3, 3.2, 3.3, 4.5.**

- **Decision:** all 23 recommendations have a task. 16 change the host (10 Automated PATCH rules + 6 Manual rules with
  an opt-in site variable, D20). 7 only report: 1.1, 2.3, 3.2, 3.3, 3.4, 3.5, 4.5. User decision 2026-10-05.
- **How a rule was weighed** (an option is added only if the answer to all four is "yes"):
  1. **One concrete fix:** does CIS's remediation name a setting or command the role can apply as written?
  2. **Simple input:** can the site express its decision as one simple value (a boolean, a string, a short list)?
  3. **Safe if wrong:** if the value is wrong, is the damage limited and reversible (no data loss, no admin lockout)?
  4. **Worth it:** does a fresh install fail the rule, or can it drift later?

  | Rule | 1. Concrete fix | 2. Simple input | 3. Safe if wrong | 4. Worth it | Result |
  |------|-----------------|-----------------|------------------|-------------|--------|
  | 1.1 | upgrade binaries (CIS: backup, stop, replace, restart) | version + change window | ❌ an upgrade restarts the database and can't be rolled back by the role | yes | report only |
  | 2.3 | `clusterAuthMode: x509` on every cluster member | certificates per member | ❌ rolling restart of a whole cluster | ❌ no cluster: the role targets a standalone `mongod` | report N/A |
  | 3.2 | ❌ "assign the **appropriate** roles" | ❌ 4–5 nested fields per account (D28 E3) | ❌ apps lose roles; login restrictions wiped (D28 E2) | — | report only |
  | 3.3 | nothing on a fresh install (`User=mongod`, D28 E4) | — | ❌ re-owning files: one missed file and mongod won't start | ❌ already passes | report only |
  | 3.4 | ❌ privileges the site picks per role | ❌ role + resource + actions per entry | ❌ an app loses an action it needs | ❌ no custom roles by default | report only |
  | 3.5 | ❌ "**review**" admin roles | ❌ which admins are legitimate | ❌ admins locked out | — | report only |
  | 4.5 | ❌ a procedure: master key, KMIP, rotation | ❌ key server design | ❌ **data loss**: existing data can't be encrypted in place; lost key = unreadable data ([manual-remediation.md](manual-remediation.md) 4.5, E1–E5) | — | report only |

- **Why:** for these seven, automating means either guessing a value only the site's people know, or building a
  complex input that end users would get wrong, where a mistake costs an outage, a lockout or data. A clear report plus
  a written hand fix ([manual-remediation.md](manual-remediation.md)) is safer and still CIS-correct: CIS marks 1.1,
  3.x and 4.5 **Manual**, which only requires a person to review them.
- **Full weighing per rule** (what an option would look like, cost, risk): [automation-decisions.md](automation-decisions.md).
- **Revisit when:** a site provides a KMIP server (4.5 option B in manual-remediation.md), or the role adds replica
  set/sharding support (2.3).

## D30. Rules 3.4 and 3.5: opt-in name lists (revises D28, D29)

- **Decision (2026-10-06, user direction; implementation accepted by the user the same day):** let a site target
  specific objects, like 3.1. Empty list (default) = report only.

  | Rule | Variable | Entry | Does | Guard |
  |------|----------|-------|------|-------|
  | 3.4 | `mongodb8_cis_drop_custom_roles: []` | `<db>.<role>` as the 3.4 report prints it | `db.getSiblingDB(db).dropRole(role)` | only custom roles are listed (built-in roles can't be dropped) |
  | 3.5 | `mongodb8_cis_revoke_superuser_roles: []` | `<db>.<user>` as the 3.5 report prints it | `revokeRolesFromUser` with every 3.5 role the account holds (`root`, `dbOwner`, `userAdmin*`, `*AnyDatabase`, `clusterAdmin`, `hostManager`, on any database) | assert: `admin.<mongodb8_cis_admin_user>` is never in the list (the role logs in with it, D24) |

- **Why it is now acceptable** (D29's four questions, answered again):
  1. **Concrete fix:** CIS 3.4 description *"eliminating unneeded roles"*; 3.5 remediation `revokeRolesFromUser`.
     The site names the object, so "necessary/legitimate" stays a person's decision; the role only executes it.
  2. **Simple input:** a list of names copied from the report, same shape as 3.1. The rejected 3.4 design
     (`revokePrivilegesFromRole`: role + resource + actions) stays rejected; dropping a whole role is the simple form.
  3. **Safe if wrong:** limited to what is listed. 3.5 can't remove the role's own admin, which (with 2.1) holds `root`,
     so the server always keeps an admin. Dropping a role or revoking admin roles is undone by hand (`createRole`,
     `grantRolesToUser`); no data is touched.
  4. **Worth it:** drift on long-lived servers (old roles, extra admins).
- **Idempotent:** a dropped role or a revoked account no longer matches the next read, so the PATCH is skipped
  (`changed=0`). `mongodb_shell` always reports `changed`, so it only runs when there is something to do.
- **Read-only runs:** the PATCH steps are tagged `patch`; use `--tags audit --skip-tags patch` (README).
- **Tested:** expressions with fake `mongosh` output (2026-10-06): only listed objects; a listed account without 3.5 roles and
  an unlisted role skipped; the guard stops the run when the role's admin is listed. **Not yet on a real mongod.**

## D31. Wider scope, fresh installs first: 2.3, 3.2 and 4.5 (revises D29, D30, "Not adopted")

- **Decision (2026-10-10, user direction):** the role targets a **fresh** server first and should be easy for any
  organization; 2.3, 3.2 and 4.5 can now change the host. Existing servers are handled where it is simple, otherwise
  reported. 1.1 and 3.3 stay report only (3.3: the RPM already runs mongod as `mongod`, user decision).

  | Rule | Setting | Does | Kept out (not in CIS, or over-engineering) |
  |------|---------|------|------------|
  | 2.3 | `mongodb8_cis_rule_2_3: false` (WARNING, like other restart rules), `mongodb8_cis_cluster_file: ""` | on a replica set / shard / config server member: `security.clusterAuthMode: x509` + `net.tls.clusterFile` (empty = the 4.3 server cert); standalone → N/A | keyFile mode (CIS: *"development only"*); TLS settings (4.3 writes them, 2.3 checks 4.3 is on) |
  | 3.2 | `mongodb8_cis_users: []` (user, db, password, roles) | creates listed accounts that don't exist; adds listed roles an account is missing | removing roles (3.1/3.5 lists do that), changing passwords, `authenticationRestrictions` |
  | 4.5 | `mongodb8_cis_encryption: {}` (written as-is under `security:`, like 5.1's `mongodb8_cis_audit_log`) | `enableEncryption: true` + the KMIP or keyfile settings, in `section_4/cis_4.5.yml` (imported from `main.yml` before mongod's first start), only when dbPath has no data | migrating existing data (hand procedure in manual-remediation.md); key creation (the site owns the key and its backup) |

- **Why each passes D29's test now:** 1 ✅ CIS gives the setting (2.3 config lines, 3.2 *"assign the appropriate users
  to each role"*, 4.5 `enableEncryption` + key management). 2 ✅ one switch / one list / one dict, using MongoDB's own
  names. 3 ✅ nothing is removed: 2.3 only on cluster members, 3.2 only adds, 4.5 only on an empty dbPath. 4 ✅ a fresh
  server fails all three.
- **Scope note (2.3):** the CIS title says *sharded cluster*; replica set members use the same internal
  authentication and a shard is a replica set, so `sharding.clusterRole` **or** `replication.replSetName` = member.
  `mongos` is not managed. CIS Audit also lists `authenticationMechanisms: MONGODB-X509`; setting it turns off password
  login, including the role's own admin (D24), so it is not written.
- **3.2 when the list is empty:** nothing happens. Accounts not in the list are never touched.
- **Tested:** Jinja expressions with fake config/`mongosh` output, 2026-10-10. **Not yet on a VM.**
- **Revised 2026-10-10 (same day):** a first version had a 2.3 mode variable (x509/keyFile), an exact-set 3.2 dict that
  also removed roles, and 4.5 mode + 5 KMIP variables with file checks. Trimmed after user review ("easy to use,
  fresh first, don't over-engineer").

## D32. TLS certificate files: the role can copy them (4.3)

- **Decision (2026-10-10, user choice):** 4.3 keeps mongod's two settings, `certificateKeyFile` (server certificate +
  private key in one PEM) and `CAFile` (the CA certificate), now with default paths `/etc/pki/mongodb/server.pem` and
  `ca.pem`. Optional `mongodb8_cis_tls_certificate_key_src` / `_tls_ca_src`: the role copies the files from the
  control node (owner mongod, `0600`, restart on change). Empty = files already on the server (as before).
  Optional `mongodb8_cis_shell_tls_certificate_key_src`: the role's own client certificate, copied to
  `/etc/pki/mongodb/mongodb8_cis-client.pem` (owner root); replaces `mongodb8_cis_shell_tls_certificate_key_file`.
- **Why:** an organization gets certificates from its own CA; copying them was a separate lab playbook
  (`prep-tls.yml` in the test project), an extra step a non-technical team would miss. Certificate renewal becomes
  "replace the source file, rerun". Company server certificates often allow only `serverAuth`; with `requireTLS` +
  `CAFile` mongod asks every client for a certificate, so the role needs its own client certificate there.
- **Evidence:** CIS 4.3 remediation (`requireTLS`, `certificateKeyFile`, `CAFile`); MongoDB `net.tls.certificateKeyFile`
  holds certificate and key in one file on Linux (no separate key option).
- **Revised 2026-10-10 (same day, user review):** the two path variables were redundant with `_src`: removed from
  defaults. The server paths are user choices in defaults, empty by default (4.3 needs both sources and both destinations, else `NOT APPLIED`; second revision the same day), `mongodb8_cis_tls_certificate_key_file` / `_tls_ca_file`, defaulting to `/etc/pki/mongodb/` + the `_src` file's name (revised the same day: user wanted each path free; `vars/` can't be overridden from group_vars); 4.3 applies when both
  `_src` are set, otherwise reports `NOT APPLIED`. Files placed on the server another way are no longer an option.
- **Not added:** `CRLFile`, `certificateKeyFilePassword` (not in CIS; add when a site needs them).
- **Tested:** lint and syntax of the role rebuilt from `build-guide.md`; not yet on a VM.

## D33. Every CIS rule on by default (revises D6/D20 defaults, user direction 2026-10-10)

- **Decision:** like `chrome_cis` D5: risky rules (2.1, 2.2, 2.3, 4.3, 4.4, 6.1) default `true`, and so do the Manual
  fixes with a fixed CIS value (6.2 limits, 6.3 JavaScript off, 7.1, 7.2). The organization sets `false` for what it
  can't accept; `group_vars` = its exception list. Values only the site knows (admin, port, certificates, lists, filter,
  encryption key) stay empty: the rule reports `NOT APPLIED` and skips, never fails the run. Level 2 stays opt-in (D18).
- **Why:** the user applies hardening this way ("CIS said it, so it's on; we turn it off only when it limits what we
  need") and wants one convention across roles.
- **Safety kept:** 2.2 waits for a user; 2.3 and 4.3 wait for certificate files (`mongodb8_cis_tls_ready`); 4.4 waits for TLS (a first draft also required OS FIPS mode;
  removed: the 2026-10-06 lab test shows mongod activating FIPS on RHEL 8/9/10 without it); 4.5 only on an empty dbPath.
- **Tested:** lint, syntax, and the `mongodb8_cis_tls_ready` expression locally; not yet on a VM.

## D34. 4.5 back to report only (revises D31 for 4.5)

- **Decision (2026-10-10, user):** 4.5 only reports; the organization enables encryption at rest by hand
  (manual-remediation.md 4.5). `mongodb8_cis_encryption` removed.
- **Why:** automating it needed the rule to run before mongod's first start, out of the section order
  (`main.yml` calling 4.5 early, or a CIS step inside `install.yml`). The user preferred the plain Lockdown layout; CIS
  marks 4.5 Manual, so report only is still compliant. Key management (KMIP/keyfile) is the organization's design anyway.

## Not adopted

- **Lockdown's Goss audit framework** (`setup_audit`, `run_audit`, `audit_only`, L3): it adds an extra tool and binary to maintain. The AUDIT steps plus `--check --diff` already give a read-only compliance view.
- **Per-OS vars files** (`vars/RedHat.yml`, `vars/Rocky.yml`, ...): there is no per-OS difference (D1).
- **Section 3 fixes for 3.2–3.3** (Revised 2026-10-10: 3.2 role assignment adopted as an opt-in, D31; 3.3 stays report only) (declared users/roles via `mongodb_user`/`mongodb_role`; 3.4/3.5 name lists adopted in D30): report only (D28,
  user decision 2026-10-04/05). A 3.2 declared-accounts list was built on the rebuild branch and reverted as too complex.
- **3.3 fix** (switch a root-run mongod to a service account): the RPM already passes; the FAIL case means re-owning
  unknown files, and one miss stops mongod. Hand steps in [manual-remediation.md](manual-remediation.md).
- **3.4 fix** (`revokePrivilegesFromRole` from a site list): role + db + resource + actions per entry; only the app
  team knows what is unneeded. Hand steps in [manual-remediation.md](manual-remediation.md).
- ~~**2.3 fix**~~ **Revised 2026-10-10: adopted as an opt-in (D31).**
- **4.5 fix on existing data** (Revised 2026-10-10: a new install is now an opt-in, D31): MongoDB *"cannot encrypt existing data"*; keyfile *"does not meet most regulatory
  key management guidelines"*; KMIP is external infrastructure; a lost key loses all data. Evidence and future options:
  [manual-remediation.md](manual-remediation.md) 4.5.
