# Code walkthrough — `mongodb8_cis`

How the role works, file by file, in the order a run goes through it. Read it next to the code.
Each `Dn` points to the full reasoning and sources in [design-decisions.md](design-decisions.md).

---

## 1. The big picture

```
ansible-playbook site.yml
│
├─ tasks/main.yml ............ the table of contents: runs the parts below, in this order
│
├─ tasks/prelim.yml .......... ALWAYS runs first, changes nothing (except the optional install)
│    check Ansible version, OS, MongoDB version
│    install MongoDB ......... only if mongodb8_cis_install: true  (tasks/install.yml)
│    is MongoDB installed? ... no → stop this host cleanly
│    read /etc/mongod.conf ... → discovered_mongod_conf   (every rule compares against this)
│    read the mongod service → user, PID, limits
│
├─ tasks/section_1 … section_7   one folder per CIS section, one file per CIS rule
│    each rule: is it switched on?  → compare  → fix only if wrong  (or just report)
│
│
└─ handlers/main.yml ......... at the end, only if something changed: restart mongod once, wait for it
```

**A rule runs only if all its switches are on:** its section (`mongodb8_cis_section4`), its level
(`mongodb8_cis_level_1` / `_level_2`), and its own toggle (`mongodb8_cis_rule_4_3`). All three are in `defaults/main.yml`.

## 2. The one pattern to understand: AUDIT → PATCH

Almost every rule that changes something looks like rule 5.4 (`tasks/section_5/cis_5.4.yml`):

```yaml
- name: "5.4 | PATCH | Ensure that new entries are appended to the end of the log file"   # CIS ID | type | exact CIS title
  when:
    - mongodb8_cis_rule_5_4                     # 1. switched on?
    - mongodb8_cis_level_2                      # 2. its level on?
    - discovered_mongod_conf['systemLog']['logAppend'] | default(false) is not true   # 3. AUDIT: is it wrong right now?
  tags: [level2, automated, patch, rule_5.4, logging]   # lets you run/skip it: --tags rule_5.4
  vars:
    mongodb8_cis_5_4_settings: {systemLog: {logAppend: true}}   # the CIS value
  block:
    - name: "5.4 | PATCH | ... | Set systemLog.logAppend: true"   # 4. PATCH: write current config + CIS value
      ansible.builtin.copy:
        content: "{{ discovered_mongod_conf | combine(mongodb8_cis_5_4_settings, recursive=true) | to_nice_yaml(indent=2) }}"
        ...
      notify: Restart mongod                     # 5. ask for one restart at the end
    - name: "5.4 | PATCH | ... | Update the parsed config"         # 6. remember the change for the next rules
      ansible.builtin.set_fact:
        discovered_mongod_conf: "{{ discovered_mongod_conf | combine(mongodb8_cis_5_4_settings, recursive=true) }}"
```

| Step | In plain words | Why |
|------|----------------|-----|
| AUDIT (the `when:`) | "Is the setting already right?" Compared against the config prelim read | Already right → whole block skipped → file untouched, no restart. That's why a second run shows `changed=0` (D2) |
| PATCH `copy` | Writes the **current** config plus the one CIS value | Keeps every site setting (dbPath, bindIp…). `lineinfile` can't safely edit nested YAML (D7). `backup: true` keeps the old file |
| `notify` | Asks for a restart | mongod reads its config only at start. Several rules → still **one** restart at the end |
| `set_fact` | Updates the in-memory config | The next rule builds on this change instead of overwriting it |

