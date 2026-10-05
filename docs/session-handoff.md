# Session handoff — `mongodb8_cis` rebuild (state on 2026-10-02)

Context for an AI assistant (or person) continuing this work on another device. Read this first, then
`docs/build-guide.md`. The parent workspace also has a `CLAUDE.md` with the full project rules; the essentials are below.

## 1. What this project is

- An Ansible role **`mongodb8_cis`** that hardens **MongoDB 8.0 Enterprise** (standalone `mongod`) on **RHEL 8, 9, 10**
  to the **CIS MongoDB 8 Benchmark v2.0.0** (23 recommendations: 13 Level 1, 10 Level 2).
- Repo: `github.com/bunnywkwk/mongodb-cis` (public). **Branch `mongodb8-cis-rebuild`** = the user is **retyping the
  role by hand, batch by batch**, to learn it. Branch `main` = the finished, VM-tested previous version (reference).
- Structure follows **ansible-lockdown** (`RHEL9-CIS`): `tasks/section_<n>/main.yml` + one file per rule `cis_<id>.yml`,
  AUDIT → PATCH pattern, `discovered_*` names, tags on the rule block.
- Test project (separate repo): `github.com/bunnywkwk/mongodb8-cis-test` (inventory, `group_vars`, playbooks).

## 2. How the user wants to work (important)

- **The user types `tasks/` and `handlers/` by hand.** The assistant writes everything else (`defaults/`, `vars/`,
  `meta/`, lint config, docs, README) and generates task code for the user to type, with a short **what/why**.
- The code to type is in **`docs/build-guide.md`** (batches 1–10, code copied from tested files). The chat is for Q&A.
- **When the user pastes a snippet, answer concisely:** just the syntax, a tiny example, no whole-setup essays.
- The user is learning; explain in plain words, with tables; back claims with sources (CIS PDF text, official docs).
- After the user types a batch: run `yamllint . && ansible-lint --offline` **inside the role folder**, compare with the
  tested version, list fixes. The user may ask the assistant to apply fixes.
- Don't commit or push unless asked. Never run the role against real hosts without the user's OK.

## 3. Key decisions (the "why")

| Topic | Decision |
|-------|----------|
| Source of truth | CIS PDF + certification spreadsheet (kept **locally only**; never commit them: CIS terms forbid hosting) |
| Rule = | one task block per CIS rule: name `"<ID> \| PATCH/AUDIT \| <exact CIS title>"`, `when:` toggle + level + AUDIT check |
| AUDIT → PATCH | read state first (prelim parses `/etc/mongod.conf` into `discovered_mongod_conf`), change only on drift → rerun `changed=0` |
| Config edits | `copy` with `discovered_mongod_conf \| combine(<rule keys>, recursive=true) \| to_nice_yaml`, then `set_fact` to update the in-memory config; `notify: Restart mongod` |
| Handlers | `listen: Restart mongod` → restart + `wait_for` the port (60 s) |
| Levels | `mongodb8_cis_level_1: true`, `_level_2: false`; level vars really gate rules (unlike Lockdown) |
| Risky rules | their own toggle defaults to `false`: 2.1, 2.2, 4.3, 4.4, 6.1 (no global `disruption_high`) |
| `\| bool` | **not used** on switches (Lockdown style). Users write unquoted `true/false`; on the CLI only JSON `-e '{"x": true}'` |
| Manual rules | report by default; **if CIS's fix is a concrete setting, add an optional site variable** (empty/false = report only). Done: 5.2 `mongodb8_cis_audit_filter`, 6.3 `mongodb8_cis_javascript_needed`, 7.1 `mongodb8_cis_fix_key_file_permissions`, 7.2 `mongodb8_cis_fix_db_path_permissions`. Candidates later: 6.2 (limits drop-in), 4.5 (needs KMIP) |
| Version | only MongoDB **8.0**; prelim stops on other versions (a CIS 7 benchmark exists separately) |
| DB access (sections 2/3) | pymongo in a venv on each server (`/opt/mongodb8_cis/venv`, python3.12, `pymongo>=4.9,<5`, `tasks/pymongo.yml`); `mongodb_user` for 2.1, `mongodb_info` for 2.2/3.1/3.2/3.5 (roles come as a `str()` → `from_yaml`), `mongodb_shell` for 3.4. **Accepted gap:** `mongodb_info` doesn't list users of databases that hold no data |
| Ansible | one ansible-core **2.16.x** env (`~/.ansible-2.16.1-env`) for RHEL 8/9/10; `min_ansible_version: "2.16.1"`. Collections: `community.mongodb` 1.8.x, `community.general` 11.x |
| 5.1 audit | `auditLog.destination` default `syslog` (CIS lists syslog/console/file); `file` adds `format` + `path` |
| `quiet` on asserts | removed by the user (cosmetic only) |

