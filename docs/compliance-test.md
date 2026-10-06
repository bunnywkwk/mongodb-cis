# Compliance test — proof that a MongoDB host is hardened to CIS (`main`)

**What this proves:** after `mongodb8_cis` (branch `main`) runs with the full benchmark, the host passes the **Audit**
procedure of every CIS MongoDB 8 Benchmark v2.0.0 recommendation, checked **by hand on the host**, not from the role's
own output. Same format as the rebuild branch's `docs/compliance-test.md`, so the two implementations can be compared
rule by rule.

| Item | Value (fill in) |
|------|-----------------|
| Hosts / OS | `rhel8-mongo`, `rhel9-mongo`, `rhel10-mongo` / `cat /etc/redhat-release` → |
| MongoDB | `mongod --version` → |
| Role | branch `main`, commit → (`git log --oneline -1` in the installed role) |
| Profile | `mongodb8-cis-test/profiles/c3-full-benchmark.yml` (Level 1 + 2, risky rules on, site decisions) |
| Date / tester | |

## How `main` differs from the rebuild (what to expect)

Both implement the same 23 recommendations with the same task titles, levels, tags, toggles and site-decision
variables (3.1, 5.2, 6.2, 6.3, 7.1, 7.2; ported to `main` in `da298cf`). Checked against the PDF, 2026-10-05. The only
difference is **how the database is read**:

| | `main` | rebuild |
|---|---|---|
| Database reads (2.1, 2.2, 3.x) | `mongodb_shell` with CIS's own queries on `admin.system.users`: **every** user is seen | `mongodb_info` per database: users of databases without data are missed (3.1 can show a false PASS) |
| Software on the DB server | none extra (`mongosh` comes with MongoDB) | Python 3.12 venv + pymongo |

Everything else (role output, hand checks, expected results) is the same, so the per-rule table below applies to both.

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

Fresh RHEL 8, 9 and 10 snapshots (no MongoDB). In `mongodb8-cis-test`, point the role at `main` first:

```bash
# requirements.yml: version: main   (back to mongodb8-cis-rebuild for the rebuild test)
ansible-galaxy install -r requirements.yml -p roles --force                                   # S0 role = main
ansible mongodb -m setup -a 'filter=ansible_distribution*'                                     # S1 real OS per host
ansible-playbook playbooks/site.yml -e @profiles/c2-level2-defaults.yml                        # S2 install + defaults
ansible-playbook playbooks/prep-tls.yml                                                        # S3 TLS files
ansible-playbook playbooks/site.yml -e @profiles/c3-full-benchmark.yml --force-handlers        # S4 full benchmark
ansible-playbook playbooks/site.yml -e @profiles/c3-full-benchmark.yml                         # S5 rerun: changed=0
```

Add `--limit rhel9-mongo` (etc.) to run one host at a time. Then SSH to each host and run the checks below. Set this
once **in the same SSH session** (`echo "$M"` must print it; mongosh asks the admin password each time):

```bash
M='sudo mongosh --quiet --port 27100 --tls --tlsCAFile /etc/pki/mongodb/ca.pem --tlsCertificateKeyFile /etc/pki/mongodb/server.pem --tlsAllowInvalidHostnames -u frqadminDB --authenticationDatabase admin'
```

Screenshots: `docs/evidence-images/NN-<os>-<rule>-<result>.png` (next free number: 08), listed in the last column.

## Run evidence

| Step | Expected | RHEL 8 | RHEL 9 | RHEL 10 | Evidence |
|------|----------|--------|--------|---------|----------|
| S0 | `mongodb8_cis (main) was installed successfully` | | | | |
| S1 | `ansible_distribution_major_version` = 8 / 9 / 10 on the right host | | | | |
| S2 | `failed=0` | | | | |
| S3 | `failed=0`, files in `/etc/pki/mongodb/` | | | | |
| S4 | `failed=0`, one `Restart mongod` | | | | |
| S5 | `changed=0 failed=0` (idempotent) | | | | |

## Per-rule checks on the host (CIS Audit procedure, RHEL commands)

✅ pass · ➖ not applicable · 👤 reviewed by a person (Manual rule: record the decision).

