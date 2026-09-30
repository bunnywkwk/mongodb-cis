# Design decisions — `mongodb_cis`

Every decision below has three parts: **Decision**, **Why**, and **Evidence**. Source IDs (S1–S9) are defined in
[platform-notes.md](platform-notes.md#sources). Four more sources are used here:

| # | Source |
|---|--------|
| L1 | ansible-lockdown `RHEL9-CIS` (branch `devel`), `tasks/section_5/cis_5.4.2.x.yml` rule 5.4.2.2: AUDIT then PATCH in one block |
| L2 | ansible-lockdown `RHEL9-CIS`, `tasks/section_1/cis_1.1.1.x.yml` rule 1.1.1.1: declarative `lineinfile`, no AUDIT step |
| L4 | ansible-lockdown `RHEL9-CIS` (devel, cloned 2026-09-29): `grep -rn rhel9cis_level_ tasks/` finds nothing. The level vars appear only in `templates/lockdown_audit.yml.j2` and `templates/etc/ansible/compliance_facts.j2`. README, *Matching a security Level for CIS*: *"This is managed using tags"*. Tag count: 237 rules tagged level1 only, 57 level2 only, 8 both |
| L3 | ansible-lockdown `RHEL9-CIS`, `defaults/main/main.yml` (`rhel9cis_disruption_high: false`, `rhel9cis_section1`, `rhel9cis_level_1`) and `defaults/main/audit.yml` (`setup_audit`, `run_audit`, `audit_only`) |

Scope: **MongoDB Community 8.0, standalone `mongod`, RHEL 8, 9 and 10**, CIS MongoDB 8 Benchmark v2.0.0 (S5, S6).
Chosen with the user on 2026-09-29 (RHEL 8 added the same day).

> Recovery note: this file and `platform-notes.md` were rebuilt on 2026-09-29 after an accidental
> `ansible-galaxy role init --force` deleted `mongodb_cis/`. Content is the last version from the working session.

---

## D1. One role for RHEL 8, 9 and 10, no OS branches

- **Decision:** a single `mongodb_cis` role. The only OS-dependent value is the repo `baseurl`, built from `ansible_facts['distribution_major_version']`.
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

## D6. Disruptive rules behind `mongodb_cis_disruption_high: false`

- **Decision:** rules that can lock out users or break clients also require `mongodb_cis_disruption_high: true`:
  - 2.1: enable authorization.
  - 2.2: disable localhost bypass.
  - 4.3: require TLS.
  - 6.1: change the port.
- **Why:** it is one clear switch, and it is Lockdown's own convention, not something we invented. A fresh install fails all four rules (see platform-notes "Fresh-install state"). Enabling them without an admin user, certificates or updated client connection strings breaks access.
- **Evidence:** L3, `rhel9cis_disruption_high: false` with the comment *"Run tests that are considered higher risk and could have a system impact if not properly tested"*.

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

- **Decision:** `dbPath`, the log path and the service user are read from `mongod.conf` and the systemd unit. The user comes from prelim's read-only `systemd_service` call (`status.User`) → `mongodb_cis_service_user`. The package defaults (`/var/lib/mongo`, `/var/log/mongodb/mongod.log`, `mongod`) are used only as fallbacks in `vars/main.yml`.
- **Why:** the MongoDB install page and the RHEL package disagree. The docs say user `mongodb` and `/var/lib/mongodb`; the RPM uses `mongod` and `/var/lib/mongo`. Sites may also move `dbPath`.
- **Evidence:** S1 vs S3 (see platform-notes, "The docs and the package disagree").

## D9. Community edition: rules that cannot be applied

| Rule | Why not applicable | Evidence | Role behaviour |
|------|--------------------|----------|----------------|
| 4.4 FIPS mode | *"FIPS mode is only available with MongoDB Enterprise edition."* | S4 FIPS page | AUDIT: report "not available on Community" |
| 4.5 Encryption at rest | *"Available in MongoDB Enterprise only."* | S4 encryption page; S5 4.5 Additional Information | AUDIT: report |
| 5.1 System activity audited | *"MongoDB Enterprise includes an auditing facility"*; Community is not listed | S4 auditing page | AUDIT: report as a documented gap |
| 5.2 Audit filters | S5: *"This check is only for Enterprise editions."* | S5 5.2 | AUDIT: report |

These are reported, not hidden, so an auditor sees the gap and the reason.

## D10. Rule 5.3 on Community — deviation to confirm with the mentor

- **Decision:** apply 5.3 (`systemLog.quiet: false`) on Community.
- **Why:** S5 5.3 says *"This check is only for Enterprise editions"*, but `systemLog.quiet` exists in Community with default `false` (S4 configuration options). The setting is harmless, and a fresh install is already compliant (S3), so the PATCH only runs if someone set `quiet: true`.
- **Status:** a deviation from the benchmark text. The user can disable it with `mongodb_cis_rule_5_3: false`.

## D11. Rule 6.1 (non-default port) conflicts with the MongoDB SELinux policy

- **Decision:**
  - The rule is disruptive (D6).
  - The port is a site value, `mongodb_cis_port`, with no default invented by the role.
  - The rule fails with a clear message if enabled without a port.
- **Why:** CIS gives no port number (*"$Orginasation Defined port"*, S5 6.1). The MongoDB SELinux policy only supports default ports (S1: *"you cannot use the MongoDB supplied SELinux policy"* with custom ports). A port change therefore also needs SELinux port labelling and a firewall change.
- **SELinux detail (platform-notes "SELinux", D17):**
  - Without MongoDB's module, `mongod` is `unconfined_service_t`, so SELinux does not block the new port.
  - With the module, the new port must first be labelled `mongod_port_t`, or `mongod` will not start.
  - Firewalld needs the new port opened either way.
- **Status:** PATCH written 2026-09-30 (assert 1024–65535 and ≠ 27017; `net.port` merged; handler waits on the new port). With the default SELinux state (`mongod` unconfined) no port label is needed; the optional SELinux extra (D17) will label it. Firewall stays site policy (bindIp is `127.0.0.1` by default, so no firewall change is needed). **VM test under enforcing still pending.**

## D12. Install is opt-in

- **Decision:** `mongodb_cis_install: false`. When `true`, the role creates the S1 repo with `yum_repository` and installs `mongodb-org`.
- **Why:** the role's job is hardening. Installing must be a deliberate choice, and it is needed now only because the test VMs are empty.
- **Evidence:** S1 repo definition and install command. CLAUDE.md "Application roles: Install is opt-in".

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

## D14. What the CIS spreadsheet (S6) tells us about AUDIT steps

- **Most automated audits are config-file reads.** S6 "Audit Procedure": 2.1, 2.2, 4.2, 4.3, 5.1, 5.3, 5.4 and 6.1 are all `cat /etc/mongod.conf | grep <key>`. Our shared AUDIT (D7: `slurp` + `from_yaml` → `discovered_mongod_conf`) checks the same keys, structurally. That is more reliable than `grep`, which can't tell a commented-out key or a key in the wrong section.
- **Legacy `ssl` vs `tls` names.** S6 4.2 greps under `ssl`, while 4.1/4.3 use `net.tls`. MongoDB 4.2 renamed `net.ssl.*` to `net.tls.*` and kept `ssl` as deprecated aliases. The role writes only `net.tls.*`. The AUDIT for 4.1/4.2/4.3 flags a legacy `net.ssl` block as drift, so it can't hide an old setting.
- **Runtime, not file, for 6.2.** S6 6.2 audits `/proc/<mongod PID>/limits`, not the unit file. The AUDIT reads `MainPID` (`systemctl show mongod -p MainPID`) and then `/proc/<pid>/limits`, so a systemd drop-in that lowers limits is caught.
- **4.1 and 4.2 check the same key.** Both require `TLS1_0,TLS1_1` in `net.tls.disabledProtocols` (S5/S6), at L2 and L1 respectively. One PATCH task satisfies both; each rule keeps its own ID, toggle and block, so skipping one does not skip the other.
- **Defaults per S6 "Default Value":** authorization disabled (2.1), `enableLocalhostAuthBypass` `true` (2.2), `javascriptEnabled` enabled (6.3). This agrees with the fresh-install state in platform-notes.

## D15. RHEL 8: run it from ansible-core 2.16.1+; the role stays 2.16-compatible

- **Decision:**
  - RHEL 8 targets are run from the separate ansible-core `>=2.16.1,<2.17` environment; RHEL 9/10 from the main 2.20 environment (see [../../docs/control-node-setup.md](../../docs/control-node-setup.md)).
  - The role declares `min_ansible_version: "2.16.1"` in `meta/main.yml` and uses only 2.16 features.
  - `prelim.yml` asserts:
    1. `ansible_version.full is version('2.16.1', '>=')`: 2.16.0 and older are rejected.
    2. When `ansible_facts['distribution_major_version'] == '8'`: `ansible_version.full is version('2.17', '<')`. The failure message points to the control-node doc.
- **Why:**
  - ansible-core 2.17+ doesn't support Python 3.6 on targets (S7).
  - A newer Python on RHEL 8 doesn't help, because the `dnf` bindings are 3.6-only (control-node doc A2, A3).
  - Every 2.16 patch release supports Python 3.6 targets. The floor is 2.16.1, not 2.16.0 (user choice, 2026-09-29). It matches Lockdown exactly: `ansible-lockdown/RHEL9-CIS` `meta/main.yml` has `min_ansible_version: 2.16.1`. The el8 env installs the newest patch anyway (control-node doc).
  - Assert 2 catches the case where someone sets python3.12 on RHEL 8 and runs 2.17+. Facts would be gathered fine, but `dnf` would fail halfway through the run.
- **Revised:** an earlier version of this decision said "python3.12 + `ansible_python_interpreter` + assert Python ≥ 3.9". That was dropped because the `dnf` bindings problem makes it fail.
- **Open risk:** confirm on the RHEL 8 VM from the 2.16 env: `ansible.builtin.dnf` works, and the `mongod.conf` write works under SELinux enforcing.

## D16. Test plan follows the CIS certification grid (S6 columns E–G)

S6 has three result columns per rule. Our VM test runs use the same three states on RHEL 8, 9 and 10:

| State | Expected | How we produce it |
|-------|----------|-------------------|
| Default installation | mix of Pass/Fail (see platform-notes "Fresh-install state") | fresh VM + `mongodb_cis_install: true`, run with `--check` |
| Non-hardened | all Fail | edit `mongod.conf` to violate every automated rule, run `--check` → every PATCH reports "would change" |
| Remediated/Hardened | all Pass | normal run, then a second run → `changed=0` |

Column H ("exceptions") is where Community gaps (D9) and deviations (D10) are recorded.

## D17. SELinux policy for `mongod`: optional extra, off by default

- **Decision:**
  - `mongodb_cis_selinux_policy: false` by default. When set to `true`:
    1. Install MongoDB's official policy module (S9). It is compiled once on the control node, and only the `.pp` file is copied to the target, so there are no compilers on servers.
    2. Label the non-default `dbPath`, log path and port that the role knows about (`semanage fcontext` + `restorecon`, `semanage port` for rule 6.1).
    3. Verify that `mongod` then runs as `mongod_t`.
  - It is **not a CIS rule**:
    - It has no CIS ID or level tags. It is named and tagged `selinux_policy`.
    - It is documented in the README as "extra hardening beyond CIS MongoDB".
    - It is built **after** all CIS rules are done.
- **Why optional:**
  - The scope is the CIS MongoDB 8 benchmark (S5), which has no SELinux recommendation. CLAUDE.md says never present an extra control as CIS.
  - Confinement is still real defense in depth: without it, `mongod` runs as `unconfined_service_t` (platform-notes "SELinux"). So users who want it can switch it on.
- **Why off by default:**
  - MongoDB supports its policy only with default paths and ports (S1). Every non-default value must be labelled, or `mongod` fails to start.
  - The policy is maintained by MongoDB, not Red Hat, so each MongoDB or RHEL update needs a retest.
- **Limits (documented, not hidden):** the role labels only the paths and ports it manages or reads from `mongod.conf`. Anything else a site adds (backup dirs, extra ports) is the site's job.
- **Revised:** twice on 2026-09-29. First it was proposed as an opt-in with a reference to the RHEL OS benchmark; then removed as out of scope; now re-added as an optional non-CIS extra, without any OS-benchmark reference (user decision).

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
  - `mongodb_cis_level_1: true` and `mongodb_cis_level_2: false`.
  - Every Level 1 rule has `when: mongodb_cis_level_1`; every Level 2 rule has `when: mongodb_cis_level_2`. The per-rule toggle (`mongodb_cis_rule_<id>`) still applies on top.
- **Tags kept as well,** Lockdown-style: `level1` or `level2`. The MongoDB benchmark has no server/workstation split, so there is no `-server` suffix. A full Level 2 run by tags is `--tags level1,level2`.
- **Profiles expressed with variables:**
  - Level 1 profile (default): `level_1: true`, `level_2: false`.
  - Level 2 profile: `level_1: true`, `level_2: true`.
  - `level_1: false`, `level_2: true` runs L2 rules without their base, which CIS does not define. The README documents it; there is no runtime warning (removed as unnecessary, user decision 2026-09-29).

**Why it differs from Lockdown:**
- We don't use Goss (see "Not adopted"). A level variable that changes nothing would mislead users; the exact misunderstanding L4 shows is easy to have.
- Default **Level 1 only:** S5 defines Level 1 as *"practical and prudent"* and not limiting utility, and Level 2 as for environments *"where security is paramount"*. Lockdown enables both by default; we choose the CIS baseline and make Level 2 an explicit choice.
- **Confirmed** by the user on 2026-09-29, after comparing it with pure Lockdown (tags only, no level vars): keep vars that gate rules, plus tags. Everything else matches Lockdown: section switches, rule toggles, level tags, `disruption_high`, AUDIT → PATCH.

## D19. Every user switch in a condition gets `| bool`

**In plain words:**
- When you type `-e mongodb_cis_install=false` on the command line, Ansible receives the **text** `"false"`, not the value *false*. AWX/Tower surveys do the same.
- Text that isn't empty counts as "yes", so without `| bool`, "false" can mean **true**.
- `| bool` turns the text back into a real yes/no: `"true"`/`"yes"`/`true` → true, `"false"`/`"no"`/`false` → false.
- It is **not** about `--check`; it's about where the value comes from. Values from `defaults/` or `group_vars` YAML are already real booleans, and `| bool` leaves them unchanged. So it's never harmful, and it protects every way a user can set a switch.

**Evidence 1: the real error** (2026-09-30, RHEL 10 install test, ansible-core 2.20.7, T-M4):
```
[ERROR]: Task failed: Conditional result (True) was derived from value of type 'str' at "<CLI option '-e'>". Conditionals must have a boolean result.
Origin: .../mongodb_cis/tasks/prelim.yml:39:9
39   when: mongodb_cis_install
```

**Evidence 2: the silent danger on 2.16.** A debug-only playbook on localhost, `when: mongodb_cis_install` vs `when: mongodb_cis_install | bool`:

| ansible-core | `-e mongodb_cis_install=` | Without `\| bool` | With `\| bool` |
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

**Decision:** every user-facing switch used in a `when:` (install, levels, sections, rule toggles, `disruption_high`) is written `<var> | bool`. Internal facts the role sets itself (`discovered_*`) don't need it.

**Why deviate from Lockdown here:**
- This role is run by **two** ansible-core generations (2.16 for RHEL 8, 2.20 for RHEL 9/10), and they fail in opposite ways (silent wrong run vs hard stop).
- `| bool` costs nothing for correct input and makes a hardening toggle mean exactly what the user wrote, whatever the input source.

## D20. Manual rules: report by default, optional "site decision" variable where CIS gives one clear fix

- **Decision:**
  - Manual rules stay report-only by default (D4).
  - A site may declare its decision with a variable **only where the benchmark gives one clear remediation**:
    - 6.3: `mongodb_cis_javascript_needed: true` by default. `false` → PATCH `security.javascriptEnabled: false`.
    - 7.2: opt-in permission fix.
  - All other Manual rules stay report-only:
    - 3.x: which users and roles are right is site-specific.
    - 1.1: never auto-upgrade.
    - 6.2: the shipped unit is already compliant.
    - 7.1: no keyFile on standalone.
    - 4.5, 5.2: Enterprise-only.
- **Why:**
  - CIS marks a rule Manual because *"the expected state can vary depending on the environment"* (S5, Assessment Status).
  - A variable set by the site is the site's decision, not the role guessing (CLAUDE.md principle 5 still holds).
- **Status:** agreed 2026-09-30. 7.2 built (`mongodb_cis_fix_db_path_permissions`); 6.3 comes with Section 6.

## Not adopted

- **Lockdown's Goss audit framework** (`setup_audit`, `run_audit`, `audit_only`, L3): it adds an extra tool and binary to maintain. The AUDIT steps plus `--check --diff` already give a read-only compliance view.
- **Per-OS vars files** (`vars/RedHat.yml`, `vars/Rocky.yml`, ...): there is no per-OS difference (D1).