Rules that **only report** (all Manual rules, and rules that can't be applied yet) are named `AUDIT` and print a
`PASS` / `FAIL` / `REVIEW` / `NOT APPLICABLE` line with `debug`. They never change anything (D4).

Recurring words:

| Word | Meaning |
|------|---------|
| `discovered_*` | A fact the role **read** from the host (Lockdown naming) |
| `combine(..., recursive=true)` | Merge two dicts; only the given keys change |
| `changed_when: false` | "This task only reads, never report it as a change" |
| `check_mode: false` | Run it even in `--check`, because it only reads (so the preview sees real data) |
| `block:` | Group tasks under one `when:` / `tags:` |

---

## 3. Files that configure the role (no tasks)

| File | What's in it | Key points |
|------|--------------|------------|
| `meta/main.yml` | Name `bunnywkwk.mongodb8_cis`, EL 8/9/10, `min_ansible_version: "2.16.1"`, `collections:` | 2.16.1 = Lockdown's floor and the only ansible-core that runs on RHEL 8 (D15) |
| `requirements.yml` | `community.mongodb` (only `mongodb_shell` is used), `community.general` 11.x | Installed on the **control node**; nothing extra on the DB servers (D13). Keep it: users and ansible-lint install from it |
| `defaults/main.yml` | **Every** switch and site value, with comments | Users override them in `group_vars`, never by editing tasks. Write `true`/`false` unquoted (D19) |
| `vars/main.yml` | Internal values: repo URL, package names, RPM default paths, values computed from the live config | Not for users. Paths come from the live config; RPM defaults are only fallbacks (D8) |
| `.yamllint`, `.ansible-lint` | Lint rules (`production` profile) | Run lint inside the role folder |
| `files/selinux/` | MongoDB's own SELinux module sources (GPL-2.0) | Only for the optional extra (section 8 below) |

**Switches that are off by default** because they can lock users out or break clients (D6):
2.1 (login), 2.2 (no localhost bypass), 4.3 (require TLS), 4.4 (FIPS), 6.1 (new port).
**Site values** CIS leaves to you: admin user/password (2.1), TLS files (4.3), port (6.1), audit destination (5.1),
and two decisions: `mongodb8_cis_javascript_needed` (6.3), `mongodb8_cis_fix_db_path_permissions` (7.2) (D20).

---

## 4. `tasks/main.yml`: the table of contents

Runs prelim (tag `always`), then sections 1–7 in CIS order behind their section switches, then post (writes mongod.conf).

Sections 2 and 3 talk to the **database** (users, roles), so they sit in one `block` with **`module_defaults`**:
the connection settings for `community.mongodb.mongodb_shell` (host, port, login, TLS) are written once there instead
of in every task (D24). They come from the config mongod is **running** with, so they're still right while this run
changes the port or turns on TLS.

## 5. `tasks/prelim.yml`: checks and discovery

| Step | Does | Why |
|------|------|-----|
| Gather minimal facts (only if missing) | OS, version, architecture | Works even with `gather_facts: false` |
| Check ansible-core ≥ 2.16.1 | Stops with a clear message if older | D15 |
| Check OS | RHEL family 8/9/10, x86_64 | Fail fast |
| Check `mongodb8_cis_version` is 8.0 | The benchmark covers MongoDB 8 | D22 |
| Install (if `mongodb8_cis_install`) | Runs `install.yml` (section 6 below) | Before detection, so detection sees the result |
| Detect `mongodb-enterprise-server` | Installed? Which version? | Detect, don't assume |
| Not installed → stop this host | Message + `end_host` | Clean skip, not a failure |
| Check the installed version is 8.0.x | Stops on 7.x / 9.x | Wrong benchmark otherwise (D22) |
| Read + parse `/etc/mongod.conf` | → `discovered_mongod_conf`, plus a copy of the **running** config | The shared AUDIT for all config rules (D7, D14) |
| No deprecated `net.ssl` block | Stops before Section 4 | mongod refuses `net.ssl` and `net.tls` together |
| Database login values | If authorization is already on, user/password must be set; password without spaces/quotes | Sections 2/3 must be able to log in |
| Read the mongod service | User, PID, limits (read-only) | Used by 3.3, 6.2, 7.x and the report |
| Show what was found | Version, config, dbPath, log, service user | Visible summary every run |

## 6. `tasks/install.yml`: optional install

Official MongoDB steps (S1), all declarative modules, so a second run changes nothing:
1. Import the signing key.
2. Add the Enterprise repo (URL built from the OS version).
3. `dnf install mongodb-enterprise` (`present`, never upgrades).
4. Start and enable mongod.

Limitation: `--check` on a host **without** MongoDB fails at `dnf` (T-M5); install with a real run.

---

## 7. The rules, section by section

Type: **PATCH** = changes the host when wrong · **REPORT** = only prints a result · **DECISION** = reports, and fixes
only if you set the decision variable.

### Section 1: Installation and Patching (`tasks/section_1/`)

| Rule | Lvl | Type | What it does |
|------|-----|------|--------------|
| 1.1 | L1 | REPORT | Prints the installed version and where to check for patches. Never upgrades |

### Section 2: Authentication (`tasks/section_2/`), uses the database

| Rule | Lvl | Type | What it does | Why / note |
|------|-----|------|--------------|------------|
| 2.1 | L1 | PATCH, **off by default** | Needs admin user + password → checks if that user exists (`mongodb_shell`) → creates it (role `root`) → sets `security.authorization: enabled` | User is created **before** login is required, so nobody is locked out (D26). Password task has `no_log` |
| 2.2 | L1 | PATCH, **off by default** | Counts database users → stops if there are none → sets `setParameter.enableLocalhostAuthBypass: false` | Without a user, turning off the localhost exception would lock everyone out (D26) |
| 2.3 | L2 | PATCH, off by default | "not applicable" on a standalone; on a member: assert 4.3 is on, then `combine` `clusterAuthMode: x509` + `clusterFile` and copy (restart) | Cluster members only (D31) |

### Section 3: Authorization (`tasks/section_3/`), REPORT, read from the database (3.1 + optional revoke)

| Rule | Lvl | Shows |
|------|-----|-------|
| 3.1 | L1 | Users with `dbOwner`, `userAdmin`, `userAdminAnyDatabase` in `admin` (PASS if none), named `<db>.<user>`. Accounts listed in `mongodb8_cis_revoke_admin_roles` lose those roles (`revokeRolesFromUser`, D20) |
| 3.2 | L1 | Authorization on/off and every user with their roles; `createUser` for listed accounts not found, `grantRolesToUser` with the listed roles an account is missing (`difference`) (D31) |
| 3.3 | L1 | Who mongod runs as (unit `User=` and the real process owner). PASS if not root |
| 3.4 | L1 | Every user-defined role (`<db>.<role>`), its actions and inherited roles. Roles listed in `mongodb8_cis_drop_custom_roles` are dropped (`dropRole`, D30) |
| 3.5 | L2 | Users holding superuser/admin roles (`root`, `clusterAdmin`, …), named `<db>.<user>`. Accounts in `mongodb8_cis_revoke_superuser_roles` lose them; an assert stops the run if the role's own admin is listed (D30) |

Each DB read is `mongodb_shell` with `changed_when: false` and `check_mode: false`: it only reads, so it also runs in `--check`.

### Section 4: Data Encryption (`tasks/section_4/`), **4.3 runs first**

`section_4/main.yml` runs 4.3 before 4.1/4.2/4.4: mongod refuses any `net.tls` option unless TLS is on (D25).

| Rule | Lvl | Type | What it does |
|------|-----|------|--------------|
| 4.3 | L1 | PATCH, **off by default** | Checks both PEM files exist on the host → `net.tls.mode: requireTLS` + `certificateKeyFile` + `CAFile`. The role never creates certificates |
| 4.1 | L2 | PATCH | Adds `TLS1_0,TLS1_1` to `net.tls.disabledProtocols` (keeps any others). If TLS is off: prints FAIL instead |
| 4.2 | L1 | PATCH | Same setting as 4.1 (CIS lists it twice, at different levels); whichever runs first fixes it, the other then finds it compliant |
| 4.4 | L2 | PATCH, **off by default** | `net.tls.FIPSMode: true`, only when TLS is on, otherwise prints FAIL |
| 4.5 | L2 | REPORT | Encryption at rest on/off and the key management (KMIP or keyfile); enabling it is done by hand (D34) |

### Section 5: Audit Logging (`tasks/section_5/`)

| Rule | Lvl | Type | What it does |
|------|-----|------|--------------|
| 5.1 | L1 | PATCH | If there is no `auditLog`, adds one (`syslog` by default; `file` needs JSON/BSON). Never replaces an existing one (D21) |
| 5.2 | L2 | DECISION | Shows `auditLog.filter` (or "auditing off, see 5.1"). With `mongodb8_cis_audit_filter` set (and auditing on) → writes it (D20) |
| 5.3 | L2 | PATCH | If `systemLog.quiet` is true → sets `false`. Fresh install: already compliant |
| 5.4 | L2 | PATCH | If `systemLog.logAppend` isn't true → sets `true`. Fresh install: already compliant |

### Section 6: OS Hardening (`tasks/section_6/`)

| Rule | Lvl | Type | What it does |
|------|-----|------|--------------|
| 6.1 | L1 | PATCH, **off by default** | Checks `mongodb8_cis_port` (1024–65535, not 27017) → labels it `mongod_port_t` for SELinux (if enforcing and not a MongoDB default port) → sets `net.port`. The restart handler then waits on the **new** port (D11) |
| 6.2 | L2 | DECISION | The six limits of the running service vs `mongodb8_cis_resource_limits` (CIS: f, t, v, m = unlimited; n, u = 64000), from systemd (D14). With `mongodb8_cis_fix_resource_limits: true` and drift → systemd drop-in + restart (D20) |
| 6.3 | L2 | DECISION | Shows `security.javascriptEnabled`. With `mongodb8_cis_javascript_needed: false` → sets it `false` (D20) |

### Section 7: File Permissions (`tasks/section_7/`)

| Rule | Lvl | Type | What it does |
|------|-----|------|--------------|
| 7.1 | L1 | DECISION | PASS/FAIL per `keyFile`, TLS key and CA file (0600/0400, owner **and** group = service user), or "not applicable". With `mongodb8_cis_fix_key_file_permissions: true` → `0600` + owner (D20) |
| 7.2 | L1 | DECISION | dbPath PASS/FAIL vs `0770`, owner = service user (RPM ships `0755` → FAIL). With `mongodb8_cis_fix_db_path_permissions: true` → fixes it (D20) |

---

## 9. `handlers/main.yml`: one restart, then a check

Both handlers listen to `Restart mongod`. They run **once**, at the end, and only if a PATCH changed something:
1. restart mongod (`daemon_reload: true`, so a 6.2 drop-in is read);
2. wait up to 60 s until it accepts connections on its (possibly new) address and port.

If a config change broke mongod, the run fails **here**, right after the change. mongod has no config dry-run, so this is the check (D7).
