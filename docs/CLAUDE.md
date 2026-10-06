# CIS Hardening with Ansible (Ansible Lockdown layout)

Purpose: build enterprise-grade, reusable, end-user-friendly Ansible roles that implement CIS Benchmarks for
operating systems and applications.
Default targets: **RHEL 8, 9 and 10**. Structure and conventions mirror
[ansible-lockdown](https://github.com/ansible-lockdown) (`RHEL9-CIS`, `RHEL8-CIS`, ...).

**Every target is different.** An approach that worked for one role (OS, browser, database, ...) is not evidence
that it fits the next one. Before designing a role, discover how *this* target is configured, on *these* OS
versions, and justify the chosen pattern. Never copy a previous role's structure by default.

## Core principles

1. **The benchmark is the source of truth.** Every task/rule maps to exactly one recommendation (ID + title + profile + Automated/Manual). Never invent controls. If the benchmark text is not in the repo or provided by the user, ask for it — do not write rule IDs or audit/remediation from memory.
2. **Minimal implementation.** Implement only what the recommendation requires. Prefer the idiomatic module (`ansible.builtin.lineinfile`, `sysctl`, `systemd_service`, `dnf`, `mount`, `file`, `community.general.*`, `ansible.posix.*`) over `command`/`shell`. No extra tasks, helpers, or vars beyond what a rule needs. No speculative features.
3. **Idempotent, check-mode safe, non-destructive.** Second run = zero changes. `--check --diff` must work. Anything that can lock users out, break boot, remove data or break an application (SSH, PAM, faillock, sudo, fstab, partitions, firewall, SELinux, crypto-policies, disabling services, auth/bind settings of an app) needs a toggle, a safe default, and a `# WARNING:` comment.
4. **Every rule is toggleable.** Users enable/disable per rule, per section, or per profile without editing tasks.
5. **Manual recommendations report by default; never guess at remediation.** Report only (`ansible.builtin.debug` gated by a toggle). **But when CIS's remediation is a concrete setting whose value the site decides** (e.g. MongoDB 5.2 audit filter, 6.3, 7.2), add an **optional site variable** next to the rule's toggle: empty/false by default = report only; set = the role applies it with AUDIT → PATCH. Rules whose fix is a design or people decision (users/roles, upgrades, key servers) stay report-only. (User direction 2026-10-02.)
6. **Evidence over assumption.** Paths, package names, config formats, defaults and version behaviour are verified on the target OS/app version (see Environment discovery), not recalled from memory or another role.
7. **Ask when ambiguous** (which benchmark version, profile level, site-specific values like NTP servers, banners, allowed users, app variant/version). Otherwise pick the conventional default and state it.

## Environment discovery (before implementing a role)

For each target OS version (RHEL 9, RHEL 10, ...) and each app variant/version in scope, find out and record:

- Package source and version (base repo, AppStream, EPEL, vendor repo, flatpak), package and binary names.
- How configuration is applied: config file, drop-in dir, policy JSON, DB/runtime parameter, systemd unit override, sysctl, etc.
- Where it lives, and whether it differs per OS version / variant / app version.
- OS-level effects: SELinux contexts, crypto-policies, firewalld, systemd defaults, display server (RHEL 10 is Wayland-only), FIPS.
- How to audit the effective state (not just the file on disk).

Record findings in the role's `docs/platform-notes.md` with the exact commands used. Differences between OS
versions drive version gates or lookup dicts; no difference found = no gate.

## No benchmark for the target

When CIS has no benchmark for the exact target (e.g. Chromium, which only has a Google Chrome benchmark):

- Use the closest benchmark and name it in `defaults/main.yml` as `<role>_source_benchmark` and in the README.
- Classify every recommendation, with evidence: `applies`, `na_os` (e.g. Windows-only), `na_variant` (only the benchmarked product supports it), `manual`.
- Keep the benchmark's IDs; never renumber. Document the mapping in `docs/rule-mapping.md`.
- Never claim CIS compliance for the substitute target; say "aligned with <benchmark>".

## Role layout (match Lockdown)

```
<ROLE>/                       e.g. RHEL9-CIS, chromium_cis
├── defaults/main.yml         ALL user-facing variables + rule toggles (documented)
├── vars/main.yml             internal, non-overridable values only
├── tasks/
│   ├── main.yml              orchestration: prelim -> sections -> post
│   ├── prelim.yml            facts, assertions (OS/version), package facts, service facts
│   ├── post.yml              final steps (e.g. auditd rules load, aide init, reboot flag)
│   ├── section_1.yml ...     (or cis_<n>.x.yml per benchmark section, or a data-driven task for app roles)
│   └── parse_etc_password.yml etc. only if a rule truly needs it
├── handlers/main.yml         restart sshd, reload sysctl, augenrules, etc.
├── templates/                only when a file must be rendered
├── files/                    only when static content is required
├── meta/main.yml             galaxy_info, supported platforms, min_ansible_version
├── molecule/                 default scenario (+ per-OS platforms)
├── docs/                     platform-notes.md (evidence per OS), design-decisions.md (justification + sources), rule-mapping.md (only for substitute benchmarks)
├── README.md                 user-facing usage (see below)
├── .ansible-lint, .yamllint
```

Do not add directories or files beyond this unless a rule needs them.

## Naming and variables

- Prefix all vars with the role short name: `rhel9cis_`, `rhel10cis_`, `chromium_cis_`, `mongodb8_cis_`.
- Rule toggle (OS roles): `<prefix>_rule_<id with underscores>` default `true` (or `false` for manual/risky), e.g. `rhel9cis_rule_1_1_1_1: true`.
- Section/profile switches: `<prefix>_section1`.., `<prefix>_level_1`, `<prefix>_level_2`, and `_server`/`_workstation` where the benchmark has those profiles.
- Level variables must **gate rules** (`when: <prefix>_level_1` / `_level_2` on every rule) as well as tags. Unlike Lockdown, where `rhel9cis_level_*` only feed the Goss audit and levels are chosen only by tags. Level 2 = L1 + L2 (CIS: Level 2 extends Level 1). Default: level 1 on, level 2 off unless the user decides otherwise (see `mongodb8_cis` `docs/design-decisions.md` D18).
- Parameters for a rule live next to its toggle, named for what they do (`rhel9cis_ssh_allow_users`, `rhel9cis_ssh_maxauthtries`). Defaults = CIS-recommended values.
- Group defaults.yml by benchmark section with a comment header; keep IDs in order.
- Use fully qualified collection names (FQCN) everywhere. YAML `true/false`, 2-space indent, `---` header.

## Task template

```yaml
- name: "1.1.1.1 | PATCH | Ensure cramfs kernel module is not available"
  when:
    - rhel9cis_rule_1_1_1_1
    - "'cramfs' not in rhel9cis_allowed_kernel_modules"    # extra conditions only if required
  tags:
    - level1-server
    - level1-workstation
    - automated
    - patch
    - rule_1.1.1.1
    - cramfs
  ansible.builtin.copy:
    ...
```

- Name format: `"<ID> | <PATCH|AUDIT> | <CIS title>"`. Multi-step rules add a suffix: `"<ID> | PATCH | <title> | <what this step does>"`.
- Tags: profile (`level1-server`, `level2-server`, `level1-workstation`, `level2-workstation`, or `level1`/`level2` when the benchmark has no server/workstation split), `automated`/`manual`, `patch`/`audit`, `rule_<id>`, plus a short topic tag.
- Keep tasks in benchmark order inside each section file.
- **Switches are used without `| bool`** (as Lockdown; D19 revised 2026-10-01): users set them as unquoted YAML booleans in `group_vars`, or on the CLI only as JSON (`-e '{"mongodb8_cis_rule_6_1": true}'`, never `-e x=false`). Keep `| bool` only for values read from a human-written file (e.g. `mongod.conf`: 2.2 `enableLocalhostAuthBypass`, 4.5 `enableEncryption`).
- Use `notify:` handlers rather than inline restarts. Use `validate:` on config files that support it (`sshd -t -f %s`, `visudo -cf %s`).
- No `ignore_errors: true` without a comment explaining why; prefer `failed_when`/`changed_when`. No `command`/`shell` without `changed_when` and (where possible) `check_mode` handling.
- Use `ansible_facts[...]` not injected `ansible_*` vars. Use `ansible_facts['distribution_major_version']`.
- Never hardcode secrets; sensitive values are variables, and `no_log: true` where relevant.

## AUDIT → PATCH pattern (Lockdown standard — apply it, don't over-apply it)

Purpose: read the current state first (AUDIT), change only when it is non-compliant (PATCH). A compliant host is
never touched: no file rewrite, no restart. This is also what makes the second run report zero changes.

Reference (real Lockdown code): `ansible-lockdown/RHEL9-CIS` `tasks/section_5/cis_5.4.2.x.yml` rule 5.4.2.2
(block = AUDIT `register: discovered_gid0_members` → PATCH `when: discovered_gid0_members.stdout | length > 0`), and
`tasks/section_1/cis_1.1.1.x.yml` rule 1.1.1.1 (declarative `lineinfile`, no AUDIT step).

```yaml
- name: "2.2 | PATCH | Ensure that MongoDB does not bypass authentication via the localhost exception"
  when:
    - mongodb8_cis_rule_2_2          # risky rule: its own toggle defaults to false
  tags: [level1, automated, patch, rule_2.2, authentication]
  block:
    - name: "2.2 | AUDIT | ... | Get current enableLocalhostAuthBypass"
      ...                                   # read-only
      register: discovered_localhost_bypass

    - name: "2.2 | PATCH | ... | Disable localhost auth bypass"
      when: <discovered_localhost_bypass shows drift>
      ...
      notify: Restart mongod
```

1. **Add an AUDIT step only when reading state needs its own step** (DB query, `slurp` of a structured config, `stat`, `systemctl show`, command output). AUDIT steps are read-only: `changed_when: false`, never modify anything; add `check_mode: false` to read-only commands so `--check` still sees real state (otherwise guard the PATCH with `is defined`, as Lockdown does).
2. **PATCH runs only when the AUDIT shows drift**, via `when:` on the registered/parsed result. The rule block and its AUDIT/PATCH steps share one name prefix `"<ID> | ..."`.
3. **No AUDIT step when the module is already declarative** (`file`, `lineinfile`, `sysctl`, `dnf`, `systemd_service`): the module compares and reports by itself; a separate audit would duplicate it.
4. **Manual rules are AUDIT by default**: the rule is named `"<ID> | AUDIT | ..."`, tagged `audit`, gathers and reports with `ansible.builtin.debug`. A PATCH step is allowed only behind an optional site variable (principle 5), with its own `patch` tag.
5. **Tags go on the rule block** (Lockdown): level, `automated`/`manual`, `patch` (rule changes things) or `audit` (report-only rule), `rule_<id>`, topic. Shared reads needed by many rules (parsed config, `package_facts`) live in `prelim.yml` tagged `always`. Read-only run of the whole role = `--check --diff`.
6. Registered audit results are named `discovered_<what>`.
7. **Disruptive rules** (lockout, restart that breaks clients, port changes) default their **own rule toggle to `false`** with a `# WARNING:` comment, and are switched on by name. No global `disruption_high` switch (removed 2026-10-01, D6: not in CIS, used inconsistently across Lockdown roles, hid which rules it covered).
8. Do not add Lockdown's Goss audit framework (`setup_audit`, `run_audit`, `audit_only`) unless the user asks.

## OS roles: multi-version handling

- Separate roles per major RHEL version is the Lockdown standard. Do not add `when: version == x` branches inside one role unless the user asks for a unified role.
- When porting between versions, diff against the actual benchmark of the target version; recommendation IDs and content change. Never bulk-rename IDs.
- Assert supported OS in `prelim.yml` (RHEL and compatible: Rocky, Alma, Oracle) and fail fast with a clear message.

## Application roles — one role across OS versions and app versions

Exception to the per-OS rule: application hardening is ONE role that works on the supported RHEL versions and app versions.
The mechanics below are defaults; choose the implementation pattern from discovery and state why in the README.

- **Detect, don't assume.** `prelim.yml` uses `package_facts` (or `<bin> --version` with `changed_when: false`) to set `<app>_installed` and `<app>_major`. Not installed and install toggle off -> clear skip message, end the role cleanly (no failure).
- **Install is opt-in:** `<app>_install: false`. Version selection is one variable (`<app>_channel` / `<app>_version`). Repo definition is derived from it, never duplicated per version.
- **Pick the pattern that fits the config mechanism:**
  - Single declarative artifact (e.g. browser policy JSON, one YAML/INI config): rules as data in `vars/cis_<version>.yml`, rendered by one task; skip by rule ID list.
  - Per-setting mechanisms with different modules/restarts (e.g. DB parameters + OS users + file perms + TLS): one task per rule with the standard task template and toggles.
  - Mixed is fine; don't force data-driven where rules need different modules or ordering.
- **Version gates only where the target truly differs** (renamed/removed settings, different paths): `when: <app>_major is version('X', '>=')` or `since`/`until` on a data entry. Do not gate what the app ignores harmlessly.
- **Benchmark version is a variable** (`<app>_cis_benchmark_version`) selecting `vars/cis_<version>.yml` when content differs.
- **Variants via a lookup dict** (package name, binary, config/policy path, service name) keyed by one `<app>_variant` variable.
- Audit tasks (`--tags audit`) are read-only and report installed version, config path, and drift from the expected state.

## Collaboration mode (how the user wants to work)

- **Who writes what:** the user hand-types `tasks/` and `handlers/` (to learn them). Claude generates those block by block, each block with a one-line **what** and a short **why**, and the user must be able to defend every block. Claude writes all other role files directly (`defaults/`, `vars/`, `meta/`, lint config, templates, README, docs) unless the user says otherwise.
- After the user writes a file, Claude reviews it and runs the read-only validation below, then proposes the next block.
- Keep blocks small (one task / one logical group of vars) so each can be understood on its own.
- Claude may edit `CLAUDE.md` and scratchpad files when asked.

## Workflow for each rule

1. Identify the benchmark, version, section, ID, profile, Automated/Manual.
2. Check if it is already implemented (grep the ID in `tasks/`, `defaults/`, `vars/`). Update instead of duplicating.
3. Implement the minimum: one task or data entry, one toggle, only necessary vars, handler if needed.
4. Add/update `defaults/main.yml`, tags, README variable notes if user-facing.
5. Validate (below). Report what changed, and any assumption or risk.

## Validation (run before declaring done)

```bash
yamllint .
ansible-lint                      # profile: production
ansible-playbook -i localhost, -c local <scratch playbook> --syntax-check   # playbook in scratchpad, not the repo
molecule test                     # when a scenario exists; converge twice -> idempotence
ansible-playbook <scratch playbook> --check --diff   # against a disposable RHEL VM/container only
```

- Never run hardening against the local workstation or any real host without explicit user approval. Use containers/VMs (Molecule, UBI/Rocky/Alma images, systemd-enabled). Some rules (kernel modules, mounts, sysctl, grub, GUI/browser behaviour) cannot be fully tested in containers; say so.
- If a tool is unavailable, say which check was skipped. Don't claim verified when it wasn't.

## End-user experience (README must cover)

- One-paragraph purpose, source benchmark and version, supported OSes/app versions, Ansible min version, required collections (`requirements.yml`).
- 5-line quick start (install, inventory, run playbook).
- How to choose a profile / disable a rule / override a parameter, with an example `group_vars` snippet.
- How to run audit-only vs remediate (`--tags audit`, `--tags level1`, `--skip-tags rule_5.2.4` or the role's skip list).
- Known risky rules and their toggles.
- Keep it short; no marketing text.

## Style rules for the agent

- Match surrounding code; don't reformat untouched files.
- Concise comments: only for CIS deviations, risk warnings, or non-obvious logic. Reference the CIS ID in the task name, not in prose.
- Don't create docs, changelogs, or extra files unless asked or listed above.
- Don't `git commit`/push unless asked.
- Final response: what was implemented (IDs), files touched, checks run and results, assumptions/risks. Short.

## Project state

- Each role is a standalone folder at the repo root, self-contained and reusable. Do NOT add playbooks, inventories, `ansible.cfg` or a `roles/` wrapper here; orchestration lives in a separate repository owned by the user.
- Benchmarks live in `cis-pdf/`; extract with `pdftotext -layout` into the scratchpad. Present: CIS MongoDB 8 Benchmark v2.0.0, CIS Google Chrome Benchmark v3.0.0.
- CIS certification spreadsheets live in `cis-spreadsheet/` (e.g. `CIS_MongoDB_8_Benchmark_v2.0.0-Certification.xlsx`: one row per rule with level, Automated/Manual, Audit Procedure, Remediation, Default Value). Prefer it over the PDF for per-rule data. openpyxl is not installed; read it with Python `zipfile` + `xml.etree` into the scratchpad.
- Active: **`mongodb8_cis`** (renamed from `mongodb_cis`, D23; local folder may still be `mongodb_cis/`). Its own git repo: `github.com/bunnywkwk/mongodb-cis` (**public**). MongoDB **Enterprise 8.0** (switched from Community 2026-10-01, D12/D9: auditLog, FIPS, encryption at rest are real rules), **standalone** mongod, RHEL 8/9/10, opt-in install from the Enterprise repo (`repo.mongodb.com`, `mongodb-enterprise`). **Scope = CIS MongoDB benchmark only**; OS facts only where a rule needs them (e.g. 6.1 port, SELinux label). One optional non-CIS extra: `mongodb8_cis_selinux_policy: false` (`tasks/selinux.yml`, builds MongoDB's module from `files/selinux/` on RHEL 9/10; RHEL 8 handled separately; tag `selinux_policy`).
- Status (2026-10-06, `origin/main` 85155d7): **all 23 rules implemented**, Lockdown layout `tasks/section_<n>/main.yml` + `cis_<id>.yml`; `main` reads the database with `community.mongodb.mongodb_shell` only (D13/D24, nothing installed on the server). Risky rules default `false`: 2.1, 2.2, 4.3, 4.4, 6.1. Manual rules with opt-in site variables (D20, D30): 3.1 `revoke_admin_roles`, 3.4 `drop_custom_roles`, 3.5 `revoke_superuser_roles` (guard: never the role's own admin), 5.2 `audit_filter`, 6.2 `fix_resource_limits`, 6.3 `javascript_needed`, 7.1 `fix_key_file_permissions`, 7.2 `fix_db_path_permissions`; report only: 1.1, 2.3, 3.2, 3.3, 4.5 (D29, `docs/automation-decisions.md`). 5.1 uses one block `mongodb8_cis_audit_log`; 5.3 writes `quiet: false` explicitly. **Compliance test passed 2026-10-06 on RHEL 8, 9 and 10, Level 2**: full run `changed=11`, rerun `changed=0`; per host 18 pass, 2.3 N/A, 3.2/3.5/5.2 reviewed, 4.5 accepted exception (no key management), 0 fail; plus an app-access test on RHEL 8. Record: `mongodb8-cis-test/docs/compliance-test.md` (screenshots 12–36, logs in `runs/`); summary in the role's `docs/test-results.md` and `docs/summary.md`. **Not yet tested:** the 3.1/3.4/3.5 lists with real entries; the SELinux extra on this run. Branch `mongodb8-cis-rebuild` (hand-retyped, pymongo) is parked; compare/merge decision open (`docs/session-handoff.md`).
- **Open issues:** (1) the public `mongodb-cis` repo history (commit 7543959) still contains `cis-pdf/` and `cis-spreadsheet/`; CIS terms forbid hosting them → recreate the GitHub repo or purge history; never commit CIS PDFs/spreadsheets. (2) public `mongodb8-cis-test`: `pki/*.key` (lab CA, expires 2026-10-31) and a plaintext lab password are in its history → regenerate certs, change the password, purge before real use. (3) A copy of this file is kept in the role repo as `docs/CLAUDE.md` for moving between laptops; keep both in sync.
- License note: `files/selinux/mongodb.te`/`.fc` are MongoDB's GPL-2.0 files inside an MIT role; keep their headers and mention it in the README.
- Paused: `chromium_cis` (Chrome benchmark as the substitute source). Not started.
- Local machine is an AlmaLinux 10 workstation (has google-chrome-stable, EPEL enabled). Never harden it.
- Documentation placement (index: `docs/README.md`):
  - **General** topics (workstation/control node, tooling, Ansible versions, conventions shared by all roles) → root `docs/`.
  - **Role-specific** topics (its platforms, benchmark mapping, design choices) → `<role>/docs/` (`platform-notes.md` = facts per OS, `design-decisions.md` = Decision/Why/Evidence).
  - Every doc states what, why, and evidence (source table with URLs, benchmark IDs, or commands + output). When a decision changes, update the doc and mark it "Revised" with the reason; don't silently rewrite history. Add new docs to `docs/README.md`.
  - Workstation changes are given to the user as commands to run manually (documented in `docs/`), never executed by Claude.
  - **Test evidence:** the user saves screenshots in `<role>/docs/evidence-images/`, named `NN-<os>-<scenario>-<result>.png` (e.g. `05-rhel10-install-rerun-idempotent-changed0.png`). Claude checks them (Read the image), renames them if needed, and indexes them in `<role>/docs/test-results.md`. No evidence log per build step.
  - **Avoid over-engineering for edge cases** (user feedback 2026-09-30): don't add check-mode or other special handling for rare paths (e.g. `--check` on a first install). Document the limitation in troubleshooting instead.
  - **Troubleshooting log:** every error hit while building or testing gets an entry, with the **exact** error output pasted in a code block, then cause, fix and prevent. General/tooling errors go in `docs/troubleshooting.md` (`T-Gn`); role errors in `<role>/docs/troubleshooting.md` (`T-Mn` for mongodb_cis). Add the entry in the same turn the fix is given.
  - **Code walkthrough:** each role has `<role>/docs/code-walkthrough.md`. For every file (typed by the user or written by Claude), Claude adds a **concise** section: one line of purpose plus a short `Key | Does | Why` table, linking `Dn` decisions instead of repeating them. No long prose.
- Test targets: RHEL 8.10, 9.x and 10.2 VMs on Proxmox, each with a "base" snapshot (no MongoDB) and a state with MongoDB installed by hand. Reachable: rhel10 = 192.168.20.40, rhel8 = 192.168.20.50, rhel9 = 192.168.20.30. Proxmox CPU type must be `host` (x86-64-v3 + AVX). Verify host→OS mapping with `-m setup -a filter=ansible_distribution*` (the KVM lab had them swapped). SSH key: `~/.ssh/id_ed25519_cis`. Orchestration (inventory/playbook) lives outside this repo.
- Test project `~/mongodb8-cis-test` (`github.com/bunnywkwk/mongodb8-cis-test`): `requirements.yml` picks the role branch; settings in **one file** `sysconfig/group_vars/mongodb.yml` (site values, `lab_tls_cert_src`/`lab_tls_ca_src`, Level 1+2; a **FULL BENCHMARK** block of risky rules + site decisions is commented for the first install and uncommented after `playbooks/prep-tls.yml`). Run order: `site.yml` (install, safe) → `prep-tls.yml` → uncomment block → `site.yml --force-handlers` → rerun (`changed=0`). Host checks use a shell function `m` (mongosh with TLS + login) defined before `read -rsp ... PW`.
- Control node (D15 revised 2026-10-01): **one** ansible-core `>=2.16.1,<2.17` environment runs the role against RHEL 8, 9 and 10 (2.16 supports target Python 3.6–3.12; 2.17+ breaks `dnf` on RHEL 8). `min_ansible_version: "2.16.1"`, prelim asserts `>= 2.16.1` only (the RHEL 8 `< 2.17` assert was removed). 2.16 is upstream EOL (July 2025), accepted. Collections from 2.16: `community.mongodb` `>=1.8.0,<2.0.0` (pymongo modules + `mongodb_shell` for 3.4, D13a). Rebuild (branch `mongodb8-cis-rebuild`, 2026-10-02): pymongo in a venv on each server (`/opt/mongodb8_cis/venv`, python3.12, `pymongo>=4.9,<5`, `tasks/pymongo.yml`); `mongodb_user` for 2.1, `mongodb_info` for 2.2/3.1/3.2/3.5 (roles come as a `str()` → `from_yaml`; **accepted gap: users of databases without data are not listed**), `mongodb_shell` for 3.4. Tested on a real mongod 8.0.32 with ansible-core 2.16.19 and 2.20.7. The user retypes the tasks from `mongodb_cis/docs/build-guide.md` (batches 1–10); `docs/code-walkthrough.md` is the new walkthrough with sources and `community.general` 11.x (SELinux labels), via the role's `requirements.yml`. Workstation: venv `~/.ansible-2.16.1-env` (python3.12), collections per project (`-p ./collections`, `collections_path`), role installed from GitHub via the test project's `requirements.yml` (`docs/control-node-setup.md`). The uv tools (ansible-core 2.20.7, ansible-lint, yamllint) are only for linting; `~/.venvs/ansible-el8` is the old env.
