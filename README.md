# mongodb8_cis

Ansible role that hardens **MongoDB 8.0 Enterprise** (`mongod`) on **RHEL 8, 9 and 10** to the
**CIS MongoDB 8 Benchmark v2.0.0**, in the [ansible-lockdown](https://github.com/ansible-lockdown) style.
Every task is one CIS recommendation. Automated rules change `mongod.conf` (written once, one restart);
Manual rules report what a person should review, and some apply a decision you give them.
Platforms: RHEL and compatible (Rocky, Alma, Oracle); standalone `mongod`, or a replica set / shard member (2.3).

## Requirements

- ansible-core **2.16.x** on the control node: the last version that can manage RHEL 8 (Python 3.6).
  Setup: [`docs/control-node-setup.md`](docs/control-node-setup.md).
- Collections: `ansible-galaxy collection install -r requirements.yml` (`community.mongodb`, `community.general` 11.x).
- MongoDB Enterprise 8.0 installed, or `mongodb8_cis_install: true` (adds MongoDB's Enterprise repo and installs it).

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
| values | `""` / `[]` | settings only your organization knows: the rule reports `NOT APPLIED` until set |

**The role applies the full benchmark, including the rules that change what clients can do.** Switch off what your
organization does not accept; that list is also your compliance exception list. Switches are unquoted `true` / `false`.
Every variable, with a one-line explanation: [`defaults/main.yml`](defaults/main.yml).

## Values the rules need

| Rule | Variable | Example |
|------|----------|---------|
| 2.1 | `mongodb8_cis_admin_user`, `mongodb8_cis_admin_password` | `dbadmin`, `"Change.Me.1"` (no spaces, quotes or backslashes) |
| 4.3 | `mongodb8_cis_tls_certificate_key_src` | `files/pki/{{ inventory_hostname }}.pem`: server certificate + private key in one PEM |
| 4.3 | `mongodb8_cis_tls_ca_src` | `files/pki/ca.pem`: certificate of the CA that signed it |
| 4.3 | `mongodb8_cis_tls_certificate_key_file`, `mongodb8_cis_tls_ca_file` | `/etc/pki/mongodb/server.pem`, `/etc/pki/mongodb/ca.pem`: where they go on the server |
| 6.1 | `mongodb8_cis_port` | `27100` (1024–65535, not 27017) |

The 4.3 files are copied from the control node to the paths you choose, owner mongod, `0600`.
A renewed certificate is a new source file: the next run copies it and restarts mongod.
Optional: `mongodb8_cis_shell_tls_certificate_key_src` + `mongodb8_cis_shell_client_cert_path`, the role's own client certificate, when your server
certificates are not allowed for client use; `mongodb8_cis_cluster_file` (2.3), a separate member certificate.

## Rules that change what clients can do (on by default)

| Rule | What changes | What clients notice | Switch |
|------|--------------|---------------------|--------|
| 2.1 | creates the admin (role `root`), then login is required | clients without a user can't connect | `mongodb8_cis_rule_2_1` |
| 2.2 | no localhost login without a user (only once a user exists) | the "no user yet" localhost login stops | `mongodb8_cis_rule_2_2` |
| 2.3 | cluster members authenticate with x509 (N/A on a standalone; needs 4.3) | members not changed in the same run can't rejoin | `mongodb8_cis_rule_2_3` |
| 4.3 | TLS required | clients without TLS and a certificate signed by your CA can't connect | `mongodb8_cis_rule_4_3` |
| 4.4 | FIPS mode (needs 4.3) | SCRAM-SHA-1 and non-FIPS ciphers stop working | `mongodb8_cis_rule_4_4` |
| 6.1 | non-default port (labelled for SELinux) | every connection string and firewall rule using 27017 | `mongodb8_cis_rule_6_1` |
| 6.2 | CIS resource limits as a systemd drop-in | none (restart once) | `mongodb8_cis_fix_resource_limits` |
| 6.3 | server-side JavaScript off | `$where`, `mapReduce`, `$function` fail | `mongodb8_cis_javascript_needed: true` keeps it |
| 7.1 / 7.2 | key files `0600`, dbPath `0770`, owner mongod | none | `mongodb8_cis_fix_key_file_permissions` / `_db_path_permissions` |

Also changed, without client impact: 4.1/4.2 (TLS 1.0/1.1 off, when TLS is on), 5.1 (audit log, `syslog` by default,
only when none is set), 5.3, 5.4. `mongod.conf` is written once with a backup, and mongod restarts once.

## Manual rules: report, or apply your decision

Empty = report only. The names come from the rule's own report.

| Rule | Variable | Example |
|------|----------|---------|
| 3.1 | `mongodb8_cis_revoke_admin_roles` | `["admin.badadmin"]`: removes `dbOwner` / `userAdmin` / `userAdminAnyDatabase` on admin |
| 3.2 | `mongodb8_cis_users` | `[{user: appuser, db: shop, password: "App.Pass.1", roles: [{role: readWrite, db: shop}]}]`: creates missing accounts, adds missing roles, never removes |
| 3.4 | `mongodb8_cis_drop_custom_roles` | `["shop.orderReader"]`: drops custom roles |
| 3.5 | `mongodb8_cis_revoke_superuser_roles` | `["admin.ops"]`: removes superuser/admin roles; never the role's own admin |
| 5.1 | `mongodb8_cis_audit_log` | `{destination: file, format: BSON, path: /var/log/mongodb/auditLog.bson}` (default `syslog`) |
| 5.2 | `mongodb8_cis_audit_filter` | `'{ atype: { $in: [ "authenticate", "createUser", "dropUser" ] } }'` |

Report only (a person decides): 1.1 version and patches, 3.3 service account, 4.5 encryption at rest.
Hand fixes: [`docs/manual-remediation.md`](docs/manual-remediation.md).

## Run one rule, skip one rule

| Goal | Command |
|------|---------|
| Only some rules | `--tags rule_4.3` or `--tags level1` |
| Skip a rule this run | `--skip-tags rule_6.1` (to skip it for good, use its switch) |
| Reports only, change nothing | `--tags audit --skip-tags patch` |

Reports are printed as `<ID> PASS`, `FAIL`, `REVIEW`, `NOT APPLICABLE` or `NOT APPLIED` with the value found.

## Check the result

```bash
cat /etc/mongod.conf                                        # what the role wrote
mongosh --port 27100 --tls --tlsCAFile /etc/pki/mongodb/ca.pem \
  --tlsCertificateKeyFile /etc/pki/mongodb/<host>.pem -u dbadmin --authenticationDatabase admin \
  --eval 'db.adminCommand({getCmdLineOpts: 1}).parsed'      # what mongod runs with
```

How to prove a host is compliant, rule by rule: [`docs/compliance-test.md`](docs/compliance-test.md).

## Layout

`tasks/main.yml` → `prelim.yml` (checks, opt-in `install.yml`, reads `mongod.conf`) → `section_1..7/` (one file per CIS
rule; config rules add their setting to one fact) → `post.yml` (writes `mongod.conf` once). Design decisions, platform
facts and test evidence: [`docs/`](docs/).

## License

MIT
