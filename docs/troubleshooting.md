# Troubleshooting log: `mongodb8_cis`

Errors hit while building and testing this role, with the exact message, cause and fix.
Workstation/tooling errors: [../../docs/troubleshooting.md](../../docs/troubleshooting.md).
Newest last. Logs before 2026-10-01 show the old role name `mongodb_cis` (renamed to `mongodb8_cis`, D23). Format: **Symptom** (exact output) → **Cause** → **Fix** → **Prevent**.

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

**Prevent:** run lint **inside** `mongodb8_cis/`; yamllint reads `.yamllint` from the current directory.

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

## T-M4: "Conditionals must have a boolean result" with `-e mongodb8_cis_install=true` (2026-09-30)

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
- On 2.16 (RHEL 8 env) there is no error, but any non-empty string is true, so `-e mongodb8_cis_install=false` would install.

**Fix:** `when: mongodb8_cis_install | bool`, and `mongodb8_cis_install | bool` in the Report message. Full proof on both versions, including `=false` running on 2.16: D19.

**Prevent:** every user-facing switch in a condition uses `| bool` (CLAUDE.md rule). Alternative on the CLI: pass JSON, `-e '{"mongodb8_cis_install": true}'`.

## T-M5: `No package mongodb-org available.` in `--check` (2026-09-30)

**Symptom** (RHEL 10 base snapshot, `ansible-playbook site.yml --limit rhel10 -e mongodb8_cis_install=true --check --diff`)
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

**Prevent:** don't combine `--check` with `mongodb8_cis_install=true` on a host without MongoDB.

## T-M6: `net.port` written as a string `'27018'` on ansible-core 2.16 (2026-09-30, found in pre-test)

**Symptom** (Section 6 draft run on localhost against a copy of the RPM `mongod.conf`, ansible-core 2.16.19; 2.20.7 was fine)
```
   written:
3:  port: '27018'
```

**Cause:**
- The draft put the setting in the rule's `vars:` as `port: "{{ mongodb8_cis_port | int }}"`.
- On 2.16, a templated value stored in a variable is turned back into **text**, so `to_nice_yaml` quoted it.
- mongod expects a number for `net.port`.
- 2.20 keeps native types, so the bug only shows on the RHEL 8 env.

**Fix:** build the setting **inside the same expression** that writes the file: `combine({'net': {'port': mongodb8_cis_port | int}}, recursive=true)`. Re-test on both versions: `port: 27018` (number), run 2 `changed=0`.

**Prevent:** only put **literal** values (`true`, `false`, fixed text) in a rule's `vars:` settings dict. Anything computed from a variable goes inline in the `combine(...)`. Test every PATCH on **both** ansible-core versions before handing it over.

## T-M7: mongod does not start after the SELinux extra loads MongoDB's module (2026-10-01, VM test T9)

**Symptom** (lab VM at 192.168.122.83, RHEL 9.8, SELinux enforcing; `mongodb8_cis_selinux_policy: true`)
```
RUNNING HANDLER [mongodb8_cis : Wait for mongod to accept connections]
fatal: [rhel8-mongo]: FAILED! => {"changed": false, "elapsed": 60, "msg": "Timeout when waiting for 127.0.0.1:27100"}
```
`systemctl status mongod`: `Main process exited, code=exited, status=14`. `/var/log/mongodb/mongod.log`:
```
"s":"E","c":"NETWORK","id":23024,"msg":"Failed to unlink socket file","attr":{"path":"/tmp/mongodb-27100.sock","error":"Permission denied"}
"s":"F","c":"ASSERT","id":23091,"msg":"Fatal assertion","attr":{"msgid":40486,...}
```
`ausearch -m AVC`:
```
avc: denied { unlink } for comm="mongod" name="mongodb-27100.sock" scontext=system_u:system_r:mongod_t:s0
     tcontext=system_u:object_r:unlabeled_t:s0 tclass=sock_file permissive=0 trawcon="system_u:object_r:mongod_tmp_t:s0"
```

**Cause:**
- The RHEL 9 base policy's `mongodb` module labels mongod's socket files `mongod_tmp_t`
  (`type_transition mongod_t tmp_t:sock_file mongod_tmp_t`).
- The SELinux extra loads MongoDB's module at priority 200, which replaces the base module. MongoDB's module has no
  `mongod_tmp_t` (its sockets stay `tmp_t`, `allow mongod_t tmp_t:sock_file { create setattr unlink }`).