## 4. Rebuild progress

| Batch | Content | Status |
|-------|---------|--------|
| 1 | `tasks/main.yml`, prelim checks | ✅ done |
| 2 | rest of `prelim.yml` | ✅ done (fixed by the assistant: install hook moved after the version check) |
| 3 | `install.yml`, `handlers/main.yml` | ✅ done |
| 4 | Section 5 (5.1–5.4), incl. new 5.2 filter | ✅ done, lint clean (fixed by the assistant) |
| 5 | Section 1 + Section 7 (1.1, 7.1, 7.2) | ✅ done, lint clean (typos fixed by the assistant 2026-10-03: 7.1 `results`/`exists`/`item['item']`) |
| 6 | Section 6 | 🟡 **in progress** (2026-10-03): user typing `cis_6.1.yml`/`cis_6.2.yml`. 6.2 was upgraded (site variable + drop-in) → retype it from the guide; retype the handler (`daemon_reload: true`) |
| 7 | `pymongo.yml`, sections 2+3 block, prelim login check, Section 3 | ⏳ |
| 8 | Section 2 | ⏳ |
| 9 | Section 4 + prelim `net.ssl` check | ⏳ |
| 10 | SELinux extra | ⏳ |

**Not yet tested on VMs in this branch:** batches 2–5. Suggested test after batch 5 on a fresh VM:
`mongodb8_cis_install: true` → install + 5.1 adds `auditLog`, one restart; rerun `changed=0`; 1.1 REVIEW, 7.1 NOT APPLICABLE,
7.2 FAIL 0755 (fix with `mongodb8_cis_fix_db_path_permissions: true`).

## 5. Files changed today by the assistant (uncommitted on this branch)

- `defaults/main.yml`: regrouped. Each rule's toggle is followed by its settings (`# ↳` one-liners + examples). Added `mongodb8_cis_audit_filter` and `mongodb8_cis_fix_key_file_permissions`.
- `vars/main.yml`: pymongo values (`mongodb8_cis_pymongo_venv`, `_pymongo_spec`, `_python_bin`, `_python_packages`).
- `.yamllint`, `.ansible-lint`: restored.
- `docs/build-guide.md` (all batches), `docs/code-walkthrough.md` (lint + main + prelim, with sources), `docs/section5.md` (Section 5 summary), this file.
- The user deleted the old `docs/` on this branch. `main` still has `docs/cis-requirements.md` (23-rule checklist with "Verify on the VM" commands; restore with `git checkout main -- docs/cis-requirements.md`; its 2.x/3.x rows need updating to the pymongo modules).
- 2026-10-03: new `docs/design-decisions.md` for the rebuild (same D-numbers as `main`, D0 + D13a new, D11 has the port-range evidence), `docs/reading-config.md` (how `[...]`, `default`, `combine`, `set_fact` work, tested), `docs/simplicity-review.md` (why every line is there).

## 6. Environment

