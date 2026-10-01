# Platform notes — MongoDB 8.0 Enterprise on RHEL 8 / 9 / 10

What the role can rely on per OS, and where each fact comes from.
Status legend: **Verified** = checked against the source listed; **To verify** = must be confirmed on a test VM.

## Sources

| # | Source | Used for |
|---|--------|----------|
| S1 | MongoDB docs, *Install MongoDB Enterprise Edition on Red Hat or CentOS* (v8.0): https://www.mongodb.com/docs/v8.0/tutorial/install-mongodb-enterprise-on-red-hat/ (the `/docs/manual/` page now shows 9.0) | Supported OS, repo definition, install command, SELinux policy |
| S2 | Official Enterprise repo metadata, `https://repo.mongodb.com/yum/redhat/<8|9|10>/mongodb-enterprise/8.0/x86_64/repodata/`, and the signing keys at `https://pgp.mongodb.com/` (read 2026-10-01) | Package availability, latest version per RHEL, key file names |
| S3 | `mongodb-enterprise-server-8.0.32-1.el8`, `-1.el9` and `-1.el10` RPMs from S2 (and `mongodb-org-server-8.0.32-1.el9` for comparison), inspected with `rpm -qp --scripts --requires --conflicts`, `rpm -qplv`, `rpm2cpio`, `mongod --version` (2026-10-01) | Service user, paths, shipped `mongod.conf`, systemd unit, dependencies |
| S4 | MongoDB docs: auditing https://www.mongodb.com/docs/v8.0/core/auditing/ , FIPS https://www.mongodb.com/docs/v8.0/tutorial/configure-fips/ , encryption at rest https://www.mongodb.com/docs/v8.0/core/security-encryption-at-rest/ , parameters https://www.mongodb.com/docs/v8.0/reference/parameters/ , config options https://www.mongodb.com/docs/v8.0/reference/configuration-options/ | Which features are Enterprise-only; defaults; `auditLog` options |
| S5 | `cis-pdf/CIS_MongoDB_8_Benchmark_v2.0.0 PDF.pdf`. Extract page by page (`pdftotext -f N -l N -layout`): a whole-file run stops at an xref error and drops pages (Section 6 = pages 83–91) | Rule text |
| S6 | `cis-spreadsheet/CIS_MongoDB_8_Benchmark_v2.0.0-Certification.xlsx` (sheets `Level 1- MongoDB`, `Level 2 - MongoDB`) | Level, Automated/Manual, Audit Procedure, Default Value per rule |
| S7 | ansible-core support matrix: https://docs.ansible.com/ansible/latest/reference_appendices/release_and_maintenance.html | Target Python per ansible-core version |
| S9 | MongoDB SELinux policy source: https://github.com/mongodb/mongodb-selinux | SELinux module content |
| S8 | EPEL repo metadata, `https://dl.fedoraproject.org/pub/epel/<8|9>/Everything/x86_64/repodata/` (read 2026-09-29) | pymongo availability |

## Installation (S1) — same steps on RHEL 8, 9, 10

S1 lists these as supported (64-bit): RHEL / CentOS Stream / Oracle / Rocky / AlmaLinux **8, 9 and 10**.
Oracle Linux only with the Red Hat Compatible Kernel (RHCK), not UEK (S1).

Repo file `/etc/yum.repos.d/mongodb-enterprise-8.0.repo`. The only difference between OS versions is the number in `baseurl`:

```ini
[mongodb-enterprise-8.0]
name=MongoDB Enterprise Repository
baseurl=https://repo.mongodb.com/yum/redhat/<8|9|10>/mongodb-enterprise/8.0/$basearch/
gpgcheck=1
enabled=1
gpgkey=https://pgp.mongodb.com/server-8.0.asc
```

Install and start (S1):

```bash
sudo yum install -y mongodb-enterprise   # latest 8.0.x
sudo systemctl enable --now mongod
```

`mongodb-enterprise` is a metapackage (S2, `requires` of the 8.0.32 build on el8/9/10):
- `mongodb-enterprise-database` → `-server`, `-mongos`, `-cryptd`, `-database-tools-extra`
- `mongodb-enterprise-tools` → `mongodb-database-tools`
- `mongodb-mongosh`, so `mongosh` is still installed by the main package (D13 unchanged).

