# Session handoff — `mongodb8_cis` rebuild (state on 2026-10-05)

Read this first on a new laptop or in a new AI session, then `docs/compliance-test.md`. The parent workspace's
`CLAUDE.md` (not in this repo) has the full project rules; the essentials are below.

## 1. What this is

- Ansible role **`mongodb8_cis`**: hardens **MongoDB 8.0 Enterprise**, standalone `mongod`, on **RHEL 8, 9, 10** to the
  **CIS MongoDB 8 Benchmark v2.0.0** (23 recommendations: 13 Level 1, 10 Level 2).
- Repo `github.com/bunnywkwk/mongodb-cis` (public). **Branch `mongodb8-cis-rebuild`** = the role retyped by hand
  (all 10 batches done, commit `ce2abeb` + this session's docs). Branch `main` = the older, VM-tested version.
- Test project (separate repo): `github.com/bunnywkwk/mongodb8-cis-test` (inventory, `group_vars`, profiles, playbooks).
- Layout = ansible-lockdown: `tasks/section_<n>/main.yml` + `cis_<id>.yml`, AUDIT → PATCH, `discovered_*`, tags on the rule block.

## 2. How the user works

- The user types `tasks/` and `handlers/` from `docs/build-guide.md`; the assistant writes everything else and, on
  request, fixes typed files to match the guide. Answer pasted snippets **concisely**. Plain words, tables, sources.
- After changes: `cd mongodb_cis && yamllint . && ansible-lint --offline` (run **inside** the role).
- Never commit/push unless asked. Never run the role against a host without the user's OK. Never commit the CIS PDF or
  spreadsheet (CIS terms forbid hosting them).

## 3. State of the role

| Area | State |
|------|-------|
| Code | all 23 rules + prelim, install, pymongo, handlers, SELinux extra. Lint clean (production profile), `--syntax-check` OK on ansible-core 2.16 |
| Risky rules (off by default) | 2.1, 2.2, 4.3, 4.4, 6.1 (+ 2.3 off: sharded only) |
| Site variables for Manual rules (D20) | 3.1 `mongodb8_cis_revoke_admin_roles: []`, 5.2 `_audit_filter`, 6.2 `_fix_resource_limits`, 6.3 `_javascript_needed`, 7.1 `_fix_key_file_permissions`, 7.2 `_fix_db_path_permissions` |
| Report only, by decision | 1.1, 2.3 (N/A standalone), 3.2–3.5 (D28), 4.5 (evidence in `manual-remediation.md`) |
| Built then reverted | 3.2 `mongodb8_cis_db_users` (too complex, 2026-10-04) |
| `quiet: true` on asserts | left out (user's choice) |

Key docs: `compliance-test.md` (the proof sheet to fill), `cis-requirements.md` (23-rule checklist + how to check
compliance), `manual-remediation.md` (hand fixes + why not automated), `design-decisions.md` (D0–D28, "Not adopted"),
`troubleshooting.md` (T-M1…T-M8), `test-results.md` (evidence index), `build-guide.md` (code by batch).

## 4. Testing — where we are

| VM | Run | Result |
|----|-----|--------|
| `rhel8-mongo` | C2 (Level 2 defaults) | ✅ `changed=7 failed=0` (`runs/rhel8-c2.log`). Rerun for `changed=0` not done yet. OS not yet confirmed as RHEL 8 |
| `rhel9-mongo` | C3 (full benchmark) | ❌ `changed=5 failed=1`: 4.3 stopped, TLS files missing (prep-tls not run). **T-M8.** 2.1 admin created, 2.1/2.2 written to `mongod.conf`, mongod **not restarted yet** |
| `rhel10-mongo` | — | not started on this branch |

**Next steps (RHEL 9 first):**

```bash
cd mongodb8-cis-test
ansible-galaxy role install -r requirements.yml --force                     # role from branch mongodb8-cis-rebuild
ansible rhel9-mongo -m setup -a 'filter=ansible_distribution*'              # confirm real OS
ansible-playbook playbooks/prep-tls.yml --limit rhel9-mongo
ansible-playbook playbooks/site.yml --limit rhel9-mongo -e @profiles/c3-full-benchmark.yml --force-handlers | tee runs/rhel9-c3.log
ansible-playbook playbooks/site.yml --limit rhel9-mongo -e @profiles/c3-full-benchmark.yml | tee runs/rhel9-c3-rerun.log   # expect changed=0
```

Then on the host: every check in `docs/compliance-test.md`, screenshots `04-rhel9-…` to `27-rhel9-…` in
`docs/evidence-images/`, results + verdict in that file. Then the same for RHEL 8 and RHEL 10.

**Unconfirmed expectations in `compliance-test.md`** (correct them from the host): 4.4 FIPS log wording, 5.1
`journalctl -t mongod` shows audit events, 6.2 limit names in `/proc/<pid>/limits`.

## 5. Test project (`mongodb8-cis-test`) — set up 2026-10-05, **not committed yet**

| File | Content |
|------|---------|
| `requirements.yml` | role `mongodb8_cis` from branch `mongodb8-cis-rebuild`; `community.mongodb` 1.8.x, `community.general` 11.x |
| `sysconfig/inventory.yml` | group `mongodb`: rhel8 `192.168.20.50`, rhel9 `192.168.20.30`, rhel10 `192.168.20.40`; user `frqadmin`, key `~/.ssh/id_ed25519_cis` |
| `sysconfig/group_vars/mongodb/main.yml` | site values: install true, admin `frqadminDB` + **plaintext password (lab only, user's choice)**, TLS `/etc/pki/mongodb/server.pem` + `ca.pem`, port 27100 |
| `profiles/c1-level1-defaults.yml` | Level 1, defaults (expected results in the header) |
| `profiles/c2-level2-defaults.yml` | Level 1 + 2, defaults |
| `profiles/c3-full-benchmark.yml` | Level 2 + risky rules on + site decisions (6.2, 6.3 false, 7.1, 7.2) |
| `playbooks/prep-tls.yml` | copies `pki/<host>.pem` → `server.pem`, `pki/ca.pem` → `ca.pem` (run before C3) |
| `docs/test-runs.md` | "Compliance check" section with the run order |
| `runs/` | `tee` logs (gitignored) |

Run: `ansible-playbook playbooks/site.yml --limit <host> -e @profiles/<profile>.yml`. Values live in `group_vars`,
switches in the profile; Ansible combines them at run time.

**To continue on another laptop**, the test project must travel too. `pki/` is gitignored (but old copies are still
tracked: `git rm -r --cached pki/`), so copy `pki/` by hand (USB/scp), or regenerate the certs. The group_vars file has
the plaintext lab password: push only if it is a throwaway lab password (the repo is public).

## 6. New laptop setup (control node)

- Python 3.12 venv with **ansible-core 2.16.x** (one env for RHEL 8/9/10; 2.17+ breaks `dnf` on RHEL 8):
  `python3.12 -m venv ~/.ansible-2.16.1-env && ~/.ansible-2.16.1-env/bin/pip install 'ansible-core>=2.16.1,<2.17'`
- Collections per project: `ansible-galaxy collection install -r requirements.yml`.
- SSH key `~/.ssh/id_ed25519_cis` (copy it, or create a new one and add it to the VMs).
- Lint tools (only for linting): `yamllint`, `ansible-lint`.
- Proxmox VMs need CPU type `host` (MongoDB 8 needs AVX). Check host → OS mapping first (an earlier lab had them swapped).

## 7. Open issues

- Public `mongodb-cis`: old commit `7543959` still serves the CIS PDF by direct URL → delete and recreate the GitHub repo.
- Public `mongodb8-cis-test`: `pki/*.key` and a plaintext password in history → regenerate certs, change the password,
  purge history before real use.
- RHEL 8 never tested for real (earlier inventory swap).
- Offered, not done: `code-walkthrough.md` for batches 2–10; `| int` findings in `reading-config.md`; README (with the
  kickstart preset); port 2.1–7.2 improvements to `main` (`rebuild-vs-main.md`); rename the two "Check …" asserts in
  2.1/2.2 from PATCH to AUDIT.