- Workstation: AlmaLinux 10 (never harden it). Proxmox VMs: rhel8 `192.168.20.50`, rhel9 `192.168.20.30`, rhel10 `192.168.20.40`; user `frqadmin`, key `~/.ssh/id_ed25519_cis` (inventory needs `ansible_ssh_private_key_file`); CPU type `host`; `base` snapshots (check they really have no MongoDB: `rpm -q mongodb-enterprise-server`).
- Lint: `cd mongodb_cis && yamllint . && ansible-lint --offline`. Run **inside** the role, or yamllint ignores `.yamllint`.

## 7. Open issues to remind the user about

- `mongodb-cis` is public and the **old commit 7543959 still serves the CIS PDF** by direct URL → delete and recreate the GitHub repo, then push the clean history.
- `mongodb8-cis-test` (public) contains **private keys** (`pki/*.key`) and a plaintext DB password → regenerate certs, change the password, purge `pki/` from history, use Ansible Vault, and gitignore `pki/` and `collections/`.
- In the KVM lab the inventory names and IPs were swapped, so **RHEL 8 was never actually tested**. Verify with `ansible mongodb -m setup -a 'filter=ansible_distribution*'`.
- Offered but not done: a README section with a kickstart "new build" `group_vars` preset (all risky rules on + site values).

## 8. Pick up here (end of 2026-10-03)

- Batch 6 open fixes for the user: `tasks/main.yml` section 6 import uses `when: mongodb8_cis_section7` (should be `section6`);
  `cis_6.1.yml` missing final newline; `cis_6.2.yml` → retype from the guide (new version); handler `daemon_reload: true`.
- Offered, not done: change the `mongodb8_cis_port` example 47017 (inside the ephemeral range 32768–60999) to e.g. 27117 + note in D11;
  add the `| int` findings (`"27017" != 27017` is True) to `reading-config.md`; catch up `code-walkthrough.md` for batches 2–5;
  RHEL 9 port/data labels not captured in `test-results.md`.
- CLAUDE.md says the workstation is AlmaLinux 10, but it reports Fedora 44 (`selinux-policy-targeted-44.9-1.fc44`). Ask the user.
- Today's lab VMs were reached on 192.168.122.x (libvirt), hosts `rhel8-pc`, `rhel9-pc`, `rhel10-pc`.
- 2026-10-04: Section 3 stays report-only except 3.1: new `mongodb8_cis_revoke_admin_roles: []` (D20) + PATCH task in
  the build guide (`mongodb_shell` `revokeRolesFromUser`; rendering tested with fake data, **not yet on a real mongod**).
  3.2 declared-accounts option was built and reverted (too complicated); 3.2 stays report-only. `check_mode: false` on `mongodb_info` reads is redundant (module supports check
  mode); offered to drop it from the guide, not decided.
- 2026-10-05: 3.3 and 3.4 stay report-only ("Not adopted" rows); new `docs/manual-remediation.md` (Section 3 hand fixes),
  to be linked from the README. 3.1 revoke list kept.
- 2026-10-05 (end): batches 7–9 typed by the user; the assistant made every typed file match the tested guide (user's
  typos and bugs: levl_2, .key(), ['user'], transform_output, attributes=, monogdb8/mainPID, 3.4 indent, 2.2 assert
  inside vars, main.yml imports, pymongo venv path, handler daemon_reload, prelim `lenght`). `quiet: true` left out
  (user's choice); SELinux import left out (batch 10 not typed). yamllint + ansible-lint (production) clean,
  `--syntax-check` on 2.16 OK. **Not yet run on a VM.** 4.5 stays report-only: evidence + future options in
  manual-remediation.md. `docs/cis-requirements.md` restored from main and updated (compliance check section, full-
  benchmark group_vars example). Next: user pushes this branch, points mongodb8-cis-test's requirements.yml at it,
  runs the acceptance test (cis-requirements.md "How to check").
- 2026-10-05: batch 10 (SELinux extra) written by the assistant on request: `tasks/selinux.yml` + import in
  `tasks/main.yml` (now identical to the guide's appendix). All 10 batches done; lint + syntax-check clean; VM test next.