**Signing key name per release (S2):** `server-8.0.asc` exists; from 9.0 the file is
`server-9.asc` (`server-9.0.asc` → 404). The 8.0 key (`4B0752C1BCA238C0B4EE14DC41DE058A4E7DCA05`,
*MongoDB 8.0 Release Signing Key*) is the key the 8.0.32 Enterprise RPMs are signed with (`41de058a4e7dca05`).

**Community and Enterprise cannot be installed together (S3).** `mongodb-enterprise-server` declares
`Conflicts: mongodb-org-server` (and the reverse). A host with Community must have it removed first
(`dnf remove 'mongodb-org*'`); data in `/var/lib/mongo` and `/etc/mongod.conf` stay, because both RPMs use the same paths.

**Extra system libraries pulled in by Enterprise (S3, all from the RHEL base repos):** `cyrus-sasl`, `cyrus-sasl-gssapi`,
`cyrus-sasl-plain`, Kerberos (`libgssapi_krb5`), OpenLDAP (`libldap`), `libcurl`. They back the Enterprise features (LDAP/Kerberos auth, KMIP).

**Role consequence:**
- One `ansible.builtin.yum_repository` task. `baseurl` is built from `ansible_facts['distribution_major_version']` and
  `mongodb8_cis_version`, and the key name from `mongodb8_cis_version` (`X.Y` before 9.0, major from 9.0). No per-OS or per-version repo files.
- The CIS rules follow the MongoDB 8 benchmark (S5), so `mongodb8_cis_version: "8.0"` is the tested value.

## MongoDB version covered — CIS MongoDB 8 Benchmark v2.0.0 (S5)

- S5 covers MongoDB **8.x**. The role supports `mongodb8_cis_version: "8.0"` only (`mongodb8_cis_supported_versions`), and prelim stops any other requested or installed series with a clear message (D22).
- The repo URL and key name are still built from the version (see "Installation"), so a new series needs only a list entry once a matching CIS benchmark exists.

## Repo availability (S2) — Verified 2026-10-01

| RHEL | Latest `mongodb-enterprise-server` | Server builds published for 8.0 |
|------|------------------------------------|---------------------------------|
| 8  | 8.0.32-1.el8  | 29 |
| 9  | 8.0.32-1.el9  | 29 |
| 10 | 8.0.32-1.el10 | 4 (RHEL 10 support is recent) |

## What the RPM actually installs (S3) — Verified, identical on el8, el9 and el10

The shipped `/etc/mongod.conf` and `mongod.service` are byte-identical across the el8, el9 and el10 Enterprise RPMs, **and identical to the Community RPM** (`diff -r` showed no difference). The user, paths and modes below are the same in all of them. `mongod --version` of the Enterprise build reports `"modules": ["enterprise"]`.

| Item | Value from the RPM |
|------|--------------------|
| Service user / group | `mongod` / `mongod` (`useradd -M -r -g mongod -d /var/lib/mongo -s /bin/false ... mongod`) |
| Data dir (`storage.dbPath`) | `/var/lib/mongo` (owner `mongod:mongod`, mode 0755) |
| Log file | `/var/log/mongodb/mongod.log` (owner `mongod:mongod`, mode 0640) |
| Config file | `/etc/mongod.conf` (owner `root:root`, mode 0644) |
| systemd unit | `/usr/lib/systemd/system/mongod.service`, runs `mongod -f /etc/mongod.conf` |
| Extra options | `EnvironmentFile=-/etc/sysconfig/mongod` |
| TLS library | el9/el10: `libssl.so.3` (OpenSSL 3). **el8: `libssl.so.1.1` (OpenSSL 1.1.1).** Both are the system OpenSSL, so RHEL crypto-policies apply |

> **The docs and the package disagree.** The v8.0 S1 page says the default user is `mongodb` with
> `/var/lib/mongodb`. The RHEL RPM (S3) uses user **`mongod`** and **`/var/lib/mongo`**; the S1 values match the
> Debian/Ubuntu packages. The role follows the package (S3) and never hardcodes paths: it reads `storage.dbPath` and
> `systemLog.path` from the live `mongod.conf`.

