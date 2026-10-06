# Session handoff — `mongodb8_cis` on branch `main` (state on 2026-10-05)

Read this when you come back to `main` after testing the rebuild branch. The rebuild branch has its own
`docs/session-handoff.md`; the parent workspace's `CLAUDE.md` has the full project rules.

## 1. The two branches

| | `main` (this branch) | `mongodb8-cis-rebuild` |
|---|---|---|
| Written by | assistant, VM-tested 2026-10-01 (T1–T10) | user, retyped by hand batch by batch |
| Database access (2.x, 3.x) | **`mongodb_shell` only** (mongosh on the server, JS queries, D24) | pymongo venv + `mongodb_info` / `mongodb_user`; `mongodb_shell` for 3.4 and the 3.1 revoke |
| Users of databases without data | listed (shell queries read `system.users`) | not listed by `mongodb_info` (accepted gap) |
| Site variables for Manual rules (D20) | 3.1, 5.2, 6.2, 6.3, 7.1, 7.2 (**ported 2026-10-05**) | same six |
| Test status | T1–T10 on the old KVM lab (RHEL 10 + a RHEL 9.8 VM named rhel8); new options **not VM-tested yet** | compliance test in progress (see its handoff) |

Same layout, rule names, defaults and docs on both, so the test project works with either branch by changing
one line (`version:` in `requirements.yml`).

## 2. What changed on `main` on 2026-10-05 (commit `da298cf`)

| File | Change |
|------|--------|
| `tasks/section_3/cis_3.1.yml` | report names accounts `<db>.<user>` (the query returns `_id` + `roles`); new PATCH: `revokeRolesFromUser` for accounts in `mongodb8_cis_revoke_admin_roles` (only roles that are one of the three **and** on admin) |
| `tasks/section_5/cis_5.2.yml` | block: report + PATCH `auditLog.filter` when `mongodb8_cis_audit_filter` is set and auditing is on |
| `tasks/section_6/cis_6.2.yml` | compares with `mongodb8_cis_resource_limits` (`\| string`); PATCH systemd drop-in when `mongodb8_cis_fix_resource_limits` and drift |
| `tasks/section_7/cis_7.1.yml` | PASS/FAIL (mode 0600/0400, owner **and** group); PATCH `0600` + owner when `mongodb8_cis_fix_key_file_permissions`; `get_checksum: false` |
| `handlers/main.yml` | `daemon_reload: true` (6.2 drop-in) |
| `defaults/main.yml` | `mongodb8_cis_revoke_admin_roles`, `_audit_filter`, `_fix_resource_limits`, `_resource_limits`, `_fix_key_file_permissions` |
| `vars/main.yml` | `mongodb8_cis_limits_dropin` |
| `README.md` | "Manual rules: report, or let the role apply your decision" table; read-only = `--tags audit --skip-tags patch` |
| `docs/design-decisions.md` | D20 revised (table of six variables), D28 (Section 3 weighing + evidence), "Not adopted" bullets (3.2–3.5, 3.3, 3.4, 2.3, 4.5) |
| `docs/manual-remediation.md`, `docs/compliance-test.md`, `docs/cis-requirements.md` | copied from the rebuild branch (same content; compliance test is branch-neutral) |
| `docs/troubleshooting.md` | T-M8 (4.3 stops when TLS files are missing) |
| `docs/code-walkthrough.md`, `docs/test-results.md` | rows for the new options; link to the compliance test |

Checks run: `yamllint .` and `ansible-lint --offline` (production) clean; `--syntax-check` on ansible-core 2.16 OK;
3.1 revoke expressions tested with fake `mongosh` output (a CIS-query false positive like `dbOwner@shop` + `read@admin`
is reported but nothing is revoked).

## 3. How to test `main` (after the rebuild branch)

In `mongodb8-cis-test`:

```bash
sed -i 's/version: mongodb8-cis-rebuild/version: main/' requirements.yml
ansible-galaxy role install -r requirements.yml --force
```

Then the same procedure as the rebuild, on a **fresh snapshot** per VM (`docs/compliance-test.md`):

```bash
ansible <host> -m setup -a 'filter=ansible_distribution*'
ansible-playbook playbooks/site.yml --limit <host> | tee runs/main-<host>-c2.log   # FULL BENCHMARK block commented
ansible-playbook playbooks/prep-tls.yml --limit <host>
# uncomment the FULL BENCHMARK block in group_vars/mongodb/main.yml
ansible-playbook playbooks/site.yml --limit <host> --force-handlers | tee runs/main-<host>-c3.log
ansible-playbook playbooks/site.yml --limit <host> | tee runs/main-<host>-c3-rerun.log   # changed=0
```

Extra checks for the new options on `main`:

