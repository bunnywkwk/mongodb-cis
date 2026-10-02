# Build sequence — rebuilding `mongodb8_cis` by hand, testable at every step

Order to write the role from scratch, following the existing structure. Each phase ends with a **test you can run**,
so you always know the last thing you wrote works before writing the next. Rules come from
[cis-requirements.md](cis-requirements.md); explanations of each file are in [code-walkthrough.md](code-walkthrough.md).

**The order follows one idea: from safe to risky.**
1. Read-only first, then the simplest config fix.
2. Then reports.
3. Then talking to the database.
4. Then login, then TLS: the steps that can lock you out come last, when everything before them is proven.

Each phase leaves a role that runs cleanly.

## Before you start

- Control node ready: `docs/control-node-setup.md` (ansible-core 2.16 venv, collections in the test project).
- Test project `~/mongodb8-cis-test` pointing at your working copy of the role while you build (symlink or `roles_path`); switch to the GitHub install once it's pushed.
- VMs: RHEL 8, 9, 10, each with a `base` snapshot (no MongoDB). Check the mapping: `ansible mongodb -m setup -a 'filter=ansible_distribution*'`.
- After **every** file: `yamllint . && ansible-lint` inside the role.

## Phases

| # | Write | CIS rules | Test (one VM first, then the other two) | Expect |
|---|-------|-----------|------------------------------------------|--------|
| 0 | — | — | `ansible mongodb -m ping` and the `setup` filter | `pong`; RHEL 8/9/10 on the right names |
| 1 | `meta/main.yml`, `requirements.yml`, `.yamllint`, `.ansible-lint`, `.gitignore`, `defaults/main.yml` (install + level + section switches only), `vars/main.yml` (names, repo, paths) | — | `ansible-playbook playbooks/site.yml --syntax-check` | lint clean, syntax OK (nothing runs yet) |
| 2 | `tasks/main.yml` (prelim only), `tasks/prelim.yml`: facts, version/OS asserts, package detect, clean stop, read `mongod.conf`, service read, report | — | `base` snapshot, `--check` | clean stop *"not installed … nothing to harden"*, `failed=0` |
| 3 | `tasks/install.yml`, `handlers/main.yml` | — | `base`: run with `mongodb8_cis_install: true`, then again | run 1 installs + report; run 2 `changed=0` |
| 4 | `section_5/main.yml`, `cis_5.3.yml`, `cis_5.4.yml` + the `main.yml` line; rule toggles in defaults | 5.3, 5.4 | Level 2 on; on the VM set `logAppend: false`; run, run again | run 1: 5.4 `changed` + one restart; run 2 `changed=0` (**the AUDIT → PATCH pattern proven**) |
| 5 | `section_1/cis_1.1.yml`, `section_7/cis_7.1.yml`, `cis_7.2.yml`, `section_6/cis_6.2.yml`, `section_5/cis_5.2.yml` | 1.1, 7.1, 7.2, 6.2, 5.2 | normal run; then `mongodb8_cis_fix_db_path_permissions: true` | reports: 1.1 version, 7.1 N/A, **7.2 FAIL 0755**, 6.2 limits; with the decision: 7.2 fixed, rerun PASS |
| 6 | `section_5/cis_5.1.yml` + its site values | 5.1 | normal run, rerun | `auditLog: syslog` added, one restart; rerun `changed=0`; `journalctl -t mongod` shows audit events |
| 7 | `section_6/cis_6.3.yml`, `cis_6.1.yml` (+ SELinux port label) + `mongodb8_cis_port`, `_javascript_needed` | 6.3, 6.1 | `rule_6_1: true`, port e.g. 47017; `javascript_needed: false` | mongod back on 47017 with SELinux enforcing (handler waits there); JS off; rerun `changed=0`. Revert snapshot after |
| 8a | `tasks/pymongo.yml` (Python 3.12 + venv with `pymongo>=4.9,<5`), its vars, the `main.yml` line before sections 2/3 (D13a) | — | normal run, rerun | `/opt/mongodb8_cis/venv/bin/python -c 'import pymongo; print(pymongo.version)'` → 4.x (≥ 4.9); rerun `changed=0` |
| 8b | `main.yml` block for sections 2+3: `vars: ansible_python_interpreter` = the venv + `module_defaults` for `mongodb_info`/`mongodb_user`/`mongodb_shell`; `section_3/*` (3.1–3.3, 3.5 with `mongodb_info`, 3.4 with `mongodb_shell`) | 3.1–3.5 | normal run (no login yet) | reports: no users, 3.3 PASS (`mongod`); 3.4 lists no user-defined roles |
| 9 | `section_2/cis_2.1.yml` (`mongodb_user`), `cis_2.2.yml` (`mongodb_info`), `cis_2.3.yml` + admin user/password (Vault), prelim login check | 2.1, 2.2, 2.3 | `rule_2_1` and `rule_2_2: true`, run, rerun | admin created, auth on, bypass off; rerun logs in, `changed=0`; 3.2/3.5 list the admin; mongosh without login is refused |
| 10 | `section_4/*` (4.3 first) + TLS file vars; put test certs on the VM first | 4.3, 4.1, 4.2, 4.4, 4.5 | `rule_4_3: true`, then `rule_4_4: true` | requireTLS, `TLS1_0,TLS1_1` off, FIPS log line; plain mongosh refused; rerun `changed=0` |
| 11 | `tasks/selinux.yml`, `files/selinux/` (optional extra) | — (not CIS) | `mongodb8_cis_selinux_policy: true` on RHEL 9 and 10 | module `200 mongodb` loaded, mongod `mongod_t`; RHEL 8 message only; rerun `changed=0` |
| 12 | `README.md`; then the acceptance test (test-results.md) | all | as a DBA using only the README + `group_vars`, on RHEL 8, 9, 10 | every row of [cis-requirements.md](cis-requirements.md) verified by its "Verify" command |

## Why this order

| Phase | Why here |
|-------|----------|
| 2 before 3 | Prelim is read-only: it proves the connection, facts and config parsing before anything changes |
| 4 early | 5.4 is the smallest real PATCH. Once it works, every later config rule is the same pattern |
| 5 before 6–10 | Reports change nothing (except the 7.2 decision), so they're safe to test on any VM |
| 8a before 8b | The database modules can't run until pymongo exists on the server |
| 8 before 9 | Database reads must work **without** login first; then 2.1 adds login and you see the same reads still work |
| 9 before 10 | Login first, TLS second: if TLS breaks a connection, you already know login works, so the cause is clear |
| 10 needs 4.3 first | mongod refuses `disabledProtocols` / `FIPSMode` unless TLS is on (D25) |
| 11 last | Not a CIS rule; it touches SELinux policy, the most system-wide change |
