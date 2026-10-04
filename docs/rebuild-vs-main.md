# Rebuild vs `main` — what changed, and what to port

`main` = the finished, VM-tested role (`origin/main` 208e3e2). `mongodb8-cis-rebuild` = the hand-typed rebuild.
Use this list to carry the rebuild's improvements back to `main` (or to know what you'd lose by going back).
Compared on 2026-10-04 (`~/mongodb-cis` = `main`, `~/ansible-cis/mongodb-cis` = rebuild).

## A. Portable to `main` as-is (don't depend on how the role talks to the database)

### A1. "Site decision" variables on Manual rules (D20)

The pattern: a Manual rule **reports** by default (AUDIT); an optional variable next to its toggle lets the site say
"apply CIS's remediation", which adds a PATCH step (AUDIT → PATCH, only on drift).

| Rule | Variable(s) | In `main`? | Port |
|------|-------------|-----------|------|
| 5.2 audit filter | `mongodb8_cis_audit_filter: ""` | report only | add var + PATCH block (`copy` + `set_fact`, `notify: Restart mongod`) |
| 6.2 resource limits | `mongodb8_cis_fix_resource_limits: false`, `mongodb8_cis_resource_limits` (CIS values) | report only, values hardcoded | add both vars, `vars/` `mongodb8_cis_limits_dropin`, drift var + PATCH block (dir + drop-in `copy`) |
| 6.3 server-side JS | `mongodb8_cis_javascript_needed: true` | ✅ already | — |
| 7.1 key file permissions | `mongodb8_cis_fix_key_file_permissions: false` | report only | add var + PATCH `file` task (loop over existing files) |
| 7.2 dbPath permissions | `mongodb8_cis_fix_db_path_permissions: false` | ✅ already | — |

### A2. Other fixes

| Change | File | Why |
|--------|------|-----|
| Restart handler `daemon_reload: true` | `handlers/main.yml` | needed by 6.2's drop-in |
| 7.1 PASS also checks the group (`gr_name`) | `section_7/cis_7.1.yml` | CIS 7.1: `chown mongodb:mongodb` = owner **and** group |
| 6.2 compares with `\| string` | `section_6/cis_6.2.yml` | `64000` and `"64000"` both match systemd's text value |

Files to copy for A: `defaults/main.yml` (rules 5.2, 6.2, 7.1 lines), `vars/main.yml` (`mongodb8_cis_limits_dropin`),
`tasks/section_5/cis_5.2.yml`, `tasks/section_6/cis_6.2.yml`, `tasks/section_7/cis_7.1.yml`, `handlers/main.yml`.
Then lint, and test on a VM (6.2 drop-in + restart is new).

## B. Tied to the pymongo approach (D13a) — only if you keep it

`main` reaches the database with `mongodb_shell` only (D13: nothing installed on the server). The rebuild uses the
pymongo modules. Going back to `main`'s approach = **don't port these**.

| Rebuild piece | Where | `main` equivalent |
|---------------|-------|-------------------|
| `tasks/pymongo.yml` (Python 3.12 + venv + pymongo) | new file, imported in `tasks/main.yml` | none |
| `mongodb8_cis_pymongo_venv`, `_pymongo_spec`, `_python_bin`, `_python_packages` | `vars/main.yml` | none |
| `vars: ansible_python_interpreter` on the sections 2+3 block | `tasks/main.yml` | none |
| `module_defaults` for `mongodb_info` + `mongodb_user` (YAML anchor) | `tasks/main.yml` | `module_defaults` for `mongodb_shell` only |
| 2.1 creates the admin with `mongodb_user` | `section_2/cis_2.1.yml` | `mongodb_shell` + `createUser` script |
| 2.2, 3.1, 3.2, 3.5 read users with `mongodb_info` | sections 2, 3 | `mongodb_shell` + JS queries |
| Accepted gap: users of empty databases not listed | `mongodb_info` limit | none (shell queries see them) |

**Trade-off in one line:** pymongo modules = structured results and built-in idempotence, but Python 3.12 + pip on
every DB server; `mongodb_shell` = nothing extra on the server, but JavaScript strings inside the tasks.

## C. Same in both (no action)

Layout (D27), AUDIT → PATCH (D2), levels gate rules (D18), risky rules off (D6), no `| bool` on switches (D19),
config writes with `combine` + `set_fact` (D7), section 4 order (D25), SELinux extra (D17).

## Where the database settings live (both branches)

Both put the sections 2+3 `block` with `module_defaults` in `tasks/main.yml`: the settings apply to every task of
both sections, so they wrap both imports once. A separate file (e.g. `tasks/database.yml` holding the block, imported
from `main.yml`) would make `main.yml` shorter but adds a file and one more hop to read; not worth it for ~25 lines.
