# Test results — `mongodb8_cis`

Screenshots in [evidence-images/](evidence-images/). Tests 01–07 ran under the old role name `mongodb_cis` (renamed to `mongodb8_cis`, D23), so screenshots show the old name. Control node: ansible-core 2.20.7 (RHEL 10), 2.16.x el8 env (RHEL 8).
Test project: `~/mongodb-test` (inventory: rhel10 = 192.168.20.40, rhel8 = 192.168.20.50).

## Prelim + install (2026-09-30)

| # | OS | VM state | Command | Result | Image |
|---|----|----------|---------|--------|-------|
| 01 | RHEL 10.2 | base (no MongoDB) | `--check`, install off | ✅ clean stop "not installed", `failed=0` | [01](evidence-images/01-rhel10-base-not-installed-clean-stop.png) |
| 02 | RHEL 10.2 | MongoDB installed by hand | `--check` | ✅ 8.0.32, `/etc/mongod.conf`, `/var/lib/mongo`, `/var/log/mongodb/mongod.log`; `changed=0` | [02](evidence-images/02-rhel10-existing-mongodb-detected.png) |
| 03 | RHEL 8 | MongoDB installed by hand | `--check` (el8 env) | ✅ same values; `changed=0` | [03](evidence-images/03-rhel8-existing-mongodb-detected.png) |
| 04 | RHEL 10.2 | base | `-e mongodb8_cis_install=true` | ✅ installed and started; `changed=4` (key, repo, packages, service) | [04](evidence-images/04-rhel10-install-first-run-changed4.png) |
| 05 | RHEL 10.2 | after 04 | same, rerun | ✅ **idempotent**, `changed=0` | [05](evidence-images/05-rhel10-install-rerun-idempotent-changed0.png) |
| 06 | RHEL 8 | MongoDB installed by hand | `-e mongodb8_cis_install=true` (el8 env) | ✅ `changed=0`: hand install matches the role; **`dnf` works on RHEL 8 via 2.16** (D15) | [06](evidence-images/06-rhel8-existing-mongodb-install-true-changed0.png) |
| 07 | RHEL 8 | MongoDB removed | `-e mongodb8_cis_install=true` (el8 env) | ✅ installed; `changed=3` (repo, packages, service; the signing key was still in rpm) | [07](evidence-images/07-rhel8-install-after-removal-changed3.png) |