| ID | L | Type | Check on the host | Pass when | R8 | R9 | R10 | Evidence |
|----|---|------|-------------------|-----------|----|----|-----|----------|
| 1.1 | 1 | M | `mongod --version \| head -1`; `dnf list --showduplicates mongodb-enterprise-server \| tail -1` | installed = latest 8.0.x (or a documented reason) | | | | |
| 2.1 | 1 | A | `grep -A3 '^security' /etc/mongod.conf`; `sudo mongosh --quiet --port 27100 --tls --tlsCAFile /etc/pki/mongodb/ca.pem --tlsCertificateKeyFile /etc/pki/mongodb/server.pem --tlsAllowInvalidHostnames --eval 'db.adminCommand({listDatabases:1})'` | `authorization: enabled`; command **without** login fails: *requires authentication* | | | | |
| 2.2 | 1 | A | `grep -A2 '^setParameter' /etc/mongod.conf` | `enableLocalhostAuthBypass: false` | | | | |
| 2.3 | 2 | A | `grep -cE 'clusterRole\|clusterAuthMode' /etc/mongod.conf` | `0` → ➖ standalone, no cluster | | | | |
| 3.1 | 1 | M | `$M --eval 'db.getSiblingDB("admin").system.users.find({"roles.role":{$in:["dbOwner","userAdmin","userAdminAnyDatabase"]},"roles.db":"admin"}).toArray()'` | `[]` | | | | |
| 3.2 | 1 | M | `$M --eval 'db.getSiblingDB("admin").system.users.find({},{_id:1,roles:1}).toArray()'` | every user has only the roles it needs (lab: `frqadminDB` → `root@admin`) | | | | |
| 3.3 | 1 | M | `ps -ef \| grep -E "mongos\|mongod" \| grep -v grep` | owner `mongod`, not root | | | | |
| 3.4 | 1 | M | `$M --eval 'db.getSiblingDB("admin").runCommand({rolesInfo:1, showPrivileges:true}).roles'` | `[]` or only needed custom roles | | | | |
| 3.5 | 2 | M | same as 3.2 | only the one admin holds `root` / admin roles | | | | (3.2) |
| 4.1 | 2 | A | `grep -A8 ' tls:' /etc/mongod.conf` | `disabledProtocols: TLS1_0,TLS1_1` | | | | |
| 4.2 | 1 | A | same as 4.1 | same as 4.1 | | | | (4.1) |
| 4.3 | 1 | A | same file: `mode: requireTLS`, `certificateKeyFile`, `CAFile`; plain `mongosh --port 27100 --eval 'db.runCommand({ping:1})'` | TLS required; plain connection **fails** | | | | |
| 4.4 | 2 | A | `grep FIPSMode /etc/mongod.conf`; `sudo grep -i fips /var/log/mongodb/mongod.log \| tail -2` | `FIPSMode: true`; log shows FIPS mode active | | | | |
| 4.5 | 2 | M | `grep -A3 enableEncryption /etc/mongod.conf` | lab: not enabled → 👤 accepted exception (no key management in the lab) | | | | |
| 5.1 | 1 | A | `grep -A3 '^auditLog' /etc/mongod.conf`; `sudo journalctl -t mongod -n 3` | `destination: syslog`; audit events in the journal | | | | |
| 5.2 | 2 | M | `grep -A4 '^auditLog' /etc/mongod.conf` | no `filter` = all events audited (or the site's agreed filter) → 👤 | | | | (5.1) |
| 5.3 | 2 | A | `grep -A6 '^systemLog' /etc/mongod.conf` | `quiet: false` | | | | |
| 5.4 | 2 | A | same | `logAppend: true` | | | | (5.3) |
| 6.1 | 1 | A | `grep -A2 '^net' /etc/mongod.conf`; `sudo ss -tlnp \| grep mongod`; `sudo semanage port -l \| grep mongod_port_t` | `port: 27100`; listening on 27100; 27100 labelled | | | | |
| 6.2 | 2 | M | `sudo cat /proc/$(pidof mongod)/limits` | file size, cpu time, address space, resident set: unlimited; open files, processes: 64000 | | | | |
| 6.3 | 2 | M | `grep javascriptEnabled /etc/mongod.conf` | `javascriptEnabled: false` | | | | |
| 7.1 | 1 | M | `sudo ls -l /etc/pki/mongodb/` | key/PEM/CA files `-rw-------` owner `mongod` | | | | |
| 7.2 | 1 | M | `sudo stat -c '%a %U:%G' /var/lib/mongo` | `770 mongod:mongod` | | | | |

## Result (fill in after the checks)

| | RHEL 8 | RHEL 9 | RHEL 10 |
|---|---|---|---|
| ✅ Pass | | | |
| ➖ Not applicable | 2.3 | 2.3 | 2.3 |
| 👤 Reviewed / accepted exception | | | |
| ❌ Fail | | | |

**Verdict:** each host is compliant with CIS MongoDB 8 Benchmark v2.0.0 Level ☐1 ☐2, with the exceptions above.
Signed off by / date:

Commands are taken from each recommendation's *Audit* section in the benchmark PDF (kept locally, never committed),
adapted to RHEL (`grep` on `/etc/mongod.conf` instead of the Ubuntu/Windows examples) and to the lab's TLS + port.
