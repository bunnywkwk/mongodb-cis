# mongodb8_cis

Ansible role that hardens a standalone **MongoDB 8.0 Enterprise** server (`mongod`) on **RHEL 8, 9 and 10** to the
**CIS MongoDB 8 Benchmark v2.0.0**, in the [ansible-lockdown](https://github.com/ansible-lockdown) style.
Every task maps to one CIS recommendation. Manual recommendations only report; Automated ones fix drift and report
what they changed. Platforms: RHEL and compatible (Rocky, Alma, Oracle with RHCK), x86_64.

## Requirements

- ansible-core **2.16.1+**. RHEL 8 targets must be run from ansible-core **2.16.x** (its Python 3.6 is not supported by 2.17+).
- Collections on the control node: `ansible-galaxy collection install -r requirements.yml` (`community.mongodb` for `mongodb_shell`; `community.general` 11.x for SELinux labels, the last line that supports ansible-core 2.16).
- On the database servers: `mongosh` comes with the `mongodb-enterprise` package. With SELinux enabled, 6.1 and the SELinux extra install `policycoreutils-python-utils`.
- MongoDB Enterprise 8.0 installed, or `mongodb8_cis_install: true` to add the official Enterprise repo and install it.

## Quick start

```bash
ansible-galaxy collection install -r requirements.yml
cat > site.yml <<'EOF'
- hosts: mongodb
  become: true
  roles:
    - mongodb8_cis
EOF
ansible-playbook -i inventory site.yml --check --diff   # read-only preview
ansible-playbook -i inventory site.yml                  # remediate; a second run reports changed=0
```

## Choosing what runs

A rule runs when its **section**, its **level** and its **rule toggle** are all `true`. Every variable is documented in
[`defaults/main.yml`](defaults/main.yml). Set switches in `group_vars`/`host_vars` as **unquoted** `true`/`false`:
`"false"` in quotes is text, which stops the run on ansible-core 2.19+ and counts as *true* on 2.16. On the command
line use JSON, `-e '{"mongodb8_cis_level_2": true}'`, not `-e mongodb8_cis_level_2=true`. Example `group_vars/mongodb.yml`:

```yaml
# Level 2 profile (Level 2 extends Level 1)
mongodb8_cis_level_1: true
mongodb8_cis_level_2: true

# Skip one rule or a whole section
mongodb8_cis_rule_6_3: false
mongodb8_cis_section7: false

# Disruptive rules: off by default, turn on each one when ready (see below)
mongodb8_cis_rule_2_1: true
mongodb8_cis_rule_4_3: true
mongodb8_cis_rule_6_1: true
mongodb8_cis_admin_user: siteAdmin
mongodb8_cis_admin_password: "{{ vault_mongodb_admin_password }}"
mongodb8_cis_tls_certificate_key_file: /etc/pki/mongodb/server.pem
mongodb8_cis_tls_ca_file: /etc/pki/mongodb/ca.pem
mongodb8_cis_port: 27100
```

## Audit only vs remediate

| Goal | Command |
|------|---------|
| Preview every change, touch nothing | `--check --diff` |
| Report rules only (Manual rules, reports) | `--tags audit` |
| One rule / skip one rule | `--tags rule_5.1` / `--skip-tags rule_6.1` |
| By level | `--tags level1` or `--tags level1,level2` |

Reports are printed as `<ID> PASS`, `FAIL`, `REVIEW` or `NOT APPLICABLE` with the value found.

## Disruptive rules (off by default)

Their rule toggles are `false` by default. Turn each one on by name, after preparing what it needs and the clients.

| Rule | Change | Needs | Breaks |
|------|--------|-------|--------|
| 2.1 | Creates the admin user (root@admin), then `security.authorization: enabled` | `mongodb8_cis_admin_user`, `_password` (Vault; no spaces, quotes or backslashes) | Clients without credentials |
| 2.2 | `enableLocalhostAuthBypass: false` | At least one user (2.1) | The localhost login without a user |
| 4.3 | `net.tls.mode: requireTLS` | PEM files on the host: `mongodb8_cis_tls_certificate_key_file`, `_tls_ca_file` | Clients without TLS |
| 4.4 | `net.tls.FIPSMode: true` | TLS (4.3) | SCRAM-SHA-1 and non-FIPS ciphers |
| 6.1 | Non-default `net.port`; with SELinux enabled, labels it `mongod_port_t` | `mongodb8_cis_port` (1024–65535) | Every connection string using 27017; firewall rules |

Other rules that change `mongod.conf` and restart `mongod` once: 4.1/4.2 (only when TLS is on), 5.1 (adds `auditLog`,
default `syslog`, only when missing), 5.3, 5.4; opt-in site decisions 6.3 (`mongodb8_cis_javascript_needed: false`)
and 7.2 (`mongodb8_cis_fix_db_path_permissions: true`). Every write keeps a backup of `mongod.conf`.

## Optional extra: SELinux confinement (not a CIS recommendation)

`mongodb8_cis_selinux_policy: true` (tag `selinux_policy`) confines `mongod` with MongoDB's own policy module, following
MongoDB's install docs:

- RHEL 9/10: builds MongoDB's module on the host (installs `selinux-policy-devel`, `make`, `checkpolicy`) and loads it.
- RHEL 8: the base policy already confines `mongod` (`mongod_t`); MongoDB's module does not build there, so only labels are added.
- All: labels a non-default `dbPath`, log directory and port, restores file labels, restarts `mongod` when something changed.

A confined `mongod` can only use labelled paths and ports. Try it on a non-production host first.

## Layout

`tasks/section_<N>/main.yml` imports one file per rule, `cis_<N>.<M>.yml` (Lockdown layout). `tasks/selinux.yml` is the
optional extra; `files/selinux/` holds MongoDB's policy sources (GPL-2.0-or-later, from github.com/mongodb/mongodb-selinux).

## Notes

- With authorization on, the role logs in as `mongodb8_cis_admin_user` for its Section 2/3 reads. The password is
  briefly visible in the server's process list while `mongosh` runs.
- Only MongoDB 8.0 is supported; prelim stops on any other installed or requested version.
- Design decisions, platform facts and test evidence: [`docs/`](docs/).

## License

MIT
