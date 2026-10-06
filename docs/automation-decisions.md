# Automation decisions — why 7 rules have no automated fix

The role implements all 23 recommendations. 16 can change the host (10 Automated rules, 6 Manual rules with an
opt-in site variable). **7 only report: 1.1, 2.3, 3.2, 3.3, 3.4, 3.5, 4.5.** This page is the weighing behind that:
for each rule, what an automated option **would have to look like**, what it would cost, what breaks if it's wrong,
and why it was rejected. Summary: [design-decisions.md](design-decisions.md) D29. Hand fixes:
[manual-remediation.md](manual-remediation.md). Quotes: CIS MongoDB 8 Benchmark v2.0.0.

## The test every rule went through

An option is added only if **all four** answers are "yes":

| # | Question | Fails when |
|---|----------|------------|
| 1 | **One concrete fix:** does CIS's remediation name a setting or command the role can apply as written? | CIS says "appropriate", "review", "necessary" (a judgment), or describes a procedure |
| 2 | **Simple input:** can the site express its decision as one simple value (boolean, string, short list)? | the variable needs nested lists of dicts, or secrets per entry |
| 3 | **Safe if wrong:** is a wrong value limited and reversible? | data loss, admin lockout, apps broken, outage the role can't undo |
| 4 | **Worth it:** does a fresh install fail the rule, or can it drift later? | the default already passes and stays passing |

For contrast, the six Manual rules that **passed** the test, so they got an option:

| Rule | Input | Why it passed |
|------|-------|---------------|
| 3.1 | a list of account names | CIS names the 3 roles and says *"drop them"*; the site only names accounts |
| 5.2 | one filter string | one setting; empty = audit everything |
| 6.2 | a boolean + the CIS values as a dict | fixed CIS values, written as a drop-in; reversible |
| 6.3 | a boolean | one setting; only the site knows if apps need it |
| 7.1 / 7.2 | a boolean | fixed mode and owner; `chmod` is reversible |

---

## 1.1 Ensure the appropriate MongoDB software version/patches are installed

**CIS:** Audit: *"determine if the MongoDB software version complies with your organization's operational needs"*.
Remediation: *"1. Backup the data set. 2. Download the binaries … 3. Shutdown the MongoDB instance. 4. Replace the
existing MongoDB binaries … 5. Restart the MongoDB instance."*

**What an option would look like**
```yaml
mongodb8_cis_upgrade: false              # true = upgrade to the target version
mongodb8_cis_upgrade_version: "8.0.32"   # pin, or "latest"
mongodb8_cis_backup_dir: /backup/mongo   # CIS step 1: where the dump goes (space, credentials, retention)
```
Role work: `mongodump` with the admin login → check free space → `dnf` to the version → restart → wait → verify.

| Question | Answer |
|----------|--------|
| 1. Concrete fix | Partly: the steps are clear, but "appropriate version" is the organisation's call |
| 2. Simple input | ❌ version + backup location + space check + credentials |
| 3. Safe if wrong | ❌ every run with `latest` could upgrade and **restart the database at an unplanned time**; the role can't roll back a bad release |
| 4. Worth it | yes (new patches appear) |

**Verdict:** report only. Upgrades belong in a change window, not in a hardening run that may be scheduled.
**Revisit when:** a separate, deliberately-run upgrade playbook exists; the hardening role should still not upgrade.

---

## 2.3 Ensure authentication is enabled in the sharded cluster

**CIS:** *"clusterAuthMode should be set to x509"*, `clusterFile` and `CAFile` configured, keyFile *"Only for
Development Purpose"*, applied *"on each host"*.

**What an option would look like**
```yaml
mongodb8_cis_cluster_auth_mode: x509
mongodb8_cis_cluster_file: /etc/pki/mongodb/member.pem   # per member
# + inventory groups for config servers, shards and mongos, and rolling-restart order
```
Role work: certificates on every member, `security.clusterAuthMode`, `net.tls.clusterFile`, then a **rolling restart
across the cluster** (`serial: 1`, secondaries first), plus `mongos` instances.