Shipped `mongod.conf` (relevant keys):

```yaml
systemLog: {destination: file, logAppend: true, path: /var/log/mongodb/mongod.log}
storage: {dbPath: /var/lib/mongo}
net: {port: 27017, bindIp: 127.0.0.1}
#security:            # commented out → authorization is OFF on a fresh install
```

Shipped unit limits: `LimitFSIZE=infinity`, `LimitCPU=infinity`, `LimitAS=infinity`, `LimitNOFILE=64000`,
`LimitNPROC=64000`, `LimitMEMLOCK=infinity`, `TasksMax=infinity`.

## Fresh-install state vs the CIS rules (from S3, before any hardening)

| Rule | Fresh install | Why |
|------|---------------|-----|
| 2.1 authorization | **Non-compliant** | `security:` commented out |
| 2.2 localhost bypass | **Non-compliant** | `enableLocalhostAuthBypass` defaults to `true` (S4 parameters) |
| 4.3 TLS | **Non-compliant** | no `net.tls` block |
| 5.1 auditing | **Non-compliant** | `#auditLog:` commented out (under "Enterprise-Only Options" in the shipped file) |
| 5.3 `systemLog.quiet` | Compliant | not set; default `false` (S4 config options) |
| 5.4 `systemLog.logAppend` | Compliant | set to `true` in the shipped file (the mongod built-in default is `false`, S4) |
| 4.1 / 4.2 TLS protocols | **Non-compliant** | no `net.tls`. `mongod` refuses `disabledProtocols` without a TLS mode, so these need 4.3 first (D25) |
| 4.4 FIPS | **Non-compliant** | not set |
| 4.5 encryption at rest | Review | not set (Manual) |
| 6.1 port | **Non-compliant** | `27017` |
| 6.2 ulimits | Compliant | unit limits match the CIS values |

## SELinux — Verified 2026-10-01 in containers (Rocky 8.10, Rocky 9, AlmaLinux 10.2)

Sources:
- S1 (install page, SELinux section): build MongoDB's module on the server with `git make checkpolicy policycoreutils selinux-policy-devel`.
- S9: https://github.com/mongodb/mongodb-selinux, commit `18181652` (2025-10-03, "rhel9-policy"), `selinux/mongodb.te`, `selinux/mongodb.fc`, `Makefile`. GPL-2.0-or-later.
- Containers: base images + `dnf install selinux-policy-targeted selinux-policy-devel policycoreutils-python-utils` from their default repos; `semodule -n` (store only, nothing loaded into a kernel).

| Fact | RHEL 8 (8.10) | RHEL 9 | RHEL 10 (10.2) |
|------|---------------|--------|----------------|
| Base policy ships a `mongodb` module (priority 100) | **yes** | **yes** | no |
| Base labels `/usr/bin/mongod` → `mongod_exec_t`, `/var/lib/mongo.*` → `mongod_var_lib_t`, `/var/log/mongo.*` → `mongod_log_t` | yes | yes | no |
| So a default install runs `mongod` as | **`mongod_t` (confined)** | **`mongod_t` (confined)** | `unconfined_service_t` |
| `mongod_port_t` (base policy) | tcp 27017-27019, 28017-28019 | same | same |
| `selinux-policy-devel`, `policycoreutils-python-utils` repo | BaseOS | AppStream | AppStream |
| MongoDB's module (S9) builds | **no**: `mongodb.te:10: syntax error at token 'init_daemon_domain'` | yes | yes |
| MongoDB's module loads (`semodule --priority 200`) | — | yes, overrides the base module | yes |

- The MongoDB RPM ships no policy (`rpm -qpl` of the server RPM, S3), and the CIS MongoDB 8 benchmark has no SELinux recommendation (S5).
- S1 supports MongoDB's module only with default ports, `dbPath`, `systemLog.path` and `pidFilePath`. Anything else must be labelled, or a confined `mongod` cannot use it.
- *Revised 2026-10-01:* the earlier note "the RHEL base policy has no MongoDB domain" was checked on AlmaLinux 10 only. It is wrong for RHEL 8 and 9, which is why rule 6.1 now labels the port (D11).