Confirms: platform-notes paths/user (not the docs page's `/var/lib/mongodb`), D1 (no OS branches), D15 (RHEL 8 via 2.16), D19 (`| bool`).

## Pre-VM end-to-end (2026-10-01): Sections 2, 3, 4 against a real mongod

The runs below used the former `mongodb8_cis_disruption_high` switch, removed later the same day (D6); "everything on" now means the five risky rule toggles set to `true`. No VM access, so the role's real `tasks/main.yml`, `section_2/3/4.yml`, `module_defaults` wiring and handlers were run on the workstation against the **real `mongod` 8.0.32 Enterprise binary** (el9 RPM, scratch dbPath, port 37019) and `mongosh` 2.12.0 from the Enterprise repo. Changes for the test only: prelim facts faked (packages, systemd), the restart handler restarts the scratch `mongod`, `owner: root` removed (not run as root). TLS used a throwaway CA and a server cert with EKU `serverAuth,clientAuth`.

| Run | Settings | ansible-core 2.20.7 | ansible-core 2.16.19 |
|-----|----------|---------------------|----------------------|
| A | defaults + Level 2, fresh DB | ✅ `changed=0`; reports 3.1–3.5, 4.1/4.2 FAIL "TLS not enabled", 4.4 FAIL "disruption_high false", 4.5 REVIEW | ✅ same |
| `--check` | everything on, fresh DB | ✅ `changed=5`, `mongod.conf` unchanged (md5) | ✅ same |
| B | everything on (`disruption_high`, admin login, TLS files) | ✅ `changed=7`: user `cisadmin` (root@admin) created, authorization, bypass off, requireTLS, `TLS1_0,TLS1_1`, FIPS; one restart; log *"FIPS 140 mode activated"* | ✅ same |
| C | same as B, rerun | ✅ `changed=0`; DB reads over TLS + client cert + login; 3.2/3.5 list `cisadmin@admin` | ✅ same |

After run B, a plain-TCP `mongosh` was refused, and a TLS client without login got *"Command aggregate requires authentication"*. Still to confirm on the RHEL VMs: systemd restart, SELinux enforcing, real certificates, FIPS on RHEL 8 and 10.

## SELinux extra in containers (2026-10-01)

`tasks/selinux.yml` run with SELinux status faked as enabled and `semodule -n` (policy store only; nothing loaded into the workstation kernel). The label and `restorecon` tasks were left out: they need SELinux active.

| Image | ansible-core | Run 1 | Run 2 | Store after |
|-------|--------------|-------|-------|-------------|
| Rocky 9 | 2.20.7 | ✅ `changed=4` (packages, sources, build, load) + restart notified | ✅ `changed=0` | `200 mongodb`, `100 mongodb` |
| AlmaLinux 10.2 | 2.20.7 | ✅ `changed=4` | ✅ `changed=0` | `200 mongodb` |
| Rocky 8.10 | 2.16.19 | ✅ `changed=1` (`policycoreutils-python-utils` only) + RHEL 8 message | ✅ `changed=0` | `100 mongodb` (base) |

## Lab runs T1–T10 on KVM VMs (2026-10-01)

Full log, screenshots and the lab setup: test repo `github.com/bunnywkwk/mongodb8-cis-test`
(`docs/test-runs.md`, `docs/test-environment.md`), ansible-core 2.16.19.

| Host (real OS) | Tests | Result |
|----------------|-------|--------|
| RHEL 10 | T4–T10 | ✅ Level 1+2, risky rules (2.1, 2.2, 4.3, 4.4, 6.1), SELinux extra (`mongod_t`); every rerun `changed=0` |
| RHEL 9.8 (named `rhel8-mongo` in the inventory) | T2–T9 | ✅ T2–T8; ❌ T9 found the socket bug T-M7 (fixed, not re-run on RHEL 9) |
| **RHEL 8** | none | **Not tested yet.** The inventory IPs don't match the lab table, so the "rhel8" runs went to the RHEL 9 VM (its mongod log says *"RHEL release 9.8 (Plow)"*) |

**Open:** RHEL 8 T2–T10, RHEL 9 T9–T10 with the fixed role, then the acceptance test below.

## Planned: acceptance test "as a real user" (after all rules are built)

Act as a DBA who must harden MongoDB to CIS, using only the README + `defaults/main.yml`, never the tasks:
1. Pick a profile (Level 1, then Level 2) and set site values (admin user, TLS certs, port, decisions like 6.3) in `group_vars`.
2. Run `--check --diff` → review → real run → second run `changed=0`, on RHEL 8, 9, 10.
3. Walk the CIS grid (D16): default install / non-hardened / hardened. Compare each rule's result with the benchmark's Audit Procedure.
4. Record gaps in the README or role and screenshots here.

## Compliance test (full benchmark, 2026-10-05)

Per-host proof against every CIS Audit procedure: [compliance-test.md](compliance-test.md). Run it on `main` after
the rebuild branch (see [session-handoff.md](session-handoff.md)).

## Compliance test result (2026-10-06, branch `main`)

| Host | Full benchmark run | Rerun | Host checks (CIS Audit) | Verdict |
|------|--------------------|-------|-------------------------|---------|
| RHEL 8 | ok=68 changed=11 failed=0, 1 restart | changed=0 | 18 ✅ · 2.3 ➖ · 3.2/3.5/5.2 👤 reviewed · 4.5 👤 exception · 0 ❌ | **Level 2 compliant** |
| RHEL 9 | ok=68 changed=11 failed=0, 1 restart | changed=0 | same | **Level 2 compliant** |
| RHEL 10 | ok=68 changed=11 failed=0, 1 restart | changed=0 | same | **Level 2 compliant** |

- Accepted exception on all hosts: 4.5 encryption at rest (no key management in the lab; `manual-remediation.md` 4.5).
- Extra (RHEL 8): an application with its own client certificate and `readWrite@shop` user works; no TLS, no client
  certificate, wrong password and other databases are refused.
- Found during the test: FIPS log wording differs by OS (RHEL 8 `FIPS 140-2 mode activated`, RHEL 9/10 `FIPS 140 mode
  activated`, same id 23172); confirm FIPS with `getCmdLineOpts` instead of the log text.
- Full record, logs and 25 screenshots: `mongodb8-cis-test/docs/compliance-test.md`.

