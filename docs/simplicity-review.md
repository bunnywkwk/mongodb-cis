# Simplicity review — why every line is there

A habit for writing and reviewing this role: **start with the simplest code that does what the CIS
recommendation says; anything more has to earn its place with a reason you can say out loud.** If someone points at a
line and asks "why is this here?", there is an answer: a CIS requirement, a real failure it prevents, or a project
rule. If there is no answer, the line goes.

## 1. Five questions for every task (or line)

| # | Question | If the answer is "no" / "I don't know" |
|---|----------|----------------------------------------|
| 1 | Does it trace back to a CIS recommendation (ID, audit, remediation) or a project rule in `CLAUDE.md`? | Remove it, or justify it as a documented non-CIS extra (like the SELinux option) |
| 2 | Is there a **simpler form** that gives the same result? (one module instead of three tasks, a filter instead of a loop) | Use the simpler form |
| 3 | What **breaks** if I delete it? Name the host, input or run where it goes wrong | Nothing breaks → delete it |
| 4 | Is it the **idiomatic module** (not `command`/`shell`)? | Switch to the module, or justify why the module can't do it |
| 5 | Does it keep the role **safe**: rerun = `changed=0`, `--check` works, a risky change is behind a toggle? | Fix it; safety rules are not optional |

Question 3 is the strongest test. "It looks cleaner" or "just in case" are not reasons; "on a host where X, task Y
would do Z" is.

## 2. The ladder: pick the lowest step that works

| Step | Shape | Use when | Example in this role |
|------|-------|----------|----------------------|
| 1 | One declarative module | The module can compare and fix by itself | `install.yml`: `dnf`, `yum_repository`, `systemd_service` |
| 2 | Module + `when:` | Only some hosts or settings need it | 5.1: write `auditLog` only when there is none |
| 3 | `block:` with AUDIT → PATCH | Reading the state needs its own step, or a human verdict is required | 7.2: `stat` → report → optional `file` |
| 4 | `command`/`shell` | No module can do it (rare) | None so far; would need `changed_when` + `check_mode: false` |

Going up a step is the "complexity" that has to be justified.

## 3. What counts as a good reason for extra code

| Reason | Why it is valid | Example |
|--------|-----------------|---------|
| The benchmark asks for it | Source of truth (principle 1) | 7.2 checks mode **and** owner, because the CIS audit checks both |
| A Manual rule needs a human-readable verdict | Manual = report by default (principle 5) | 7.1/7.2 `debug` prints PASS/FAIL with expected vs found |
| It prevents a real failure | Named host + run where it breaks | 5.1 `set_fact` after `copy` (see 4.4) |
| Fail fast before changing anything | Clear stop beats a half-hardened host | prelim asserts: OS, arch, version |
| Safety / idempotence | Principle 3 | 5.1 `when:` never touches an existing `auditLog`; `backup: true` on config writes |
| Removes repetition | One place to fix | block `vars:` for the dbPath expression in 7.2 |

**Not** good reasons: "Lockdown does it" (D6: no `disruption_high`), "might be useful later" (no speculative
features), "handles a case nobody hits" (user feedback 2026-09-30: document it in troubleshooting instead).

## 4. Worked examples from this role

(How the role reads and writes `mongod.conf` values is explained in [reading-config.md](reading-config.md).)

### 4.1 7.2: three tasks or one? (keep three)

Simpler alternative: one `file` task with `check_mode: "{{ not mongodb8_cis_fix_db_path_permissions }}"`.

| | Current: `stat` + `debug` + `file` | One `file` + `check_mode` |
|---|---|---|
| Same CIS content | yes | yes |
| Readable verdict ("7.2 FAIL: /var/lib/mongo is 0755 ...") | yes | no, only `changed` + diff |
| Rerun with the fix off | `changed=0` | `changed=1` every run |
| Fits "Manual = AUDIT + report" | yes | no |

Verdict: the extra two tasks are justified by reasons "human-readable verdict" and "idempotence".

### 4.2 `register` vs block `vars:` (both needed, different jobs)

| Line | Job | Delete it and... |
|------|-----|------------------|
| `vars: mongodb8_cis_7_2_db_path` | Names a formula: **which** directory | the same long expression is copied 4 times |
| `register: discovered_db_path` | Saves what `stat` **saw** | the report can't say PASS/FAIL |
| No `register` used by the PATCH | `file` compares by itself | (nothing to delete: step 1 of the ladder) |

### 4.3 Why `mongodb8_cis_supported_versions` is in `vars/`, not `defaults/`

`group_vars` overrides `defaults/` but not `vars/`. A user can pick `mongodb8_cis_version`, but can't widen the
supported list from inventory, so the CIS 8 rules are not applied to MongoDB 7 by accident. `-e` can still force it:
a deliberate bypass the user owns.

### 4.4 5.1: why `set_fact` after `copy` (looks redundant, isn't)

Every config rule builds the new file from `discovered_mongod_conf`. Without updating it after 5.1 writes `auditLog`,
5.2 would write the file again from the **old** in-memory copy and silently drop 5.1's `auditLog`. Delete the line
→ two rules undo each other in one run.

### 4.5 prelim: `discovered_mongod_running_conf` (a second copy of the config)

Rules edit `discovered_mongod_conf` during the run, but mongod only uses those edits after the restart handler.
The role's own DB connection (sections 2/3) must use what mongod is **running** with now (port, TLS, auth), so
prelim keeps a frozen copy. Delete it → the role connects with settings mongod hasn't loaded yet.

### 4.6 prelim: the architecture assert (keep it)

It doesn't claim other architectures fail; it says only x86_64 was tested. It stops before anything changes, with a
clear message. Cost: one line. Removing it trades a clear stop for an unknown half-run.

### 4.7 dbPath fallback `/var/lib/mongo` (accepted simplification)

mongod's built-in default is `/data/db`; `/var/lib/mongo` comes from the RPM's `mongod.conf`. Handling a config with
`dbPath` deleted would add code for a case where mongod normally won't even start, so the fallback stays simple
and the limit is written down here.

## 5. Things we decided **not** to add

| Not added | Why |
|-----------|-----|
| Global `disruption_high` switch | Not in CIS; hides which rules it covers (D6). Risky rules have their own toggle |
| `\| bool` on every switch | Users write real booleans; `\| bool` only for values read from `mongod.conf` (D19) |
| Goss audit framework (`audit_only`, `run_audit`) | `--check --diff` + AUDIT reports already cover it |
| Special `--check` handling for a first install | Rare path; documented as a limitation instead |
| Auto-fix for Manual rules whose fix is a people decision (3.x users/roles, 1.1 upgrades) | Guessing would be wrong; they stay report-only |

## 6. Review checklist (use after each batch)

```text
[ ] Every task name starts with a CIS ID (or PRELIM/INSTALL) and matches the CIS title
[ ] Every rule has: its toggle + level in when:, tags on the block
[ ] Lowest ladder step used; any extra step has a reason from section 3
[ ] Every register is read by a later task; every block var is used more than once or shortens a long expression
[ ] No command/shell, or it has changed_when (+ check_mode: false if read-only)
[ ] Rerun gives changed=0; risky change sits behind a default-false toggle with a WARNING
[ ] For each line I can answer: "what breaks if I delete it?"
```