### VM checks (per RHEL 8/9/10)

```bash
getenforce                                       # expect Enforcing
ps -eZ | grep mongod                             # 8/9: mongod_t; 10: unconfined_service_t until the extra is on
sudo semodule -lfull | grep -i mongo             # 8/9: "100 mongodb"; with the extra on 9/10: "200 mongodb"
sudo semanage port -l | grep mongod              # mongod_port_t 27017-27019, 28017-28019 (+ the 6.1 port)
sudo ausearch -m AVC -ts today | grep -i mongo   # expect none
```

## Ansible dependencies — Verified (sources S7, S8)

### Target Python on RHEL 8 (important)

S7, target Python per ansible-core version:

| ansible-core | Target Python |
|--------------|---------------|
| 2.16 | 2.7, 3.6 – 3.12 |
| 2.17 | 3.7 – 3.12 |
| 2.18 / 2.19 | 3.8 – 3.13 |
| 2.20 / 2.21 | 3.9 – 3.14 |

- RHEL 8's system Python (`/usr/libexec/platform-python`, `/usr/bin/python3`) is **3.6**, so ansible-core 2.17+ cannot manage it. RHEL 9 (3.9) and RHEL 10 (3.12) are fine.
- **Installing python3.12 on RHEL 8 is not enough:** the `dnf` bindings exist only for Python 3.6, so `ansible.builtin.dnf` still fails with 2.17+.
- **Chosen fix:** run RHEL 8 targets from a separate ansible-core **2.16** environment, pinned `>=2.16.1,<2.17` (resolves to the newest 2.16 patch). RHEL 8 then keeps its system Python, and no interpreter override is needed. Full steps and evidence: [../../docs/control-node-setup.md](../../docs/control-node-setup.md).
- **To verify on the RHEL 8 VM, from the 2.16 env:** `ansible <rhel8-host> -m ansible.builtin.dnf -a "name=tar state=present" --check` succeeds.

### pymongo (needed only by `community.mongodb` modules)

| EL | `python3-pymongo` | Source |
|----|-------------------|--------|
| 8  | **not available** in EPEL 8 | S8 |
| 9  | 3.10.1 (EPEL) | S8 |
| 10 | 4.8.0 (EPEL) | `dnf list` on the AlmaLinux 10 workstation |

pymongo is missing on EL8 and uneven elsewhere (3.x vs 4.x). By contrast, `mongosh` is installed by `mongodb-enterprise` itself on every OS (S2).
So database reads use `mongosh --quiet --eval` (read-only, `changed_when: false`). See design-decisions D13.

## RHEL 8 vs 9 vs 10 — summary

| Aspect | RHEL 8 | RHEL 9 | RHEL 10 | Role impact |
|--------|--------|--------|---------|-------------|
| Repo / install steps (S1, Enterprise) | same | same | same | only the `baseurl` number |
| Latest 8.0 Enterprise build (S2) | 8.0.32 | 8.0.32 | 8.0.32 | none |
| User, paths, config, unit (S3) | same | same | same | none (byte-identical) |
| OpenSSL (S3) | 1.1.1 | 3.x | 3.x | none for config rules; test TLS rules (4.1–4.3) on each |
| System Python (S7) | 3.6 (**unsupported by ansible-core 2.17+**) | 3.9 | 3.12 | RHEL 8 is run from the ansible-core 2.16.1+ env; prelim asserts it |
| pymongo (S8) | none | 3.10 | 4.8 | don't depend on it; use `mongosh` |
| SELinux base policy | `mongodb` module: `mongod_t` | `mongodb` module: `mongod_t` | none: `unconfined_service_t` | 6.1 labels the port (D11); optional extra (D17) |

Current evidence: **the role needs no per-OS task branches.** The repo URL is derived from the OS major version.
The only RHEL 8 difference (Python/ansible-core) is solved on the control node and checked by an assertion.
