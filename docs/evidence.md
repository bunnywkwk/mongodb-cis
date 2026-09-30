# Evidence log: `mongodb_cis`

Proof, captured by the user, for each build and test step of this role. Rules for capturing
(naming, what to screenshot, no secrets) are in [../../docs/evidence.md](../../docs/evidence.md#how-to-add-evidence).
Screenshots go in `mongodb_cis/docs/evidence/<ID>_<host>_<YYYYMMDD>.png`.

Test hosts: fill in once, then refer to them by label.

| Label | Hostname | OS release (`cat /etc/redhat-release`) | SELinux (`getenforce`) |
|-------|----------|-----------------------------------------|------------------------|
| rhel8 | | | |
| rhel9 | | | |
| rhel10 | | (upgraded to 10.2) | |

## Environment

### E-MDB-01: test hosts: OS release and SELinux mode
- **Proves:** which OS each test ran on, and that SELinux is enforcing (a project requirement).
- **Command (each host):** `hostname; cat /etc/redhat-release; getenforce`
- **Expected:** RHEL 8.10 / 9.x / 10.2; `Enforcing`
- **Date:**
- **Result:**
- **Screenshots:**

### E-MDB-02: MongoDB state before the role (RHEL 9, installed by hand)
- **Proves:** the "default installation" baseline for the CIS test grid (design-decisions D16), and that the real package matches platform-notes (user `mongod`, `/var/lib/mongo`, shipped `mongod.conf`).
- **Command:** `rpm -qa 'mongodb*'; systemctl is-active mongod; cat /etc/mongod.conf; ps -o user=,cmd= -C mongod`
- **Expected:** `mongodb-org-server-8.0.x`, `active`, `mongod.conf` as in platform-notes, process user `mongod`
- **Date:**
- **Result:**
- **Screenshot:**

## Build (lint results per file)

After typing each file, run from the repo root:
`yamllint mongodb_cis && ansible-lint mongodb_cis`.
Record one entry per file, in build order.

### E-MDB-B01: `meta/main.yml`
- **Proves:** role metadata is valid (schema, lint).
- **Command:** `yamllint mongodb_cis && ansible-lint mongodb_cis`
- **Expected:** no errors
- **Date:**
- **Result:**
- **Screenshot:**

## Tests (added when the role runs: CIS grid D16, idempotence, check mode)

_Entries added as each test step becomes possible._
