# Platform notes — MongoDB 8.0 Community on RHEL 8 / 9 / 10

What the role can rely on per OS, and where each fact comes from.
Status legend: **Verified** = checked against the source listed; **To verify** = must be confirmed on a test VM.

## Sources

| # | Source | Used for |
|---|--------|----------|
| S1 | MongoDB docs, *Install MongoDB Community Edition on Red Hat or CentOS* (v8.0): https://www.mongodb.com/docs/v8.0/tutorial/install-mongodb-on-red-hat/ | Supported OS, repo definition, install command, SELinux policy |
| S2 | Official repo metadata, `https://repo.mongodb.org/yum/redhat/<8|9|10>/mongodb-org/8.0/x86_64/repodata/` (read 2026-09-29) | Package availability and latest version per RHEL |
| S3 | `mongodb-org-server-8.0.32-1.el8`, `-1.el9` and `-1.el10` RPMs from S2, inspected with `rpm -qp --scripts`, `rpm -qplv`, `rpm2cpio` | Service user, paths, shipped `mongod.conf`, systemd unit |
| S4 | MongoDB docs: auditing https://www.mongodb.com/docs/v8.0/core/auditing/ , FIPS https://www.mongodb.com/docs/v8.0/tutorial/configure-fips/ , encryption at rest https://www.mongodb.com/docs/v8.0/core/security-encryption-at-rest/ , parameters https://www.mongodb.com/docs/v8.0/reference/parameters/ , config options https://www.mongodb.com/docs/v8.0/reference/configuration-options/ | Which features Community lacks; defaults |
| S5 | `cis-pdf/CIS_MongoDB_8_Benchmark_v2.0.0 PDF.pdf` | Rule text |
| S6 | `cis-spreadsheet/CIS_MongoDB_8_Benchmark_v2.0.0-Certification.xlsx` (sheets `Level 1- MongoDB`, `Level 2 - MongoDB`) | Level, Automated/Manual, Audit Procedure, Default Value per rule |
| S7 | ansible-core support matrix: https://docs.ansible.com/ansible/latest/reference_appendices/release_and_maintenance.html | Target Python per ansible-core version |
| S9 | MongoDB SELinux policy source: https://github.com/mongodb/mongodb-selinux | SELinux module content |
| S8 | EPEL repo metadata, `https://dl.fedoraproject.org/pub/epel/<8|9>/Everything/x86_64/repodata/` (read 2026-09-29) | pymongo availability |

## Installation (S1) — same steps on RHEL 8, 9, 10

S1 lists these as supported (64-bit): RHEL / CentOS Stream / Oracle / Rocky / AlmaLinux **8, 9 and 10**.
Oracle Linux only with the Red Hat Compatible Kernel (RHCK), not UEK (S1).

Repo file `/etc/yum.repos.d/mongodb-org-8.0.repo`. The only difference between OS versions is the number in `baseurl`:

```ini
[mongodb-org-8.0]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/<8|9|10>/mongodb-org/8.0/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://pgp.mongodb.com/server-8.0.asc
```

Install and start (S1):

```bash
sudo yum install -y mongodb-org          # latest 8.0.x
sudo systemctl enable --now mongod
```

`mongodb-org` is a metapackage. It pulls in `mongodb-org-database` (`-server`, `-mongos`, `mongodb-mongosh`) and `mongodb-org-tools` (`mongodb-database-tools`, `mongodb-org-database-tools-extra`) (S1).

**Role consequence:** one `ansible.builtin.yum_repository` task, with `baseurl` built from
`ansible_facts['distribution_major_version']`. No per-OS repo files.

## Repo availability (S2) — Verified 2026-09-29

| RHEL | Latest `mongodb-org-server` | Server builds published for 8.0 |
|------|------------------------------|---------------------------|
| 8  | 8.0.32-1.el8  | 29 |
| 9  | 8.0.32-1.el9  | 29 |
| 10 | 8.0.32-1.el10 | 4 (RHEL 10 support is recent) |

## What the RPM actually installs (S3) — Verified, identical on el8, el9 and el10

The shipped `/etc/mongod.conf` and `mongod.service` are byte-identical across the el8, el9 and el10 RPMs (`diff -r` showed no difference). The user, paths and modes below are the same in all three.

| Item | Value from the RPM |
|------|--------------------|
| Service user / group | `mongod` / `mongod` (`useradd -M -r -g mongod -d /var/lib/mongo -s /bin/false ... mongod`) |
| Data dir (`storage.dbPath`) | `/var/lib/mongo` (owner `mongod:mongod`, mode 0755) |
| Log file | `/var/log/mongodb/mongod.log` (owner `mongod:mongod`, mode 0640) |
| Config file | `/etc/mongod.conf` (owner `root:root`, mode 0644) |
| systemd unit | `/usr/lib/systemd/system/mongod.service`, runs `mongod -f /etc/mongod.conf` |
| Extra options | `EnvironmentFile=-/etc/sysconfig/mongod` |
| TLS library | el9/el10: `libssl.so.3` (OpenSSL 3). **el8: `libssl.so.1.1` (OpenSSL 1.1.1).** Both are the system OpenSSL, so RHEL crypto-policies apply |