| Question | Answer |
|----------|--------|
| 1. Concrete fix | yes, for a cluster |
| 2. Simple input | ❌ cluster topology, certificates per member, restart order |
| 3. Safe if wrong | ❌ a member that can't authenticate drops out of the cluster |
| 4. Worth it | ❌ **no cluster**: the role targets a standalone `mongod` (design-decisions.md, Scope) |

**Verdict:** reports `NOT APPLICABLE` on a standalone. **Revisit when:** the role supports replica sets/sharding.

---

## 3.2 Ensure that role-based access control is enabled and configured appropriately

**CIS:** *"Verify that the **appropriate** role or roles have been configured for each user."* Remediation:
*"Establish roles … Assign the appropriate privileges to each role … Assign the appropriate users to each role."*

**What an option would look like** (built on 2026-10-04, then reverted: D28 E3)
```yaml
mongodb8_cis_db_users:
  - name: appuser
    database: shop
    password: "{{ vault_appuser_password }}"
    roles:
      - {role: readWrite, db: shop}
      - {role: read, db: reports}
    authentication_restrictions:            # must be repeated, or they are wiped (E2)
      - {clientSource: ["10.0.0.0/24"]}
  - name: reporter
    # ... 4–5 fields per account, for every account in the database
mongodb8_cis_db_users_exclusive: false     # true = drop users not in the list?
```

| Question | Answer |
|----------|--------|
| 1. Concrete fix | ❌ "appropriate" is the site's judgment |
| 2. Simple input | ❌ nested list, a Vault secret per account, every account repeated |
| 3. Safe if wrong | ❌ an app loses a role → outage; `mongodb_user` clears `authenticationRestrictions` on update unless repeated (D28 E2); "exclusive" mode deletes users |
| 4. Worth it | a fresh install has no users |

**Verdict:** report only (every user and its roles). It would turn the hardening role into an account-management tool
whose input only the site's people can write.

---

## 3.3 Ensure that MongoDB is run using a non-privileged, dedicated service account

**CIS:** Audit: `ps -ef | grep -E "mongos|mongod"`. Remediation: *"Create a dedicated user … Set the Database data
files, the keyfile, and the SSL private key files to only be readable by the mongod/mongos user. Set the log files to
only be writable by the mongod/mongos user and readable only by root."*

**What an option would look like**
```yaml
mongodb8_cis_fix_service_account: false    # true = make mongod run as mongodb8_cis_service_user
```
Role work: create the user/group → systemd drop-in `User=`/`Group=` → `chown -R` dbPath, log dir, keyFile, TLS key,
PID/socket paths → SELinux relabel → restart.

| Question | Answer |
|----------|--------|
| 1. Concrete fix | yes |
| 2. Simple input | yes (one boolean) |
| 3. Safe if wrong | ❌ one root-owned file the role doesn't know about (custom paths, sockets) and **mongod won't start** |
| 4. Worth it | ❌ **the RPM already passes**: `User=mongod`, process owner `mongod` (D28 E4) |

**Verdict:** report only (PASS/FAIL). Only a hand-modified install fails, and that repair needs someone who knows which
files the site added ([manual-remediation.md](manual-remediation.md) 3.3).

---

## 3.4 Ensure that each role for each MongoDB database is needed and grants only the necessary privileges

**CIS:** *"Ensure that only **necessary** roles are listed and only the **necessary** privileges are listed for each
role."* Remediation: `revokePrivilegesFromRole` with the role, resource and actions.

**What an option would look like**
```yaml
mongodb8_cis_revoke_role_privileges:
  - role: orderReader
    db: shop
    privileges:
      - resource: {db: shop, collection: orders}
        actions: [remove, update]
```

| Question | Answer |
|----------|--------|
| 1. Concrete fix | ❌ "necessary" is the app team's judgment |
| 2. Simple input | ❌ role + db + resource + actions per entry (3 levels of nesting) |
| 3. Safe if wrong | ❌ the app silently loses an action it needs |
| 4. Worth it | ❌ a fresh install has **no custom roles** (D28 E5) |

