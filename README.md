# mongodb8_cis

Ansible role that hardens **MongoDB 8.0 Enterprise** (`mongod`) on **RHEL 8, 9 and 10** to the
**CIS MongoDB 8 Benchmark v2.0.0**, in the [ansible-lockdown](https://github.com/ansible-lockdown) style.
Every task is one CIS recommendation. Settings are collected during the run and written to `/etc/mongod.conf` once
(with a backup), followed by one restart. Manual rules print a report for a person to review.

Platforms: RHEL and compatible (Rocky, Alma, Oracle); a standalone `mongod`, or a replica set / shard member (2.3).

## Requirements

- ansible-core **2.16.x** on the control node: the last version that can manage RHEL 8.
- Collections: `ansible-galaxy collection install -r requirements.yml` (`community.mongodb`, `community.general`).
- MongoDB Enterprise 8.0 installed, or `mongodb8_cis_install: true` to install it from MongoDB's repository.

## Quick start

```bash
ansible-galaxy collection install -r requirements.yml
cat > site.yml <<'EOF'
- hosts: mongodb
  become: true
  roles:
    - mongodb8_cis
EOF
ansible-playbook -i inventory site.yml      # a second run reports changed=0
```

## What runs by default

| Setting | Default | Meaning |
|---------|---------|---------|
| `mongodb8_cis_level_1` | `true` | CIS Level 1 |
| `mongodb8_cis_level_2` | `false` | CIS Level 2: stricter. Set **both** to `true` for Level 2 |
| `mongodb8_cis_section1` .. `mongodb8_cis_section7` | `true` | the 7 benchmark sections |
| `mongodb8_cis_rule_<id>` | `true` | one switch per CIS rule, e.g. `mongodb8_cis_rule_6_1` for 6.1 |
| site values | `""` / `[]` | settings your organization decides: the rule reports `NOT APPLIED` until set |

**The full benchmark is on, including rules that change what clients can do.** Switch off what your organization does
not accept; that list is also your compliance exception list. Every setting, with a one-line explanation, is in
[`defaults/main.yml`](defaults/main.yml).

## Your settings (`group_vars`)

Never edit the tasks. Write switches unquoted (`true` / `false`). Example `group_vars/mongodb.yml`:

```yaml
# Install (only when MongoDB is missing)
mongodb8_cis_install: true

# Level 2 as well
mongodb8_cis_level_2: true

# Admin account (2.1)
mongodb8_cis_admin_user: dbadmin
mongodb8_cis_admin_password: "Change.Me.1"

# TLS (4.3): files on the control node, and where they go on the server
mongodb8_cis_tls_certificate_key_src: "files/pki/{{ inventory_hostname }}.pem"
mongodb8_cis_tls_ca_src: files/pki/ca.pem
mongodb8_cis_tls_certificate_key_file: /etc/pki/mongodb/server.pem
mongodb8_cis_tls_ca_file: /etc/pki/mongodb/ca.pem

# Non-default port (6.1)
mongodb8_cis_port: 27100

# Exceptions: rules this organization does not apply, and why
mongodb8_cis_rule_4_4: false           # clients still use SCRAM-SHA-1
```

## Rules that change what clients can do

On by default. Turn one off with `mongodb8_cis_rule_<id>: false` when your clients can't follow.

| Rule | What clients notice |
|------|---------------------|
| 2.1 | login required; the role creates the admin (role `root`) first |
| 2.2 | no localhost login without a user (applied once a user exists) |
| 2.3 | cluster members authenticate with certificates (needs 4.3; not applicable on a standalone) |
| 4.3 | TLS required: clients need TLS and a certificate signed by your CA |
| 4.4 | FIPS mode: SCRAM-SHA-1 and non-FIPS ciphers stop working (needs 4.3) |
| 6.1 | new port: every connection string and firewall rule using 27017 must change |
| 6.3 | server-side JavaScript off: `$where`, `mapReduce`, `$function` fail. Keep it with `mongodb8_cis_javascript_needed: true` |

Also changed, without client impact: 4.1/4.2 (old TLS versions off), 5.1 (audit log, `syslog` by default), 5.3, 5.4,
6.2 (resource limits), 7.1/7.2 (file permissions).

## Site values

The rule reports `NOT APPLIED` until you set its value.

| Rule | Setting | Example |
|------|---------|---------|
| 2.1 | `mongodb8_cis_admin_user`, `mongodb8_cis_admin_password` | `dbadmin`, no spaces, quotes or backslashes in the password |
| 4.3 | `mongodb8_cis_tls_certificate_key_src`, `mongodb8_cis_tls_ca_src` | server certificate + key in one PEM, and the CA certificate |
| 4.3 | `mongodb8_cis_tls_certificate_key_file`, `mongodb8_cis_tls_ca_file` | `/etc/pki/mongodb/server.pem`, `/etc/pki/mongodb/ca.pem` |
| 6.1 | `mongodb8_cis_port` | `27100` (1024-65535, not 27017) |

A renewed certificate is a new source file: the next run copies it and restarts mongod.

## Manual rules: report, or apply your decision

These print a report. Fill in the setting to let the role apply your decision; empty = report only.

| Rule | Setting | Example |
|------|---------|---------|
| 3.1 | `mongodb8_cis_revoke_admin_roles` | `["admin.badadmin"]`: removes `dbOwner` / `userAdmin` / `userAdminAnyDatabase` |
| 3.4 | `mongodb8_cis_drop_custom_roles` | `["shop.orderReader"]` |
| 3.5 | `mongodb8_cis_revoke_superuser_roles` | `["admin.ops"]` (never the role's own admin) |
| 5.2 | `mongodb8_cis_audit_filter` | `'{ atype: { $in: [ "authenticate", "createUser", "dropUser" ] } }'` |

Names like `admin.badadmin` are printed by the rule's own report. Report only: 1.1 version and patches (upgrade in a
change window), 3.2 accounts and their roles, 3.3 service account, 4.5 encryption at rest (enable it by hand on a new install:
[MongoDB: Configure Encryption](https://www.mongodb.com/docs/manual/tutorial/configure-encryption/)).

## Run and check

| Goal | How |
|------|-----|
| Preview, change nothing | `--check --diff` (shows the `mongod.conf` changes) |
| Reports only, change nothing | `--tags audit --skip-tags patch` |
| Apply | run the playbook without tags |
| Only some rules | `--tags rule_4.3` or `--tags level1` |
| Skip a rule | its switch, `mongodb8_cis_rule_<id>: false` |

Reports are printed as `<ID> PASS`, `FAIL`, `REVIEW`, `NOT APPLICABLE` or `NOT APPLIED` with the value found.
To check: `cat /etc/mongod.conf` shows what the role wrote; `mongosh ... --eval 'db.adminCommand({getCmdLineOpts: 1})'`
shows what mongod runs with.

## License

MIT
