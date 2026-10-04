# Test results — `mongodb8_cis` (rebuild, branch `mongodb8-cis-rebuild`)

Evidence from the test VMs. Screenshots live in [evidence-images/](evidence-images/), named
`NN-<os>-<scenario>-<result>.png`.

## SELinux baseline: is `mongod` confined without the role? (2026-10-03)

**Setup:** MongoDB Enterprise 8.0 installed and running, role **not** run, SELinux enforcing, no SELinux extra
(D17). Read-only commands on each VM:

```bash
ps -eZ | grep mongod                         # domain mongod runs in
sudo semanage fcontext -l | grep -i mongo    # MongoDB file rules in the policy
sudo semanage port -l | grep mongod          # MongoDB port label
ls -lZ /var/lib/mongo/                       # labels on the data files
```

| # | OS | `mongod` domain | MongoDB file rules | Port label `mongod_port_t` | Data file label | Evidence |
|---|----|-----------------|--------------------|----------------------------|-----------------|----------|
| 01 | RHEL 8 (`rhel8-pc`) | `mongod_t` (confined) | yes: binary, unit, `/var/lib/mongo`, `/var/log/mongo`, `/var/run/mongo` | 27017–27019, 28017–28019 | `mongod_var_lib_t` | [01-rhel8-selinux-mongod-confined.png](evidence-images/01-rhel8-selinux-mongod-confined.png) |
| 02 | RHEL 9 (`rhel9-pc`) | `mongod_t` (confined) | yes, same list as RHEL 8 | — (not captured) | — (not captured) | [02-rhel9-selinux-mongod-confined.png](evidence-images/02-rhel9-selinux-mongod-confined.png) |
| 03 | RHEL 10 (`rhel10-pc`) | `unconfined_service_t` (**not confined**) | **none** | 27017–27019, 28017–28019 | `var_lib_t` (generic) | [03-rhel10-selinux-mongod-unconfined.png](evidence-images/03-rhel10-selinux-mongod-unconfined.png) |

RHEL 8 port and data labels, and all RHEL 10 port/label output, were pasted as text in the session:

```text
[frqadmin@rhel8-pc ~]$ sudo semanage port -l | grep mongod
mongod_port_t                  tcp      27017-27019, 28017-28019
[frqadmin@rhel8-pc ~]$ ls -lZ /var/lib/mongo/
-rw-------. 1 mongod mongod system_u:object_r:mongod_var_lib_t:s0 20480 Oct  3 09:14 collection-0-9815987877185974117.wt
...

[frqadmin@rhel10-pc ~]$ sudo semanage port -l | grep mongod
mongod_port_t                  tcp      27017-27019, 28017-28019
[frqadmin@rhel10-pc ~]$ ls -lZ /var/lib/mongo/
-rw-------. 1 mongod mongod system_u:object_r:var_lib_t:s0 20480 Oct  3 21:22 collection-0-6961380469813015959.wt
...
```

**Conclusion:**
- RHEL 8 and 9: the OS base policy already confines `mongod` (`mongod_t`) and labels its files. 6.1 must label a
  new port (`seport`), or a confined `mongod` cannot bind it (D11).
- RHEL 10: the base policy keeps only the port definition; `mongod` runs unconfined and its files carry the generic
  `var_lib_t`. Real SELinux protection for MongoDB on RHEL 10 needs the optional extra
  (`mongodb8_cis_selinux_policy: true`, D17).
- The `aeolus-conductor`/`dbomatic` entries in the RHEL 8/9 list are leftovers of an unrelated project in Red Hat's
  `mongodb` module.
- Confirms D11 and D17 (previously recorded from the `main` branch tests).