| Option | How to test |
|--------|-------------|
| 3.1 revoke | `mongosh`: create `badadmin` with `userAdminAnyDatabase@admin` + `read@shop`; run → 3.1 REVIEW `admin.badadmin`; set `mongodb8_cis_revoke_admin_roles: ["admin.badadmin"]`; run → revoked, `read@shop` kept; rerun → skipped |
| 3.4 drop | `mongosh`: `use shop; db.createRole({role: "orderReader", privileges: [{resource: {db: "shop", collection: "orders"}, actions: ["find"]}], roles: []})`; run → 3.4 REVIEW `shop.orderReader`; set `mongodb8_cis_drop_custom_roles: ["shop.orderReader"]`; run → dropped; rerun → `0 user-defined role(s)`, `changed=0` |
| 3.5 revoke | create `ops` in admin with `clusterAdmin@admin` + `read@shop`; set `mongodb8_cis_revoke_superuser_roles: ["admin.ops"]`; run → `clusterAdmin` revoked, `read@shop` kept; then list `admin.frqadminDB` → run **stops** at the guard |
| 5.2 filter | set `mongodb8_cis_audit_filter: '{ atype: { $in: [ "authenticate" ] } }'` → `auditLog.filter` written, one restart; rerun `changed=0` |
| 6.2 drop-in | fresh install: limits already match → no drop-in. Set `mongodb8_cis_resource_limits.LimitNOFILE: 128000` → drop-in written, restart, `cat /proc/$(pidof mongod)/limits` shows 128000 |
| 7.1 permissions | after 4.3: `chmod 644 /etc/pki/mongodb/ca.pem` → 7.1 FAIL; with the fix on → back to 0600 |
| Read-only run | `--tags audit --skip-tags patch` → `changed=0` even with the variables set |

Record results in `docs/test-results.md` and screenshots in `docs/evidence-images/` (name `NN-<os>-main-<scenario>-<result>.png`).

## 4. Then decide

Compare the two branches after both are tested (`rebuild-vs-main.md` on the rebuild branch lists the differences; the
porting list in it is now done on `main`). Questions to answer: keep `mongodb_shell` (shorter, no venv, sees all users)
or pymongo (structured output, `mongodb_user` for 2.1)? Which branch becomes the published `main`?

## 5. Open issues (same as the rebuild handoff)

- Public `mongodb-cis`: old commit `7543959` still serves the CIS PDF → recreate the GitHub repo.
- Public `mongodb8-cis-test`: `pki/*.key` and a plaintext password in history.
- RHEL 8 never tested for real.

## 6. Review against the benchmark (2026-10-05, after `da298cf`)

- All 23 recommendations checked against the PDF: task title, level gate/tag, Automated/Manual tag, AUDIT/PATCH type,
  `rule_<id>` tag → 23/23. Audit pass values and remediation settings match each rule's PDF text.
- Coverage: 10 Automated PATCH + 6 Manual with an opt-in fix + 7 report-only (1.1, 2.3, 3.2–3.5, 4.5; why: D29). 0 missing.
- Docs trimmed: `build-sequence.md` (rebuild-by-hand plan, pymongo) and empty `evidence.md` removed; the duplicated
  "how to check compliance" part of `cis-requirements.md` now points to `compliance-test.md`; D13a marked rebuild-only.
- `compliance-test.md`: per-host columns (R8/R9/R10), S0 = select the role version in the test project.
- Ready for the compliance test. Not yet VM-tested on `main`: the five ported options, RHEL 8 overall, FIPS (4.4) on RHEL 10.
- Added `docs/sections.md` (every section + the `vars:` in each rule) and `docs/reading-config.md` (from the rebuild, + `| int`).

## 7. Added 2026-10-06: 3.4 and 3.5 opt-in lists (D30, accepted by the user; not committed yet)

- `tasks/section_3/cis_3.4.yml`: report names roles `<db>.<role>`; PATCH `dropRole` for roles in `mongodb8_cis_drop_custom_roles`.
- `tasks/section_3/cis_3.5.yml`: report names accounts `<db>.<user>`; assert that the role's own admin is not listed;
  PATCH `revokeRolesFromUser` (every 3.5 role the account holds) for accounts in `mongodb8_cis_revoke_superuser_roles`.
- `defaults/main.yml`: both variables (`[]`). Docs: D20 table, D28/D29 marked "Revised by D30", D30, `automation-decisions.md`
  (revisions), `manual-remediation.md`, `summary.md`, `README.md`, `cis-requirements.md`, `sections.md`, `code-walkthrough.md`.
- Coverage now: 10 Automated PATCH + **8** Manual opt-in + **5** report only (1.1, 2.3, 3.2, 3.3, 4.5).
- Lint + syntax-check clean; expressions and the guard tested with fake data. **Test on a real mongod** (rows in section 3).
- Not ported to the rebuild branch.

## 8. Test project change (2026-10-06)

`mongodb8-cis-test`: **one settings file, no profiles** (`profiles/` removed). `sysconfig/group_vars/mongodb/main.yml`
holds the site values, `lab_tls_*` and Level 1 + 2; the **FULL BENCHMARK** block at the end (risky rules + site
decisions) is **commented** for the first install (S2) and **uncommented** after `prep-tls.yml` (S4). Comment it again
before the next fresh VM. `prep-tls.yml` is configurable: `lab_tls_cert_src`, `lab_tls_key_src` (separate key → joined
into one PEM), `lab_tls_ca_src`; it stops with a message if MongoDB isn't installed (T-M9).