> **The docs and the package disagree.** S1 says *"By default, MongoDB runs using the `mongodb` user account"* with
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
| 5.3 `systemLog.quiet` | Compliant | not set; default `false` (S4 config options) |
| 5.4 `systemLog.logAppend` | Compliant | set to `true` in the shipped file (the mongod built-in default is `false`, S4) |
| 6.1 port | **Non-compliant** | `27017` |
| 6.2 ulimits | Compliant | unit limits match the CIS values |

## SELinux

Sources:
- S1 (install page, SELinux section).
- S9: https://github.com/mongodb/mongodb-selinux (`selinux/mongodb.te`, `selinux/mongodb.fc`, `Makefile`; last push 2025-10-03, not archived).
- Local AlmaLinux 10.2 workstation: `selinux-policy-targeted-42.1.18-4.el10_2.3`, `getenforce` = Enforcing.

Scope: confining `mongod` is an **optional, non-CIS extra** in this role (`mongodb_cis_selinux_policy: false` by default). See [design-decisions.md](design-decisions.md) D17. These facts also matter for rule 6.1 (D11).

### Verified facts

| Fact | Evidence |
|------|----------|
| The MongoDB RPM ships **no** SELinux policy | `rpm -qpl mongodb-org-server-8.0.32-1.el9` lists no policy files (S3) |
| The RHEL/EL base policy has **no MongoDB domain** (no `mongod_t`, no file contexts) | `grep -i mongo /etc/selinux/targeted/contexts/files/file_contexts` → no match (6451 entries). `matchpathcon` gives `/usr/bin/mongod` = `bin_t`, `/var/lib/mongo` = `var_lib_t`, `/var/log/mongodb` = `var_log_t` |
| RHEL-shipped servers do get a domain in the base policy | same file: `/usr/bin/mariadbd` = `mysqld_exec_t`, `/usr/bin/redis-server` = `redis_exec_t`, `/usr/bin/httpd` = `httpd_exec_t` |
| MongoDB publishes its own policy module, built from source | S9 `Makefile`: `make` → `semodule --priority 200 --install mongodb.pp` → `fixfiles`/`restorecon`. Needs `git make checkpolicy policycoreutils selinux-policy-devel` (S1) |
| That module defines `mongod_t` and labels `/usr/bin/mongod`, `/var/lib/mongo`, `/var/log/mongodb`, `/run/mongodb`, and `mongod.service` | S9 `mongodb.fc` |
| The port type `mongod_port_t` (tcp 27017-27019, 28017-28019) comes from the base policy | S9 `mongodb.te` comment: *"port is defined by refpolicy"*. To verify: `semanage port -l \| grep mongod` |
| S1 supports the module only with default ports, `dbPath`, `systemLog.path` and `pidFilePath` | S1: *"you cannot use the MongoDB supplied SELinux policy"* with custom values |
| The CIS MongoDB 8 benchmark has no SELinux recommendation | `grep -i selinux` on the extracted benchmark → no match (S5) |

### What that means (expected, to verify on the VMs)

- **Without MongoDB's module:** systemd starts `/usr/bin/mongod`, which is labelled `bin_t` and has no domain transition. So `mongod` runs as **`unconfined_service_t`**.
  - SELinux stays enforcing and `mongod` works normally, with no AVC denials.
  - But `mongod` itself is **not confined**: SELinux does not limit what a compromised `mongod` can do.
  - This is why a default install "just works" with enforcing mode.
- **With MongoDB's module:** `mongod` runs as `mongod_t` and is confined to its own files and ports. Any non-default path or port must be labelled for it (e.g. `semanage port -a -t mongod_port_t -p tcp <port>` for rule 6.1), otherwise `mongod` fails to start.

### VM checks (per RHEL 8/9/10)

```bash
getenforce                                       # expect Enforcing
ps -eZ | grep mongod                             # expect unconfined_service_t (no module) or mongod_t (module)
ls -dZ /usr/bin/mongod /var/lib/mongo /var/log/mongodb
sudo semodule -l | grep -i mongo                 # empty = MongoDB module not installed
sudo semanage port -l | grep mongod              # expect mongod_port_t 27017-27019, 28017-28019
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

pymongo is missing on EL8 and uneven elsewhere (3.x vs 4.x). By contrast, `mongosh` is installed by `mongodb-org` itself on every OS (S1).
So database reads use `mongosh --quiet --eval` (read-only, `changed_when: false`). See design-decisions D13.

## RHEL 8 vs 9 vs 10 — summary

| Aspect | RHEL 8 | RHEL 9 | RHEL 10 | Role impact |
|--------|--------|--------|---------|-------------|
| Repo / install steps (S1) | same | same | same | only the `baseurl` number |
| Latest 8.0 build (S2) | 8.0.32 | 8.0.32 | 8.0.32 | none |
| User, paths, config, unit (S3) | same | same | same | none (byte-identical) |
| OpenSSL (S3) | 1.1.1 | 3.x | 3.x | none for config rules; test TLS rules (4.1–4.3) on each |
| System Python (S7) | 3.6 (**unsupported by ansible-core 2.17+**) | 3.9 | 3.12 | RHEL 8 is run from the ansible-core 2.16.1+ env; prelim asserts it |
| pymongo (S8) | none | 3.10 | 4.8 | don't depend on it; use `mongosh` |
| SELinux (no MongoDB module in base policy) | expected `unconfined_service_t` | verify | verify | rule 6.1 only (D11, D17) |

Current evidence: **the role needs no per-OS task branches.** The repo URL is derived from the OS major version.
The only RHEL 8 difference (Python/ansible-core) is solved on the control node and checked by an assertion.
