# mongodb8_cis: coverage summary

Ansible role that hardens MongoDB 8.0 Enterprise on RHEL 8, 9 and 10 to the CIS MongoDB 8 Benchmark v2.0.0.
All 23 recommendations are implemented, one task file per recommendation.

## Coverage matrix

Mode: **Apply** = the role applies the CIS setting. **On value** = the role reports, and applies the fix when the
organization provides a value or list. **Report** = the role reports; a person decides.

| ID | Level | Type | Recommendation | Mode | Organization provides |
|----|-------|------|----------------|------|-----------------------|
| 1.1 | 1 | Manual | Software version and patches | Report | |
| 2.1 | 1 | Automated | Authentication is configured | Apply | Admin user and password |
| 2.2 | 1 | Automated | No localhost authentication bypass | Apply | |
| 2.3 | 2 | Automated | Authentication in the sharded cluster | Apply (cluster members only) | |
| 3.1 | 1 | Manual | Least privilege for database accounts | On value | Accounts to revoke |
| 3.2 | 1 | Manual | Role-based access control | On value | Accounts and roles |
| 3.3 | 1 | Manual | Non-privileged service account | Report | |
| 3.4 | 1 | Manual | Only necessary roles and privileges | On value | Roles to drop |
| 3.5 | 2 | Manual | Review superuser and admin roles | On value | Accounts to revoke |
| 4.1 | 2 | Automated | Legacy TLS protocols disabled | Apply | |
| 4.2 | 1 | Automated | Weak protocols disabled | Apply | |
| 4.3 | 1 | Automated | Encryption in transit (TLS) | Apply | Certificate files and their server paths |
| 4.4 | 2 | Automated | FIPS mode enabled | Apply | |
| 4.5 | 2 | Manual | Encryption at rest | Report | |
| 5.1 | 1 | Automated | System activity audited | Apply | |
| 5.2 | 2 | Manual | Audit filters configured | On value | Audit filter |
| 5.3 | 2 | Automated | Logging captures maximum detail | Apply | |
| 5.4 | 2 | Automated | Log entries appended | Apply | |
| 6.1 | 1 | Automated | Non-default port | Apply | Port |
| 6.2 | 2 | Manual | OS resource limits | Apply | |
| 6.3 | 2 | Manual | Server-side scripting disabled | Apply | |
| 7.1 | 1 | Manual | Key file permissions | Apply | |
| 7.2 | 1 | Manual | Database file permissions | Apply | |

| Mode | Count | Recommendations |
|------|-------|-----------------|
| Apply | 15 | 2.1, 2.2, 2.3, 4.1, 4.2, 4.3, 4.4, 5.1, 5.3, 5.4, 6.1, 6.2, 6.3, 7.1, 7.2 |
| On value | 5 | 3.1, 3.2, 3.4, 3.5, 5.2 |
| Report | 3 | 1.1, 3.3, 4.5 |

## Key points

- Every recommendation is enabled by default. An organization disables what it cannot accept; that list is its
  compliance exception list.
- A recommendation that needs a value that has not been provided reports "NOT APPLIED" and the run continues.
- Configuration changes are collected and written to `mongod.conf` once, with a backup, followed by one restart.
  A second run reports no changes.
- TLS uses two files: the server certificate with its private key, and the CA certificate.
- Encryption at rest (4.5) is enabled by hand: MongoDB cannot encrypt existing data, and key management is an
  organizational decision.
- Supported: MongoDB 8.0 only, the scope of the benchmark.

## Test status

| Test | Result |
|------|--------|
| Compliance test, RHEL 8/9/10, Level 2, 2026-10-06 (previous version of the role) | Passed, no failures |
| Lint and syntax check, current version | Clean |
| Compliance test, current version | Pending |
