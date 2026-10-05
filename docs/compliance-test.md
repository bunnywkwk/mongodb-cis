# Compliance test — proof that a MongoDB host is hardened to CIS

**What this proves:** after `mongodb8_cis` runs with the full benchmark, the host passes the **Audit** procedure of
every CIS MongoDB 8 Benchmark v2.0.0 recommendation, checked **by hand on the host**, not from the role's own output.

| Item | Value (fill in) |
|------|-----------------|
| Host / OS | `rhel9-mongo` / `cat /etc/redhat-release` → |
| MongoDB | `mongod --version` → |
| Role | branch `mongodb8-cis-rebuild`, commit → |
| Profile | `mongodb8-cis-test/profiles/c3-full-benchmark.yml` (Level 1 + 2, risky rules on, site decisions) |
| Date / tester | |

## Level 1 vs Level 2 (what "compliant" means)

CIS *Profile Definitions*: **Level 1** items are *"practical and prudent; provide a clear security benefit; and not
inhibit the utility of the technology beyond acceptable means."* **Level 2** *"extends"* Level 1 for environments
*"where security is paramount"* / *"defense in depth"*.

So **Level 1 is the CIS baseline, not "the role's defaults"**. Four Level 1 rules (2.1, 2.2, 4.3, 6.1) need your values
(admin, certificates, port) and can lock clients out, so the role ships them **off**. Running the role with defaults
gives a safe but **not** Level 1 compliant host; turning those rules on with your values gives Level 1 compliance.
This test uses Level 2 (all 23 rules), which includes Level 1.

| Profile | Rules | Compliant when |
|---------|-------|----------------|
| Level 1 | 13: 1.1, 2.1, 2.2, 3.1–3.4, 4.2, 4.3, 5.1, 6.1, 7.1, 7.2 | all 13 pass or are reviewed |
| Level 2 | all 23 (adds 2.3, 3.5, 4.1, 4.4, 4.5, 5.2–5.4, 6.2, 6.3) | all 23 pass, are not applicable, or are reviewed |

## Procedure

Fresh RHEL 9 snapshot (no MongoDB). From `mongodb8-cis-test`:

```bash
ansible rhel9-mongo -m setup -a 'filter=ansible_distribution*'                                  # S1 real OS
ansible-playbook playbooks/site.yml --limit rhel9-mongo -e @profiles/c2-level2-defaults.yml     # S2 install + defaults
ansible-playbook playbooks/prep-tls.yml --limit rhel9-mongo                                     # S3 TLS files
ansible-playbook playbooks/site.yml --limit rhel9-mongo -e @profiles/c3-full-benchmark.yml --force-handlers   # S4 full benchmark
ansible-playbook playbooks/site.yml --limit rhel9-mongo -e @profiles/c3-full-benchmark.yml      # S5 rerun: changed=0
```

Then SSH to the host and run the checks below. Set this once **in the same SSH session** (`echo "$M"` must print it; mongosh asks the admin password each time):

```bash
M='sudo mongosh --quiet --port 27100 --tls --tlsCAFile /etc/pki/mongodb/ca.pem --tlsCertificateKeyFile /etc/pki/mongodb/server.pem --tlsAllowInvalidHostnames -u frqadminDB --authenticationDatabase admin'
```

Screenshots: `docs/evidence-images/NN-rhel9-<rule>-<result>.png` (next free number: 04), listed in the last column.

## Run evidence

| Step | Expected | Result (PLAY RECAP) | Evidence |
|------|----------|---------------------|----------|
| S1 | `ansible_distribution_major_version: "9"` | | 04-rhel9-os-version.png |
| S2 | `failed=0` | | 05-rhel9-c2-defaults.png |
| S3 | `failed=0`, files in `/etc/pki/mongodb/` | | 06-rhel9-prep-tls.png |
| S4 | `failed=0`, one `Restart mongod` | | 07-rhel9-c3-full-benchmark.png |
| S5 | `changed=0 failed=0` (idempotent) | | 08-rhel9-c3-rerun-changed0.png |

## Per-rule checks on the host (CIS Audit procedure, RHEL commands)

✅ pass · ➖ not applicable · 👤 reviewed by a person (Manual rule: record the decision).

