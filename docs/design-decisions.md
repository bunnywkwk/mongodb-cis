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

- **Decision:** `dbPath`, the log path and the service user are read from `mongod.conf` and the systemd unit. The package defaults (`/var/lib/mongo`, `/var/log/mongodb/mongod.log`, `mongod`) are used only as fallbacks in `vars/main.yml`.
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
- **Status:** to be tested under SELinux enforcing on the VMs before the PATCH is written.

## D12. Install is opt-in

- **Decision:** `mongodb_cis_install: false`. When `true`, the role creates the S1 repo with `yum_repository` and installs `mongodb-org`.
- **Why:** the role's job is hardening. Installing must be a deliberate choice, and it is needed now only because the test VMs are empty.
- **Evidence:** S1 repo definition and install command. CLAUDE.md "Application roles: Install is opt-in".

## D13. Talking to the database (rules 2.1, 3.x): `mongosh`, not pymongo

- **Decision:** database reads use `ansible.builtin.command: mongosh --quiet --eval '<JSON-returning query>'` with `changed_when: false` and `check_mode: false`. The output is parsed with `from_json`. Credentials are variables, and every task that uses them has `no_log: true`. The optional admin bootstrap for 2.1 also uses `mongosh`, with `changed_when` computed from its output.
- **Why:**
  - `mongosh` is installed by `mongodb-org` on every RHEL version (S1). pymongo is **missing on EL8**, 3.10 on EL9 and 4.8 on EL10 (platform-notes, S8), so the `community.mongodb` modules would need a per-OS driver install. That is exactly the kind of variance to avoid.
  - `mongosh` also matches how CIS writes the audits (S6 uses shell and config-file checks).
- **Revised:** earlier this doc preferred `community.mongodb`. It changed after checking EPEL 8/9 (S8).

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
  - `level_1: false`, `level_2: true` would run L2 rules without their base, which CIS does not define. `prelim.yml` warns about it.

**Why it differs from Lockdown:**
- We don't use Goss (see "Not adopted"). A level variable that changes nothing would mislead users; the exact misunderstanding L4 shows is easy to have.
- Default **Level 1 only:** S5 defines Level 1 as *"practical and prudent"* and not limiting utility, and Level 2 as for environments *"where security is paramount"*. Lockdown enables both by default; we choose the CIS baseline and make Level 2 an explicit choice. Confirm with the mentor.

## Not adopted

- **Lockdown's Goss audit framework** (`setup_audit`, `run_audit`, `audit_only`, L3): it adds an extra tool and binary to maintain. The AUDIT steps plus `--check --diff` already give a read-only compliance view.
- **Per-OS vars files** (`vars/RedHat.yml`, `vars/Rocky.yml`, ...): there is no per-OS difference (D1).