**Verdict:** report only (every custom role and its actions).

---

## 3.5 Review Superuser/Admin Roles

**CIS:** *"Superuser roles provide the ability to assign any user any privilege on any database"*; lists `dbOwner`,
`userAdmin`, `userAdminAnyDatabase`, `readWriteAnyDatabase`, `dbAdminAnyDatabase`, `clusterAdmin`, `hostManager` and
`root`. Remediation: `revokeRolesFromUser`.

**What an option would look like** (the 3.1 pattern, extended)
```yaml
mongodb8_cis_revoke_superuser_roles:       # accounts and which of the 8 roles to remove
  - {account: admin.ops, roles: [clusterAdmin, hostManager]}
```

| Question | Answer |
|----------|--------|
| 1. Concrete fix | ❌ the title is **"Review"**; which admins are legitimate is a people decision |
| 2. Simple input | partly (accounts + roles) |
| 3. Safe if wrong | ❌ revoking `root` or `userAdminAnyDatabase` from the wrong account can leave **no admin at all**; CIS 2.1 itself creates the admin with `root` |
| 4. Worth it | a fresh install has only the 2.1 admin |

**Verdict:** report only. Unlike 3.1 (three named roles, *"drop them"*), 3.5 covers `root`, so a mistake is a lockout.
**Revisit when:** needed; it would need a guard "never the 2.1 admin, never the last `root`".

---

## 4.5 Ensure Encryption of Data at Rest

**CIS:** a procedure: *"Generating a master key … keys for each database … Encrypting the database keys with the master
key"*, KMIP *"Recommended"*, local keyfile as the other option, and key rotation.

**What an option would look like** (two variants: [manual-remediation.md](manual-remediation.md) 4.5)
```yaml
mongodb8_cis_enable_encryption: false
mongodb8_cis_encryption_key_file: ""       # variant A: keyfile provided by the site (never generated by the role)
mongodb8_cis_kmip_server: ""               # variant B: KMIP server, port, client cert, CA
mongodb8_cis_kmip_port: 5696
mongodb8_cis_kmip_client_certificate_file: ""
mongodb8_cis_kmip_server_ca_file: ""
```
Role work: refuse unless dbPath is empty → place key / check KMIP → `security.enableEncryption` + key settings →
restart. Existing data: dump → empty dbPath → restart → restore (a migration).

| Question | Answer |
|----------|--------|
| 1. Concrete fix | ❌ a procedure, not a setting (key design, KMIP, rotation) |
| 2. Simple input | ❌ key management infrastructure (KMIP server + certificates) |
| 3. Safe if wrong | ❌ **data loss**: *"MongoDB cannot encrypt existing data"* (E1); a lost key = unreadable data (E5); the keyfile method *"does not meet most regulatory key management guidelines"* (E2) |
| 4. Worth it | yes for regulated data |

**Verdict:** report only; hand procedure with backups in [manual-remediation.md](manual-remediation.md) 4.5.
**Revisit when:** a KMIP server is available (variant B, fresh installs only).

---

## Result

| Rule | Failed questions | Main reason |
|------|------------------|-------------|
| 1.1 | 2, 3 | upgrade = unplanned restart, no rollback |
| 2.3 | 2, 3, 4 | no cluster in scope |
| 3.2 | 1, 2, 3 | per-account input; apps break, restrictions wiped |
| 3.3 | 3, 4 | already passes; a repair can stop mongod starting |
| 3.4 | 1, 2, 3, 4 | app team's judgment; nothing to fix by default |
| 3.5 | 1, 3 | "Review"; can remove the last admin |
| 4.5 | 1, 2, 3 | data loss and key management |

All seven are still CIS-correct: CIS marks 1.1, 3.x and 4.5 **Manual** (a person validates them), and 2.3 doesn't
apply to a standalone. The role reports the facts each Audit asks for, so the reviewer has everything to decide.