| ID | L | Type | Check on the host | Pass when | Result | Evidence |
|----|---|------|-------------------|-----------|--------|----------|
| 1.1 | 1 | M | `mongod --version \| head -1`; `dnf list --showduplicates mongodb-enterprise-server \| tail -1` | installed = latest 8.0.x (or a documented reason) | | 09-rhel9-1.1-version.png |
| 2.1 | 1 | A | `grep -A3 '^security' /etc/mongod.conf`; `sudo mongosh --quiet --port 27100 --tls --tlsCAFile /etc/pki/mongodb/ca.pem --tlsCertificateKeyFile /etc/pki/mongodb/server.pem --tlsAllowInvalidHostnames --eval 'db.adminCommand({listDatabases:1})'` | `authorization: enabled`; command **without** login fails: *requires authentication* | | 10-rhel9-2.1-auth.png |
| 2.2 | 1 | A | `grep -A2 '^setParameter' /etc/mongod.conf` | `enableLocalhostAuthBypass: false` | | 11-rhel9-2.2-bypass.png |
| 2.3 | 2 | A | `grep -cE 'clusterRole\|clusterAuthMode' /etc/mongod.conf` | `0` → ➖ standalone, no cluster | | 12-rhel9-2.3-na.png |
| 3.1 | 1 | M | `$M --eval 'db.getSiblingDB("admin").system.users.find({"roles.role":{$in:["dbOwner","userAdmin","userAdminAnyDatabase"]},"roles.db":"admin"}).toArray()'` | `[]` | | 13-rhel9-3.1-users.png |
| 3.2 | 1 | M | `$M --eval 'db.getSiblingDB("admin").system.users.find({},{_id:1,roles:1}).toArray()'` | every user has only the roles it needs (lab: `frqadminDB` → `root@admin`) | | 14-rhel9-3.2-roles.png |
| 3.3 | 1 | M | `ps -ef \| grep -E "mongos\|mongod" \| grep -v grep` | owner `mongod`, not root | | 15-rhel9-3.3-service-user.png |
| 3.4 | 1 | M | `$M --eval 'db.getSiblingDB("admin").runCommand({rolesInfo:1, showPrivileges:true}).roles'` | `[]` or only needed custom roles | | 16-rhel9-3.4-custom-roles.png |
| 3.5 | 2 | M | same as 3.2 | only the one admin holds `root` / admin roles | | (14) |
| 4.1 | 2 | A | `grep -A8 ' tls:' /etc/mongod.conf` | `disabledProtocols: TLS1_0,TLS1_1` | | 17-rhel9-4.x-tls.png |
| 4.2 | 1 | A | same as 4.1 | same as 4.1 | | (17) |
| 4.3 | 1 | A | same file: `mode: requireTLS`, `certificateKeyFile`, `CAFile`; plain `mongosh --port 27100 --eval 'db.runCommand({ping:1})'` | TLS required; plain connection **fails** | | 18-rhel9-4.3-no-plain.png |
| 4.4 | 2 | A | `grep FIPSMode /etc/mongod.conf`; `sudo grep -i fips /var/log/mongodb/mongod.log \| tail -2` | `FIPSMode: true`; log shows FIPS mode active | | 19-rhel9-4.4-fips.png |
| 4.5 | 2 | M | `grep -A3 enableEncryption /etc/mongod.conf` | lab: not enabled → 👤 accepted exception ([manual-remediation.md](manual-remediation.md) 4.5) | | 20-rhel9-4.5-encryption.png |
| 5.1 | 1 | A | `grep -A3 '^auditLog' /etc/mongod.conf`; `sudo journalctl -t mongod -n 3` | `destination: syslog`; audit events in the journal | | 21-rhel9-5.1-audit.png |
| 5.2 | 2 | M | `grep -A4 '^auditLog' /etc/mongod.conf` | no `filter` = all events audited (or the site's agreed filter) → 👤 | | (21) |
| 5.3 | 2 | A | `grep -A6 '^systemLog' /etc/mongod.conf` | `quiet` absent or `false` | | 22-rhel9-5.x-systemlog.png |
| 5.4 | 2 | A | same | `logAppend: true` | | (22) |
| 6.1 | 1 | A | `grep -A2 '^net' /etc/mongod.conf`; `sudo ss -tlnp \| grep mongod`; `sudo semanage port -l \| grep mongod_port_t` | `port: 27100`; listening on 27100; 27100 labelled | | 23-rhel9-6.1-port.png |
| 6.2 | 2 | M | `sudo cat /proc/$(pidof mongod)/limits` | file size, cpu time, address space, resident set: unlimited; open files, processes: 64000 | | 24-rhel9-6.2-limits.png |
| 6.3 | 2 | M | `grep javascriptEnabled /etc/mongod.conf` | `javascriptEnabled: false` | | 25-rhel9-6.3-js.png |
| 7.1 | 1 | M | `sudo ls -l /etc/pki/mongodb/` | key/PEM/CA files `-rw-------` owner `mongod` | | 26-rhel9-7.1-keyfiles.png |
| 7.2 | 1 | M | `sudo stat -c '%a %U:%G' /var/lib/mongo` | `770 mongod:mongod` | | 27-rhel9-7.2-dbpath.png |

## Result (fill in after the checks)

| | Count | Rules |
|---|---|---|
| ✅ Pass | | |
| ➖ Not applicable | | 2.3 (standalone) |
| 👤 Reviewed / accepted exception | | e.g. 4.5 (no key management in the lab) |
| ❌ Fail | | |

**Verdict:** the host is compliant with CIS MongoDB 8 Benchmark v2.0.0 Level ☐1 ☐2, with the exceptions above.
Signed off by / date:

Commands are taken from each recommendation's *Audit* section in the benchmark PDF (kept locally, never committed),
adapted to RHEL (`grep` on `/etc/mongod.conf` instead of the Ubuntu/Windows examples) and to the lab's TLS + port.
