# Troubleshooting log: `mongodb_cis`

Errors hit while building and testing this role, with the exact message, cause and fix.
Workstation/tooling errors: [../../docs/troubleshooting.md](../../docs/troubleshooting.md).
Newest last. Format: **Symptom** (exact output) → **Cause** → **Fix** → **Prevent**.

---

## T-M1: `meta/main.yml` — `'dependencies' was unexpected` (2026-09-29)

**Symptom**
```
schema[meta]: $.galaxy_info Additional properties are not allowed ('dependencies' was unexpected).
meta/main.yml:1
```

**Cause:** `dependencies: []` was indented under `galaxy_info:`, which made it a child key. It's a **top-level** key.

**Fix:** move `dependencies: []` to column 0.

**Prevent:** in YAML, indentation decides the parent key. Compare with the schema or the walkthrough.

## T-M2: yamllint `line too long` and ansible-lint `braces` warning (2026-09-29)

**Symptom**
```
meta/main.yml
  3:81      error    line too long (111 > 80 characters)  (line-length)

WARNING  Found incompatible custom yamllint configuration (.yamllint), please either remove the file or edit it to comply with:
  - braces.max-spaces-inside must be 1.
```

**Cause:** yamllint's default line limit is 80 characters. The trimmed `.yamllint` lacked `braces`/`brackets: max-spaces-inside: 1`, which ansible-lint requires.

**Fix:** `.yamllint` gets `line-length: disable` and `braces`/`brackets` `max-spaces-inside: 1` (Lockdown settings).

**Prevent:** run lint **inside** `mongodb_cis/`; yamllint reads `.yamllint` from the current directory.

## T-M3: skeleton leftovers break lint (2026-09-29)

**Symptom**
```
yaml[comments]: Missing starting space in comment
defaults/main.yml:1
syntax-check[specific]: The role 'mongodb_cis' was not found in: .../mongodb_cis/tests/roles:...
tests/test.yml:6:7
```

**Cause:** `ansible-galaxy role init` adds `#SPDX-License-Identifier: MIT-0` lines, and a `tests/` playbook that can't find the role.

**Fix:** delete the `#SPDX` lines (`---` must be line 1), and `git rm -r tests`. The role is tested from a separate test project and Molecule.

**Prevent:** replace skeleton files completely when writing them.

## T-M4: "Conditionals must have a boolean result" with `-e mongodb_cis_install=true` (2026-09-30)

**Symptom** (RHEL 10, main env, ansible-core 2.20.7)
```
TASK [mongodb_cis : INSTALL | PATCH | Import the MongoDB package signing key] ***
[ERROR]: Task failed: Conditional result (True) was derived from value of type 'str' at "<CLI option '-e'>". Conditionals must have a boolean result.

Origin: /home/frqadmin/ansible-cis/mongodb_cis/tasks/prelim.yml:39:9

38 - name: "PRELIM | PATCH | Install MongoDB when requested"
39   when: mongodb_cis_install
           ^ column 9

fatal: [rhel10-mongo]: FAILED! => {"changed": false, "msg": "Task failed: Conditional result (True) was derived from value of type 'str' at \"<CLI option '-e'>\". Conditionals must have a boolean result."}
```

**Cause:** (not `--check`; the `-e` value)
- `-e key=value` always passes a **string** (`"true"`).
- ansible-core 2.19+ rejects non-boolean conditionals.
- On 2.16 (RHEL 8 env) there is no error, but any non-empty string is true, so `-e mongodb_cis_install=false` would install.

**Fix:** `when: mongodb_cis_install | bool`, and `mongodb_cis_install | bool` in the Report message. Full proof on both versions, including `=false` running on 2.16: D19.

**Prevent:** every user-facing switch in a condition uses `| bool` (CLAUDE.md rule). Alternative on the CLI: pass JSON, `-e '{"mongodb_cis_install": true}'`.

## T-M5: `No package mongodb-org available.` in `--check` (2026-09-30)

**Symptom** (RHEL 10 base snapshot, `ansible-playbook site.yml --limit rhel10 -e mongodb_cis_install=true --check --diff`)
```
TASK [mongodb_cis : INSTALL | PATCH | Add the official MongoDB repository] ***
--- before: /etc/yum.repos.d/mongodb-org-8.0.repo
+++ after: /etc/yum.repos.d/mongodb-org-8.0.repo
@@ -0,0 +1,7 @@
+[mongodb-org-8.0]
+baseurl = https://repo.mongodb.org/yum/redhat/10/mongodb-org/8.0/x86_64/
...
changed: [rhel10-mongo]

TASK [mongodb_cis : INSTALL | PATCH | Install MongoDB packages] ***
[ERROR]: Task failed: Module failed: Failed to install some of the specified packages
Origin: /home/frqadmin/ansible-cis/mongodb_cis/tasks/install.yml:16:3
fatal: [rhel10-mongo]: FAILED! => {"changed": false, "failures": ["No package mongodb-org available."], "msg": "Failed to install some of the specified packages", "rc": 1, "results": []}
```

**Cause:**
- In `--check`, `yum_repository` only **simulates** the repo file (reports `changed`, shows the diff, writes nothing).
- `dnf` in check mode still resolves the package against the real, enabled repos. There is no MongoDB repo yet, so the package can't be found.
- `systemd_service` would fail next for the same reason (no `mongod` unit yet).

**Resolution: accepted as a known limitation, not coded around** (user decision, 2026-09-30).
- Handling it needs extra check-mode conditions in `install.yml` for a case that only exists on a host without MongoDB.
- That's over-engineering for an opt-in install step.
- **Use a real run to install** (tests 04–07 in [test-results.md](test-results.md) passed). `--check` stays fully supported for everything after the install, i.e. prelim and the CIS rules.

**Prevent:** don't combine `--check` with `mongodb_cis_install=true` on a host without MongoDB.

## T-M6: `net.port` written as a string `'27018'` on ansible-core 2.16 (2026-09-30, found in pre-test)

**Symptom** (Section 6 draft run on localhost against a copy of the RPM `mongod.conf`, ansible-core 2.16.19; 2.20.7 was fine)
```
   written:
3:  port: '27018'
```

**Cause:**
- The draft put the setting in the rule's `vars:` as `port: "{{ mongodb_cis_port | int }}"`.
- On 2.16, a templated value stored in a variable is turned back into **text**, so `to_nice_yaml` quoted it.
- mongod expects a number for `net.port`.
- 2.20 keeps native types, so the bug only shows on the RHEL 8 env.

**Fix:** build the setting **inside the same expression** that writes the file: `combine({'net': {'port': mongodb_cis_port | int}}, recursive=true)`. Re-test on both versions: `port: 27018` (number), run 2 `changed=0`.

**Prevent:** only put **literal** values (`true`, `false`, fixed text) in a rule's `vars:` settings dict. Anything computed from a variable goes inline in the `combine(...)`. Test every PATCH on **both** ansible-core versions before handing it over.