- The socket created by the running mongod keeps the now-unknown label → `unlabeled_t`. At the restart, the new mongod
  must unlink it, SELinux denies it, and mongod aborts (exit 14) before opening its port.
- Confirmed in a Rocky 9 container with `seinfo`/`sesearch`: `mongod_tmp_t` exists in the base policy and is gone after
  loading MongoDB's module. Not caught by the earlier container tests: they cannot run mongod with SELinux active.
- RHEL 10 has no base `mongodb` module (mongod unconfined before the extra), so its sockets are not `mongod_tmp_t`.

**Fix (role):** `tasks/selinux.yml`, right after loading the module: find `mongodb-*.sock` in
`net.unixDomainSocket.pathPrefix` (default `/tmp`) and remove them; mongod recreates them at the restart with the new
label. Runs only in the run that loads the module. Re-tested in a Rocky 9 container: run 1 removes the socket, run 2
`changed=0` and leaves the running mongod's socket alone.

**Recover a host that already failed:**
```bash
sudo rm -f /tmp/mongodb-*.sock
sudo systemctl start mongod && sudo systemctl status mongod --no-pager | head -5
```

**Prevent:** when replacing an SELinux policy module, check for files the old module labelled with types the new one
does not define (`seinfo -t <type>` on both policies).

## T-M8: 4.3 stops with "PEM files that exist on the host" (2026-10-05, compliance test on RHEL 9)

```
TASK [mongodb8_cis : 4.3 | PATCH | Ensure Encryption of Data in Transit TLS or SSL (Transport Encryption) | Check the PEM files] ***
fatal: [rhel9-mongo]: FAILED! => {
    "assertion": "discovered_4_3_files['results'] | selectattr('stat.exists') | list | length == 2",
    "changed": false,
    "evaluated_to": false,
    "msg": "4.3: set mongodb8_cis_tls_certificate_key_file and mongodb8_cis_tls_ca_file to PEM files that exist on the host (found '/etc/pki/mongodb/server.pem', '/etc/pki/mongodb/ca.pem')."
}
rhel9-mongo                : ok=42   changed=5    unreachable=0    failed=1    skipped=5
```

- **Cause:** the full-benchmark profile (C3) ran before `playbooks/prep-tls.yml`, so the certificate and CA were not on
  the host. The guard worked as designed: it stopped **before** writing `requireTLS`, which would have stopped mongod.
- **Side effect:** the play stopped, so the `Restart mongod` handler did not run. 2.1/2.2 had already written
  `authorization: enabled` and `enableLocalhostAuthBypass: false` to `/etc/mongod.conf`; mongod kept running with the old
  settings until the next restart. The next run's 4.3 change triggers that restart.
- **Fix:** `ansible-playbook playbooks/prep-tls.yml --limit rhel9-mongo`, then rerun the full benchmark (`site.yml`, group_vars).
- **Prevent:** follow the order in [compliance-test.md](compliance-test.md) (S3 before S4). Add `--force-handlers` to
  runs that change the config, so a later failure still restarts mongod with what was already written.

## T-M9: `prep-tls.yml` fails with "failed to look up group mongod" (2026-10-05, compliance test, RHEL 8/9/10)

**Symptom**
```
TASK [Create /etc/pki/mongodb] *************************************************
fatal: [rhel8-mongo]: FAILED! => {"changed": false, "gid": 0, "group": "root", "mode": "0755", "msg": "chgrp failed: failed to look up group mongod", "owner": "root", "path": "/etc/pki/mongodb", "secontext": "unconfined_u:object_r:cert_t:s0", "size": 6, "state": "directory", "uid": 0}
```
(same on `rhel9-mongo` and `rhel10-mongo`)

**Cause:** `prep-tls.yml` (test project) ran on fresh VMs before MongoDB was installed. The `mongod` user and group are
created by the MongoDB RPM, so `group: mongod` can't be set yet. The directory was created as `root:root 0755`.

**Fix:** run the install step first (`site.yml` with the FULL BENCHMARK block commented, `mongodb8_cis_install: true`),
check `ansible mongodb -m command -a 'id mongod'`, then rerun `prep-tls.yml`; it corrects the directory's group and mode.

**Prevent:** follow the step order in [compliance-test.md](compliance-test.md): S2 install → S3 TLS files → S4 full benchmark.
