# Section 7 — File Permissions

**Goal:** only the MongoDB service account can read the **secrets** (key and certificate files) and the **data**
(database directory). Anyone else on the server gets nothing.

## Two kinds of files, one rule each

| What | Why it matters | Where (RHEL default) | Check it with |
|------|----------------|----------------------|---------------|
| 🔑 **Key / certificate files** (`keyFile`, TLS key, CA) | Whoever reads them can pretend to be the server or a cluster member | Not configured on a fresh standalone install | `sudo ls -l <path from mongod.conf>` |
| 🗄️ **Database directory** (`storage.dbPath`) | Holds all the data files; read access = read the whole database | `/var/lib/mongo`, RPM ships `0755 mongod:mongod` | `sudo stat -c '%a %U:%G' /var/lib/mongo` |

| Rule | Files | Goal | Role does | You can set |
|------|-------|------|-----------|-------------|
| **7.1** (L1, Manual) | 🔑 keys | Key files `0600`, owner mongod | PASS/FAIL per file found in `mongod.conf`, or NOT APPLICABLE if none; fixes existing files if you say so | `mongodb8_cis_fix_key_file_permissions: true` |
| **7.2** (L1, Manual) | 🗄️ data | dbPath `0770`, owner mongod | PASS/FAIL for the dbPath directory; fixes it if you say so | `mongodb8_cis_fix_db_path_permissions: true` |

Both are **Manual**, so by default the role only reports (D4). The fix variables are the site saying "yes, apply
CIS's remediation" (D20).

## Compare with the PDF

| CIS says | Role does | Why different (if it is) |
|----------|-----------|--------------------------|
| 7.1 Audit: grep `keyFile:`, `PEMKeyFile:`, `CAFile:` | Reads `security.keyFile`, `net.tls.certificateKeyFile`, `net.tls.CAFile` from the parsed config | `PEMKeyFile` is the old `net.ssl` name; MongoDB renamed it to `net.tls.certificateKeyFile` in 4.2. Old `net.ssl` blocks are flagged by prelim (batch 9) |
| 7.1 Remediation: `chmod 600`, `chown mongodb:mongodb` | Sets `0600`, owner and group = the user mongod runs as (`mongod` on RHEL) | `mongodb` is the Ubuntu user; the RHEL RPM uses `mongod` (D8) |
| 7.1 Remediation: "remove other permissions" | PASS = `0600` **or `0400`** (read-only is stricter, MongoDB's keyfile docs use `400`), owner and group = service user | — |
| 7.1 covers the CA file too | CA file included, as CIS lists it | A CA certificate isn't secret, but CIS groups it with the keys, so the role follows CIS |
| 7.2 Audit: grep `dbPath`, `stat -c '%a' /var/lib/mongodb` | `stat` on `storage.dbPath` (fallback `/var/lib/mongo`) | `/var/lib/mongodb` is the Ubuntu path; RHEL uses `/var/lib/mongo` (D8) |
| 7.2 Remediation: `chmod 770`, `chown mongodb:mongodb` on the directory | Sets `0770`, owner and group = service user, on the directory only | Same as CIS: no `-R`; mongod creates its files inside with its own safe modes |
| Both: "Default Value: Not configured" | 7.1 NOT APPLICABLE, 7.2 FAIL on a fresh install | RPM ships dbPath `0755` |

## Good to know

- **Fresh install:** 7.1 → `NOT APPLICABLE` (standalone, no TLS or keyFile yet); 7.2 → `FAIL ... 0755` until you set
  `mongodb8_cis_fix_db_path_permissions: true`. After the fix, rerun → `PASS`, `changed=0`.
- **7.1 becomes real after 4.3** (TLS on): the certificate and CA paths then appear in `mongod.conf`, and 7.1 checks them.
- 7.1 never creates a missing file: a missing path is reported as `FAIL ... MISSING` and skipped by the fix.
- `0770` gives the `mongod` **group** full access too; keep that group for the service account only.
- No restart needed: permission changes take effect immediately.

Source: CIS MongoDB 8 Benchmark v2.0.0, recommendations 7.1–7.2 (PDF section 7, from page 91).
