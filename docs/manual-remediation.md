# Manual remediation

Fixes the role **reports but does not apply**, because the right answer depends on your users, apps or data
(why: [design-decisions.md](design-decisions.md) D20, D28 and "Not adopted"). Run the role, read the `REVIEW`/`FAIL` lines,
then fix by hand with the commands below. Run them in `mongosh` as the admin (2.1), on the server, in a change window.

Source: CIS MongoDB 8 Benchmark v2.0.0, Remediation of each rule; MongoDB 8.0 manual (links per rule).

## Rules without an automated fix

How each was weighed (one concrete fix, simple input, safe if wrong, worth it), with what an automated option would
have looked like: [automation-decisions.md](automation-decisions.md) (summary: D29).

All other rules are automated, or have an optional site variable (D20). Risky automated rules (2.1, 2.2, 4.3, 4.4, 6.1)
are off by default but **are** automated: turn them on in `group_vars`.

| Rule | CIS type | Role does | Why no automated fix | Fix by hand |
|------|----------|-----------|----------------------|-------------|
| 1.1 | Manual | reports installed version | an upgrade restarts the database; needs a change window | `dnf update mongodb-enterprise*` in a change window |
| 2.3 | Automated | reports N/A on a standalone | sharded clusters only; needs one keyFile/x509 on every member + rolling restart (out of scope: standalone only, see design-decisions.md Scope) | MongoDB docs: "Deploy Sharded Cluster with Keyfile Authentication" |
| 3.2 | Manual | reports authorization + every user's roles | which roles each account needs is a people decision | [3.2](#32-role-based-access-control) |
| 3.3 | Manual | reports who mongod runs as | RPM default already passes; fixing a root-run mongod means re-owning unknown files | [3.3](#33-non-privileged-service-account) |
| 3.4 | Manual | reports custom roles + actions | only the app team knows which privilege is unneeded | [3.4](#34-each-role-grants-only-the-necessary-privileges) |
| 3.5 | Manual | reports superuser/admin accounts | which admins are legitimate is a people decision | [3.5](#35-review-superuser-admin-roles) |
| 4.5 | Manual | reports encryption at rest | only on an empty dbPath + key management (KMIP/keyfile) design | [4.5](#45-encryption-of-data-at-rest) |

## Section 3 — Authorization

| Rule | Role reports | Automated? | Fix by hand |
|------|--------------|-----------|-------------|
| 3.1 | accounts with `dbOwner` / `userAdmin` / `userAdminAnyDatabase` in admin | optional: `mongodb8_cis_revoke_admin_roles` | [3.1](#31-least-privilege-for-database-accounts) |
| 3.2 | authorization state + every user's roles | no | [3.2](#32-role-based-access-control) |
| 3.3 | who mongod runs as | no (RPM default passes) | [3.3](#33-non-privileged-service-account) |
| 3.4 | every custom role with its actions | no | [3.4](#34-each-role-grants-only-the-necessary-privileges) |
| 3.5 | users with superuser/admin roles | no | [3.5](#35-review-superuser-admin-roles) |

### 3.1 Least privilege for database accounts

Either list the account in `mongodb8_cis_revoke_admin_roles` (e.g. `["admin.badadmin"]`), or by hand:

```js
db.getSiblingDB("admin").revokeRolesFromUser("badadmin", [ { role: "userAdminAnyDatabase", db: "admin" } ])
```

### 3.2 Role-based access control

1. Turn on authorization: rule 2.1 (`mongodb8_cis_rule_2_1: true` + admin user/password).
2. Give each app account only the role it needs, on its own database:

```js
use shop
db.createUser({ user: "appuser", pwd: passwordPrompt(), roles: [ { role: "readWrite", db: "shop" } ] })   // new account
db.grantRolesToUser("appuser", [ { role: "read", db: "reports" } ])                                     // add a role
db.revokeRolesFromUser("appuser", [ { role: "dbOwner", db: "shop" } ])                                  // remove a role
db.getUser("appuser")                                                                                   // check
```

| Account | Typical role |
|---------|--------------|
| application | `readWrite` on its database |
| reporting | `read` on its database |
| one admin (2.1) | `root` |

MongoDB has no per-user privileges (only roles), so CIS step 4 ("remove individual privileges") does not apply.
Docs: <https://www.mongodb.com/docs/manual/tutorial/manage-users-and-roles/>

### 3.3 Non-privileged service account

Only needed when the role reports `3.3 FAIL` (mongod runs as root). The RPM default (`User=mongod`) passes.

```bash
systemctl show mongod -p User                      # empty or root = FAIL
systemctl cat mongod                               # shows which file sets User= (unit or a drop-in)
sudo systemctl stop mongod
sudo systemctl edit mongod                         # set: [Service] User=mongod  Group=mongod (or fix the file found above)
sudo chown -R mongod:mongod /var/lib/mongo /var/log/mongodb    # use your dbPath / log path from mongod.conf
sudo chown mongod:mongod <keyFile> <TLS key> <CA file>          # if set in mongod.conf (see 7.1)
sudo systemctl daemon-reload && sudo systemctl start mongod
ps -o user,cmd -C mongod                            # expect: mongod
```

If mongod does not start, `journalctl -u mongod` names the file it cannot open; `chown` it and start again.

### 3.4 Each role grants only the necessary privileges

On a fresh install there are no custom roles (the role reports `0 user-defined role(s)`). For each custom role the
report lists, ask the app owner which actions are still needed, then remove the rest:

```js
db.getSiblingDB("shop").getRole("orderReader", { showPrivileges: true })        // see what it grants
db.getSiblingDB("shop").revokePrivilegesFromRole("orderReader", [
  { resource: { db: "shop", collection: "orders" }, actions: [ "remove" ] }
])
db.getSiblingDB("shop").dropRole("oldRole")                                       // role not needed at all
```

Docs: <https://www.mongodb.com/docs/manual/reference/method/db.revokePrivilegesFromRole/>

### 3.5 Review superuser/admin roles

Keep `root` on the one admin account; remove admin roles from every other account the report lists:

```js
db.getSiblingDB("admin").revokeRolesFromUser("ops", [ { role: "clusterAdmin", db: "admin" } ])
```

Docs: <https://www.mongodb.com/docs/manual/reference/built-in-roles/>

## Section 4 — Data Encryption

### 4.5 Encryption of data at rest

The role reports `security.enableEncryption` and the key management in use. It does **not** enable it (see the
evidence below). CIS recommends KMIP (an external key server); a local keyfile is the other option.

**By hand, standalone, local keyfile** (test first; take a backup; plan downtime):

```bash
mongodump --uri "mongodb://<admin>@localhost:27017/?authSource=admin" --out /backup/pre-encryption   # 1. back up all data
sudo systemctl stop mongod
sudo mkdir -p /etc/pki/mongodb
sudo sh -c 'openssl rand -base64 32 > /etc/pki/mongodb/encryption.key'                                # 2. master key
sudo chown mongod:mongod /etc/pki/mongodb/encryption.key && sudo chmod 600 /etc/pki/mongodb/encryption.key
sudo cp /etc/pki/mongodb/encryption.key /<safe place off this server>/                                 # 3. key backup: no key = no data
sudo mv /var/lib/mongo /var/lib/mongo.unencrypted && sudo install -d -o mongod -g mongod -m 0750 /var/lib/mongo  # 4. empty dbPath
# 5. /etc/mongod.conf:
#   security:
#     enableEncryption: true
#     encryptionKeyFile: /etc/pki/mongodb/encryption.key
sudo systemctl start mongod
mongorestore --uri "mongodb://<admin>@localhost:27017/?authSource=admin" /backup/pre-encryption          # 6. restore (data is encrypted on write)
sudo grep -i "encryption" /var/log/mongodb/mongod.log | tail -3                                          # 7. check
```

Delete `/var/lib/mongo.unencrypted` and the dump only after checking the data, and store them encrypted until then.
With KMIP use `security.kmip.serverName`, `kmip.port`, `kmip.clientCertificateFile`, `kmip.serverCAFile` instead of
`encryptionKeyFile`. Docs: <https://www.mongodb.com/docs/manual/tutorial/configure-encryption/>

**Why the role does not automate it (evidence):**

| # | Source | Says | Impact on automation |
|---|--------|------|----------------------|
| E1 | MongoDB manual, *Configure Encryption* | *"MongoDB cannot encrypt existing data. When you enable encryption with a new key, the MongoDB instance cannot have any pre-existing data."* | On any server with data the role would have to dump, empty dbPath, restart and restore: a data migration, not a setting |
| E2 | same page | *"Using the keyfile method does not meet most regulatory key management guidelines … The safe management of the keyfile is critical."* | The only method the role could set up alone is the one MongoDB says is weak for compliance |
| E3 | same page | *"Key rotation is not available for local key management."* / a key manager *"is recommended over the local key management"* | Proper setup needs a KMIP server, client certificates and key ownership: infrastructure outside the role |
| E4 | CIS MongoDB 8 v2.0.0, 4.5 | **Manual**; remediation is a procedure (master key, database keys, KMIP recommended, rotation) | No single concrete setting to apply (D20) |
| E5 | key loss | the keyfile is the only way to read the data files | A role bug or a missing key backup = **all data unreadable** |

**Possible future implementation** (not built; weigh before adding):

| Option | Variables | Safe when | Still open |
|--------|-----------|-----------|------------|
| A. Keyfile on a **fresh install only** | `mongodb8_cis_enable_encryption: false`, `mongodb8_cis_encryption_key_file: ""` (key provided by the site, never generated by the role) | dbPath is empty (role checks no data files and stops otherwise) | key backup is the site's job; keyfile does not meet most regulations (E2) |
| B. KMIP on a fresh install | `mongodb8_cis_kmip_server`, `_kmip_port`, `_kmip_client_cert`, `_kmip_ca_file` | dbPath empty and KMIP reachable | needs a KMIP server in the lab to test |
| C. Existing data | — | never automatic | dump/restore or initial sync stays a manual change (E1) |

