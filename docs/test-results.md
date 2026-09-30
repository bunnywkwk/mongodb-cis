# Test results — `mongodb_cis`

Screenshots in [evidence-images/](evidence-images/). Control node: ansible-core 2.20.7 (RHEL 10), 2.16.x el8 env (RHEL 8).
Test project: `~/mongodb-test` (inventory: rhel10 = 192.168.20.40, rhel8 = 192.168.20.50).

## Prelim + install (2026-09-30)

| # | OS | VM state | Command | Result | Image |
|---|----|----------|---------|--------|-------|
| 01 | RHEL 10.2 | base (no MongoDB) | `--check`, install off | ✅ clean stop "not installed", `failed=0` | [01](evidence-images/01-rhel10-base-not-installed-clean-stop.png) |
| 02 | RHEL 10.2 | MongoDB installed by hand | `--check` | ✅ 8.0.32, `/etc/mongod.conf`, `/var/lib/mongo`, `/var/log/mongodb/mongod.log`; `changed=0` | [02](evidence-images/02-rhel10-existing-mongodb-detected.png) |
| 03 | RHEL 8 | MongoDB installed by hand | `--check` (el8 env) | ✅ same values; `changed=0` | [03](evidence-images/03-rhel8-existing-mongodb-detected.png) |
| 04 | RHEL 10.2 | base | `-e mongodb_cis_install=true` | ✅ installed and started; `changed=4` (key, repo, packages, service) | [04](evidence-images/04-rhel10-install-first-run-changed4.png) |
| 05 | RHEL 10.2 | after 04 | same, rerun | ✅ **idempotent**, `changed=0` | [05](evidence-images/05-rhel10-install-rerun-idempotent-changed0.png) |
| 06 | RHEL 8 | MongoDB installed by hand | `-e mongodb_cis_install=true` (el8 env) | ✅ `changed=0`: hand install matches the role; **`dnf` works on RHEL 8 via 2.16** (D15) | [06](evidence-images/06-rhel8-existing-mongodb-install-true-changed0.png) |
| 07 | RHEL 8 | MongoDB removed | `-e mongodb_cis_install=true` (el8 env) | ✅ installed; `changed=3` (repo, packages, service; the signing key was still in rpm) | [07](evidence-images/07-rhel8-install-after-removal-changed3.png) |

Confirms: platform-notes paths/user (not the docs page's `/var/lib/mongodb`), D1 (no OS branches), D15 (RHEL 8 via 2.16), D19 (`| bool`).

## Planned: acceptance test "as a real user" (after all rules are built)

Act as a DBA who must harden MongoDB to CIS, using only the README + `defaults/main.yml`, never the tasks:
1. Pick a profile (Level 1, then Level 2) and set site values (admin user, TLS certs, port, decisions like 6.3) in `group_vars`.
2. Run `--check --diff` → review → real run → second run `changed=0`, on RHEL 8, 9, 10.
3. Walk the CIS grid (D16): default install / non-hardened / hardened. Compare each rule's result with the benchmark's Audit Procedure.
4. Record gaps in the README or role and screenshots here.
