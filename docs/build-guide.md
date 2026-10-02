# Build guide — `mongodb8_cis`, batch by batch

Every file of the role, in the order to type it, with a test after each batch. The code here is copied from
files that passed `yamllint` + `ansible-lint` (`production`). It has been run on RHEL VMs (sections 1, 4–7, install,
SELinux: tests T1–T10) or against a real MongoDB 8.0.32 Enterprise (new sections 2 and 3 with pymongo, both ansible-core
2.16.19 and 2.20.7). Ask questions in chat; follow the steps here.

**Already written by Claude (don't retype):** `meta/main.yml`, `defaults/main.yml`, `vars/main.yml`, `requirements.yml`,
`.yamllint`, `.ansible-lint`, `files/selinux/`. **You type:** everything under `tasks/` and `handlers/`.

**After every batch**
```bash
cd ~/ansible-cis/mongodb_cis && yamllint . && ansible-lint --offline     # inside the role folder
cd ~/mongodb8-cis-test && ansible-playbook playbooks/site.yml --limit <vm> -K
```
Add `--check` first when the batch changes things. Revert a VM to `base` before a test that needs a fresh install.

| Batch | Files | CIS rules |
|-------|-------|-----------|
| 1 ✅ | `tasks/main.yml`, `prelim.yml` checks | — |
| 2 | rest of `prelim.yml` | — |
| 3 | `install.yml`, `handlers/main.yml`, prelim install hook | — |
| 4 | `section_5/` | 5.1–5.4 |
| 5 | `section_1/`, `section_7/` | 1.1, 7.1, 7.2 |
| 6 | `section_6/` | 6.1–6.3 |
| 7 | `pymongo.yml`, sections 2+3 block, prelim login check, `section_3/` | 3.1–3.5 |
| 8 | `section_2/` | 2.1–2.3 |
| 9 | `section_4/`, prelim `net.ssl` check | 4.1–4.5 |
| 10 | `selinux.yml` (optional extra) | — |

---

## Batch 1 ✅ `tasks/main.yml` + prelim checks (done)

Fixes to make in what you typed: `mongodb_cis_supported_arch` → `mongodb8_cis_supported_arch`, `mongodb_cis_version` →
`mongodb8_cis_version`, task name `ansible-code` → `ansible-core`, and delete the empty last line of `main.yml`.

---

## Batch 2 — finish `tasks/prelim.yml`

Append to `prelim.yml`:
```yaml
- name: "PRELIM | AUDIT | Gather installed packages"
  ansible.builtin.package_facts:
    manager: rpm

- name: "PRELIM | AUDIT | Detect MongoDB server package"
  ansible.builtin.set_fact:
    discovered_mongodb_installed: "{{ mongodb8_cis_server_package in ansible_facts['packages'] }}"
    discovered_mongodb_version: "{{ (ansible_facts['packages'][mongodb8_cis_server_package] | default([{'version': 'none'}]))[0]['version'] }}"

- name: "PRELIM | AUDIT | Stop when MongoDB is not installed"
  when: not discovered_mongodb_installed
  block:
    - name: "PRELIM | AUDIT | Stop when MongoDB is not installed | Report"
      ansible.builtin.debug:
        msg: >-
          {{ mongodb8_cis_server_package }} is not installed on {{ inventory_hostname }}: nothing to harden.
          mongodb8_cis_install={{ mongodb8_cis_install }}.

    - name: "PRELIM | AUDIT | Stop when MongoDB is not installed | End this host"
      ansible.builtin.meta: end_host

- name: "PRELIM | AUDIT | Check the installed MongoDB version is supported"
  ansible.builtin.assert:
    that:
      - discovered_mongodb_version.split('.')[:2] | join('.') in mongodb8_cis_supported_versions
    fail_msg: >-
      MongoDB {{ discovered_mongodb_version }} is installed. This role implements the CIS MongoDB 8 Benchmark:
      supported {{ mongodb8_cis_supported_versions | join(', ') }}.
    quiet: true

- name: "PRELIM | AUDIT | Read mongod.conf"
  ansible.builtin.slurp:
    src: "{{ mongodb8_cis_conf_path }}"
  register: discovered_mongod_conf_raw

- name: "PRELIM | AUDIT | Parse mongod.conf"
  ansible.builtin.set_fact:
    discovered_mongod_conf: "{{ discovered_mongod_conf_raw['content'] | b64decode | from_yaml | default({}, true) }}"

- name: "PRELIM | AUDIT | Keep the config mongod is running with"
  ansible.builtin.set_fact:
    discovered_mongod_running_conf: "{{ discovered_mongod_conf }}"

- name: "PRELIM | AUDIT | Read mongod service properties"
  ansible.builtin.systemd_service:
    name: "{{ mongodb8_cis_service }}"
  register: discovered_mongod_service

- name: "PRELIM | AUDIT | Show what was found"
  ansible.builtin.debug:
    msg:
      - "MongoDB server: {{ discovered_mongodb_version }}"
      - "Config file: {{ mongodb8_cis_conf_path }}"
      - "dbPath: {{ discovered_mongod_conf['storage']['dbPath'] | default(mongodb8_cis_default_db_path) }}"
      - "Log file: {{ discovered_mongod_conf['systemLog']['path'] | default(mongodb8_cis_default_log_path) }}"
      - "Service user: {{ mongodb8_cis_service_user }}"
```

| Task | Does | Why |
|------|------|-----|
| Gather installed packages | `package_facts` (rpm) | Detect, don't assume |
| Detect MongoDB server package | `discovered_mongodb_installed` / `_version` from `mongodb-enterprise-server` | The package that holds `mongod`; `default(...)` avoids an error when it's missing |
| Stop when not installed | message + `meta: end_host` | Clean skip, not a failure, for this host only |
| Check installed version | `8.0.32` → `8.0` must be supported | The CIS MongoDB 8 benchmark only fits 8.x |
| Read / parse `mongod.conf` | `slurp` → `b64decode` → `from_yaml` → `discovered_mongod_conf`, plus a "running" copy | The shared AUDIT all config rules compare against; the running copy is used to connect to the DB later |
| Read service properties | `systemd_service` with no `state` = read only | Service user, PID and limits for 3.3, 6.2, 7.x |
| Show what was found | version, config, dbPath, log, user | Summary on every run |

**Test:** fresh VM (`rpm -q mongodb-enterprise-server` → not installed): *"…not installed … nothing to harden"*, `failed=0`.
A VM with MongoDB 8.0: the summary, `changed=0`.

---

## Batch 3 — install + handlers

`tasks/install.yml`
```yaml
---
- name: "INSTALL | PATCH | Import the MongoDB package signing key"
  ansible.builtin.rpm_key:
    key: "{{ mongodb8_cis_repo_gpgkey }}"
    state: present

- name: "INSTALL | PATCH | Add the official MongoDB Enterprise repository"
  ansible.builtin.yum_repository:
    name: "{{ mongodb8_cis_repo_name }}"
    description: MongoDB Enterprise Repository
    baseurl: "{{ mongodb8_cis_repo_baseurl }}"
    gpgkey: "{{ mongodb8_cis_repo_gpgkey }}"
    gpgcheck: true
    enabled: true

- name: "INSTALL | PATCH | Install MongoDB packages"
  ansible.builtin.dnf:
    name: "{{ mongodb8_cis_package }}"
    state: present

- name: "INSTALL | PATCH | Start and enable mongod"
  ansible.builtin.systemd_service:
    name: "{{ mongodb8_cis_service }}"
    state: started
    enabled: true
```

`handlers/main.yml`
```yaml
---
- name: Restart mongod service
  listen: Restart mongod
  ansible.builtin.systemd_service:
    name: "{{ mongodb8_cis_service }}"
    state: restarted

- name: Wait for mongod to accept connections
  listen: Restart mongod
  ansible.builtin.wait_for:
    host: "{{ mongodb8_cis_listen_host }}"
    port: "{{ mongodb8_cis_listen_port | int }}"
    timeout: 60
```

Insert into `prelim.yml`, **after** "Check the requested MongoDB version is supported" and **before** "Gather installed packages":
```yaml
- name: "PRELIM | PATCH | Install MongoDB when requested"
  when: mongodb8_cis_install
  ansible.builtin.import_tasks:
    file: install.yml
```

| Piece | Does | Why |
|-------|------|-----|
| `install.yml` | key → Enterprise repo → `dnf mongodb-enterprise` (`present`) → start + enable | MongoDB's official steps; all declarative, so rerun = `changed=0`. `--check` on a host without MongoDB fails at `dnf` (known limitation) |
| Install hook in prelim | runs install only with `mongodb8_cis_install: true` | Before detection, so detection sees the result |
| Handlers | restart mongod, then wait for its port (60 s) | One restart at the end; a broken config fails right there |

**Test:** fresh VM, `mongodb8_cis_install: true` → `changed=4` (key, repo, packages, service) + summary; again → `changed=0`.

---

## Batch 4 — Section 5: Audit Logging

Add to `tasks/main.yml` (after prelim):
```yaml

- name: Run section 5 - Audit Logging
  when: mongodb8_cis_section5
  ansible.builtin.import_tasks:
    file: section_5/main.yml
```

`tasks/section_5/main.yml`
```yaml
---
- name: "SECTION | 5.1 | Ensure that system activity is audited"
  ansible.builtin.import_tasks:
    file: cis_5.1.yml

- name: "SECTION | 5.2 | Ensure that audit filters are configured properly"
  ansible.builtin.import_tasks:
    file: cis_5.2.yml

- name: "SECTION | 5.3 | Ensure that logging captures as much information as possible"
  ansible.builtin.import_tasks:
    file: cis_5.3.yml

- name: "SECTION | 5.4 | Ensure that new entries are appended to the end of the log file"
  ansible.builtin.import_tasks:
    file: cis_5.4.yml
```

`tasks/section_5/cis_5.1.yml`
```yaml
---
- name: "5.1 | PATCH | Ensure that system activity is audited"
  when:
    - mongodb8_cis_rule_5_1
    - mongodb8_cis_level_1
    - discovered_mongod_conf['auditLog']['destination'] | default('') | length == 0
  tags:
    - level1
    - automated
    - patch
    - rule_5.1
    - logging
  vars:
    mongodb8_cis_5_1_path: >-
      {{ mongodb8_cis_audit_path or ((discovered_mongod_conf['systemLog']['path'] | default(mongodb8_cis_default_log_path) | dirname)
      ~ '/auditLog.' ~ (mongodb8_cis_audit_format | lower)) }}
    mongodb8_cis_5_1_settings:
      auditLog: >-
        {{ {'destination': mongodb8_cis_audit_destination}
        | combine({'format': mongodb8_cis_audit_format, 'path': mongodb8_cis_5_1_path} if mongodb8_cis_audit_destination == 'file' else {}) }}
  block:
    - name: "5.1 | PATCH | Ensure that system activity is audited | Check the site values"
      ansible.builtin.assert:
        that:
          - mongodb8_cis_audit_destination in ['syslog', 'console', 'file']
          - mongodb8_cis_audit_destination != 'file' or mongodb8_cis_audit_format in ['JSON', 'BSON']
        fail_msg: >-
          5.1: mongodb8_cis_audit_destination must be syslog, console or file (found '{{ mongodb8_cis_audit_destination }}');
          with file, mongodb8_cis_audit_format must be JSON or BSON (found '{{ mongodb8_cis_audit_format }}').
        quiet: true

    - name: "5.1 | PATCH | Ensure that system activity is audited | Set auditLog"
      ansible.builtin.copy:
        dest: "{{ mongodb8_cis_conf_path }}"
        content: "{{ discovered_mongod_conf | combine(mongodb8_cis_5_1_settings, recursive=true) | to_nice_yaml(indent=2) }}"
        owner: root
        group: root
        mode: "0644"
        backup: true
      notify: Restart mongod

    - name: "5.1 | PATCH | Ensure that system activity is audited | Update the parsed config"
      ansible.builtin.set_fact:
        discovered_mongod_conf: "{{ discovered_mongod_conf | combine(mongodb8_cis_5_1_settings, recursive=true) }}"
```

`tasks/section_5/cis_5.2.yml`
```yaml
---
- name: "5.2 | AUDIT | Ensure that audit filters are configured properly"
  when:
    - mongodb8_cis_rule_5_2
    - mongodb8_cis_level_2
  tags:
    - level2
    - manual
    - audit
    - rule_5.2
    - logging
  block:
    - name: "5.2 | AUDIT | Ensure that audit filters are configured properly | Report"
      ansible.builtin.debug:
        msg: >-
          5.2 REVIEW: {{ ('auditLog.filter = ' ~ (discovered_mongod_conf['auditLog']['filter']
          | default('not set (all auditable events are recorded)')) ~ '. Check it against the audit requirements of the organisation.')
          if discovered_mongod_conf['auditLog']['destination'] | default('') | length > 0
          else 'auditing is off (no auditLog.destination), so there is no filter to review. See rule 5.1.' }}

    - name: "5.2 | PATCH | Ensure that audit filters are configured properly | Set auditLog.filter (site value)"
      when:
        - mongodb8_cis_audit_filter | length > 0
        - discovered_mongod_conf['auditLog']['destination'] | default('') | length > 0
        - discovered_mongod_conf['auditLog']['filter'] | default('') != mongodb8_cis_audit_filter
      tags:
        - patch
      block:
        - name: "5.2 | PATCH | Ensure that audit filters are configured properly | Write auditLog.filter"
          ansible.builtin.copy:
            dest: "{{ mongodb8_cis_conf_path }}"
            content: "{{ discovered_mongod_conf | combine({'auditLog': {'filter': mongodb8_cis_audit_filter}}, recursive=true) | to_nice_yaml(indent=2) }}"
            owner: root
            group: root
            mode: "0644"
            backup: true
          notify: Restart mongod

        - name: "5.2 | PATCH | Ensure that audit filters are configured properly | Update the parsed config"
          ansible.builtin.set_fact:
            discovered_mongod_conf: "{{ discovered_mongod_conf | combine({'auditLog': {'filter': mongodb8_cis_audit_filter}}, recursive=true) }}"
```

`tasks/section_5/cis_5.3.yml`
```yaml
---
- name: "5.3 | PATCH | Ensure that logging captures as much information as possible"
  when:
    - mongodb8_cis_rule_5_3
    - mongodb8_cis_level_2
    - discovered_mongod_conf['systemLog']['quiet'] | default(false) is true
  tags:
    - level2
    - automated
    - patch
    - rule_5.3
    - logging
  vars:
    mongodb8_cis_5_3_settings:
      systemLog:
        quiet: false
  block:
    - name: "5.3 | PATCH | Ensure that logging captures as much information as possible | Set systemLog.quiet: false"
      ansible.builtin.copy:
        dest: "{{ mongodb8_cis_conf_path }}"
        content: "{{ discovered_mongod_conf | combine(mongodb8_cis_5_3_settings, recursive=true) | to_nice_yaml(indent=2) }}"
        owner: root
        group: root
        mode: "0644"
        backup: true
      notify: Restart mongod

    - name: "5.3 | PATCH | Ensure that logging captures as much information as possible | Update the parsed config"
      ansible.builtin.set_fact:
        discovered_mongod_conf: "{{ discovered_mongod_conf | combine(mongodb8_cis_5_3_settings, recursive=true) }}"
```

`tasks/section_5/cis_5.4.yml`
```yaml
---
- name: "5.4 | PATCH | Ensure that new entries are appended to the end of the log file"
  when:
    - mongodb8_cis_rule_5_4
    - mongodb8_cis_level_2
    - discovered_mongod_conf['systemLog']['logAppend'] | default(false) is not true
  tags:
    - level2
    - automated
    - patch
    - rule_5.4
    - logging
  vars:
    mongodb8_cis_5_4_settings:
      systemLog:
        logAppend: true
  block:
    - name: "5.4 | PATCH | Ensure that new entries are appended to the end of the log file | Set systemLog.logAppend: true"
      ansible.builtin.copy:
        dest: "{{ mongodb8_cis_conf_path }}"
        content: "{{ discovered_mongod_conf | combine(mongodb8_cis_5_4_settings, recursive=true) | to_nice_yaml(indent=2) }}"
        owner: root
        group: root
        mode: "0644"
        backup: true
      notify: Restart mongod

    - name: "5.4 | PATCH | Ensure that new entries are appended to the end of the log file | Update the parsed config"
      ansible.builtin.set_fact:
        discovered_mongod_conf: "{{ discovered_mongod_conf | combine(mongodb8_cis_5_4_settings, recursive=true) }}"
```

| Rule | Type | Does |
|------|------|------|
| 5.1 L1 | PATCH | Adds `auditLog` (`syslog` by default) if missing; never replaces one |
| 5.2 L2 | REPORT + optional site value | Shows `auditLog.filter`; if `mongodb8_cis_audit_filter` is set (and auditing is on), writes it. Tested on mongod 8.0.32: filter accepted, rerun `changed=0` |
| 5.3 L2 | PATCH | `systemLog.quiet` true → false |
| 5.4 L2 | PATCH | `systemLog.logAppend` → true |

All PATCH rules follow AUDIT → PATCH: the `when:` compares the parsed config; `copy` writes current config + the CIS value; `notify` restart; `set_fact` remembers the change.

**Test:** run → 5.1 adds `auditLog`, one restart; run again → `changed=0`. With Level 2 on, set `logAppend: false` on the VM → 5.4 fixes it; rerun → `changed=0`.

---

## Batch 5 — Section 1 and Section 7 (reports)

Add to `tasks/main.yml` — section 1 **before** section 5, section 7 at the end:
```yaml

- name: Run section 1 - Installation and Patching
  when: mongodb8_cis_section1
  ansible.builtin.import_tasks:
    file: section_1/main.yml
```
```yaml

- name: Run section 7 - File Permissions
  when: mongodb8_cis_section7
  ansible.builtin.import_tasks:
    file: section_7/main.yml
```

`tasks/section_1/main.yml`
```yaml
---
- name: "SECTION | 1.1 | Ensure the appropriate MongoDB software version/patches are installed"
  ansible.builtin.import_tasks:
    file: cis_1.1.yml
```

`tasks/section_1/cis_1.1.yml`
```yaml
---
- name: "1.1 | AUDIT | Ensure the appropriate MongoDB software version/patches are installed"
  when:
    - mongodb8_cis_rule_1_1
    - mongodb8_cis_level_1
  tags:
    - level1
    - manual
    - audit
    - rule_1.1
    - patching
  ansible.builtin.debug:
    msg: >-
      1.1 REVIEW: installed MongoDB server {{ discovered_mongodb_version }}. Compare with the latest
      {{ mongodb8_cis_version }}.x release and the security alerts at https://www.mongodb.com/alerts.
      The role never upgrades automatically.
```

`tasks/section_7/main.yml`
```yaml
---
- name: "SECTION | 7.1 | Ensure appropriate key file permissions are set"
  ansible.builtin.import_tasks:
    file: cis_7.1.yml

- name: "SECTION | 7.2 | Ensure appropriate database file permissions are set."
  ansible.builtin.import_tasks:
    file: cis_7.2.yml
```

`tasks/section_7/cis_7.1.yml`
```yaml
---
- name: "7.1 | AUDIT | Ensure appropriate key file permissions are set"
  when:
    - mongodb8_cis_rule_7_1
    - mongodb8_cis_level_1
  tags:
    - level1
    - manual
    - audit
    - rule_7.1
    - permissions
  vars:
    mongodb8_cis_7_1_files: >-
      {{ [discovered_mongod_conf['security']['keyFile'] | default(''),
          discovered_mongod_conf['net']['tls']['certificateKeyFile'] | default(''),
          discovered_mongod_conf['net']['tls']['CAFile'] | default('')] | select | list }}
  block:
    - name: "7.1 | AUDIT | Ensure appropriate key file permissions are set | Get key and certificate files"
      ansible.builtin.stat:
        path: "{{ item }}"
        get_checksum: false
      loop: "{{ mongodb8_cis_7_1_files }}"
      register: discovered_7_1_stat

    - name: "7.1 | AUDIT | Ensure appropriate key file permissions are set | Report when none configured"
      when: mongodb8_cis_7_1_files | length == 0
      ansible.builtin.debug:
        msg: "7.1 NOT APPLICABLE: no keyFile, certificateKeyFile or CAFile in {{ mongodb8_cis_conf_path }}."

    - name: "7.1 | AUDIT | Ensure appropriate key file permissions are set | Report each file"
      loop: "{{ discovered_7_1_stat['results'] }}"
      loop_control:
        label: "{{ item['item'] }}"
      ansible.builtin.debug:
        msg: >-
          7.1 {{ 'PASS' if (item['stat']['exists'] and item['stat']['mode'] in ['0600', '0400']
          and item['stat']['pw_name'] == mongodb8_cis_service_user) else 'FAIL' }}:
          {{ item['item'] }} is {{ item['stat']['mode'] | default('MISSING') }}
          {{ item['stat']['pw_name'] | default('-') }}:{{ item['stat']['gr_name'] | default('-') }}
          (expected 0600 {{ mongodb8_cis_service_user }}:{{ mongodb8_cis_service_user }}).

    # CIS remediation: chmod 600, owned by the mongo user. Only files that exist; the role never creates them.
    - name: "7.1 | PATCH | Ensure appropriate key file permissions are set | Set 0600 and owner (site decision)"
      when:
        - mongodb8_cis_fix_key_file_permissions
        - item['stat']['exists']
      ansible.builtin.file:
        path: "{{ item['item'] }}"
        owner: "{{ mongodb8_cis_service_user }}"
        group: "{{ mongodb8_cis_service_user }}"
        mode: "0600"
      loop: "{{ discovered_7_1_stat['results'] }}"
      loop_control:
        label: "{{ item['item'] }}"
      tags:
        - patch
```

`tasks/section_7/cis_7.2.yml`
```yaml
---
- name: "7.2 | AUDIT | Ensure appropriate database file permissions are set."
  when:
    - mongodb8_cis_rule_7_2
    - mongodb8_cis_level_1
  tags:
    - level1
    - manual
    - audit
    - rule_7.2
    - permissions
  vars:
    mongodb8_cis_7_2_db_path: "{{ discovered_mongod_conf['storage']['dbPath'] | default(mongodb8_cis_default_db_path) }}"
  block:
    - name: "7.2 | AUDIT | Ensure appropriate database file permissions are set. | Get dbPath permissions"
      ansible.builtin.stat:
        path: "{{ mongodb8_cis_7_2_db_path }}"
      register: discovered_db_path

    - name: "7.2 | AUDIT | Ensure appropriate database file permissions are set. | Report"
      ansible.builtin.debug:
        msg: >-
          7.2 {{ 'PASS' if (discovered_db_path.stat.mode == '0770'
          and discovered_db_path.stat.pw_name == mongodb8_cis_service_user
          and discovered_db_path.stat.gr_name == mongodb8_cis_service_user) else 'FAIL' }}:
          {{ mongodb8_cis_7_2_db_path }} is {{ discovered_db_path.stat.mode }}
          {{ discovered_db_path.stat.pw_name }}:{{ discovered_db_path.stat.gr_name }}
          (expected 0770 {{ mongodb8_cis_service_user }}:{{ mongodb8_cis_service_user }}).

    - name: "7.2 | PATCH | Ensure appropriate database file permissions are set. | Set 0770 and owner (site decision)"
      when: mongodb8_cis_fix_db_path_permissions
      ansible.builtin.file:
        path: "{{ mongodb8_cis_7_2_db_path }}"
        state: directory
        owner: "{{ mongodb8_cis_service_user }}"
        group: "{{ mongodb8_cis_service_user }}"
        mode: "0770"
      tags:
        - patch
```

| Rule | Type | Does |
|------|------|------|
| 1.1 L1 | REPORT | Installed version + where to check patches; never upgrades |
| 7.1 L1 | REPORT + optional fix | PASS/FAIL per keyFile, TLS key, CA file (0600, owner mongod), or "not applicable"; fixes existing files only with `mongodb8_cis_fix_key_file_permissions: true` |
| 7.2 L1 | DECISION | dbPath vs `0770` owner mongod (RPM ships `0755` → FAIL); fixes only with `mongodb8_cis_fix_db_path_permissions: true` |

**Test:** 1.1 REVIEW, 7.1 NOT APPLICABLE, **7.2 FAIL 0755**. With the decision on → fixed; rerun → PASS, `changed=0`.

---

## Batch 6 — Section 6: OS Hardening

Add to `tasks/main.yml` (before section 7):
```yaml

- name: Run section 6 - Operating System Hardening
  when: mongodb8_cis_section6
  ansible.builtin.import_tasks:
    file: section_6/main.yml
```

`tasks/section_6/main.yml`
```yaml
---
- name: "SECTION | 6.1 | Ensure that MongoDB uses a non-default port"
  ansible.builtin.import_tasks:
    file: cis_6.1.yml

- name: "SECTION | 6.2 | Ensure that operating system resource limits are set for MongoDB"
  ansible.builtin.import_tasks:
    file: cis_6.2.yml

- name: "SECTION | 6.3 | Ensure that server-side scripting is disabled if not needed"
  ansible.builtin.import_tasks:
    file: cis_6.3.yml
```

`tasks/section_6/cis_6.1.yml`
```yaml
---
- name: "6.1 | PATCH | Ensure that MongoDB uses a non-default port"
  when:
    - mongodb8_cis_rule_6_1
    - mongodb8_cis_level_1
  tags:
    - level1
    - automated
    - patch
    - rule_6.1
    - network
  block:
    - name: "6.1 | PATCH | Ensure that MongoDB uses a non-default port | Check the site port"
      ansible.builtin.assert:
        that:
          - mongodb8_cis_port | int >= 1024
          - mongodb8_cis_port | int <= 65535
          - mongodb8_cis_port | int != 27017
        fail_msg: "6.1: set mongodb8_cis_port to 1024-65535, not 27017 (found '{{ mongodb8_cis_port }}')."
        quiet: true

    # RHEL 8/9 base policy confines mongod (mongod_t): it can only bind ports labelled mongod_port_t (D11).
    - name: "6.1 | PATCH | Ensure that MongoDB uses a non-default port | Label the port for SELinux"
      when:
        - ansible_facts['selinux']['status'] | default('disabled') == 'enabled'
        - mongodb8_cis_port | int not in mongodb8_cis_selinux_default_ports
      block:
        - name: "6.1 | PATCH | Ensure that MongoDB uses a non-default port | Install the SELinux management tools"
          ansible.builtin.dnf:
            name: policycoreutils-python-utils
            state: present

        - name: "6.1 | PATCH | Ensure that MongoDB uses a non-default port | Set mongod_port_t"
          community.general.seport:
            ports: "{{ mongodb8_cis_port | int }}"
            proto: tcp
            setype: mongod_port_t
            state: present

    - name: "6.1 | PATCH | Ensure that MongoDB uses a non-default port | Set net.port"
      when: discovered_mongod_conf['net']['port'] | default(27017) | int != mongodb8_cis_port | int
      ansible.builtin.copy:
        dest: "{{ mongodb8_cis_conf_path }}"
        content: "{{ discovered_mongod_conf | combine({'net': {'port': mongodb8_cis_port | int}}, recursive=true) | to_nice_yaml(indent=2) }}"
        owner: root
        group: root
        mode: "0644"
        backup: true
      notify: Restart mongod

    - name: "6.1 | PATCH | Ensure that MongoDB uses a non-default port | Update the parsed config"
      ansible.builtin.set_fact:
        discovered_mongod_conf: "{{ discovered_mongod_conf | combine({'net': {'port': mongodb8_cis_port | int}}, recursive=true) }}"
```

`tasks/section_6/cis_6.2.yml`
```yaml
---
- name: "6.2 | AUDIT | Ensure that operating system resource limits are set for MongoDB"
  when:
    - mongodb8_cis_rule_6_2
    - mongodb8_cis_level_2
  tags:
    - level2
    - manual
    - audit
    - rule_6.2
    - limits
  vars:
    # CIS ulimit letters -> systemd: f=FSIZE, t=CPU, v=AS, n=NOFILE, m=RSS, u=NPROC
    mongodb8_cis_6_2_limits:
      LimitFSIZE: infinity
      LimitCPU: infinity
      LimitAS: infinity
      LimitNOFILE: "64000"
      LimitRSS: infinity
      LimitNPROC: "64000"
  ansible.builtin.debug:
    msg: >-
      6.2 {{ 'PASS' if discovered_mongod_service['status'][item.key] | default('') == item.value else 'REVIEW' }}:
      {{ item.key }} = {{ discovered_mongod_service['status'][item.key] | default('unknown') }} (CIS: {{ item.value }}).
  loop: "{{ mongodb8_cis_6_2_limits | dict2items }}"
  loop_control:
    label: "{{ item.key }}"
```

`tasks/section_6/cis_6.3.yml`
```yaml
---
- name: "6.3 | AUDIT | Ensure that server-side scripting is disabled if not needed"
  when:
    - mongodb8_cis_rule_6_3
    - mongodb8_cis_level_2
  tags:
    - level2
    - manual
    - audit
    - rule_6.3
    - scripting
  vars:
    mongodb8_cis_6_3_settings:
      security:
        javascriptEnabled: false
  block:
    - name: "6.3 | AUDIT | Ensure that server-side scripting is disabled if not needed | Report"
      ansible.builtin.debug:
        msg: >-
          6.3 REVIEW: security.javascriptEnabled = {{ discovered_mongod_conf['security']['javascriptEnabled'] | default('not set (default true)') }}.
          Site decision mongodb8_cis_javascript_needed = {{ mongodb8_cis_javascript_needed }}.

    - name: "6.3 | PATCH | Ensure that server-side scripting is disabled if not needed | Set security.javascriptEnabled: false (site decision)"
      when:
        - not mongodb8_cis_javascript_needed
        - discovered_mongod_conf['security']['javascriptEnabled'] | default(true) is not false
      tags:
        - patch
      block:
        - name: "6.3 | PATCH | Ensure that server-side scripting is disabled if not needed | Write mongod.conf"
          ansible.builtin.copy:
            dest: "{{ mongodb8_cis_conf_path }}"
            content: "{{ discovered_mongod_conf | combine(mongodb8_cis_6_3_settings, recursive=true) | to_nice_yaml(indent=2) }}"
            owner: root
            group: root
            mode: "0644"
            backup: true
          notify: Restart mongod

        - name: "6.3 | PATCH | Ensure that server-side scripting is disabled if not needed | Update the parsed config"
          ansible.builtin.set_fact:
            discovered_mongod_conf: "{{ discovered_mongod_conf | combine(mongodb8_cis_6_3_settings, recursive=true) }}"
```

| Rule | Type | Does |
|------|------|------|
| 6.1 L1 | PATCH, **off** | Checks `mongodb8_cis_port` (1024–65535, not 27017) → SELinux label `mongod_port_t` → `net.port`. Port built inside `combine()` so it stays a number on 2.16 |
| 6.2 L2 | REPORT | Six limits from systemd vs CIS |
| 6.3 L2 | DECISION | Shows `javascriptEnabled`; disables it only with `mongodb8_cis_javascript_needed: false` |

**Test:** `mongodb8_cis_rule_6_1: true`, `mongodb8_cis_port: 47017` → mongod back on 47017 with SELinux enforcing; rerun `changed=0`. Revert the VM after.

---

## Batch 7 — database access (pymongo) + Section 3

Uses `community.mongodb.mongodb_info` (users) and `mongodb_shell` (3.4 privileges), with pymongo in a venv on the server (D13a).

`tasks/pymongo.yml`
```yaml
---
# Not a CIS rule: prepares the database modules used by sections 2 and 3 (D13a).
- name: "PREPARE | PATCH | Install Python 3.12 for pymongo"
  ansible.builtin.dnf:
    name: "{{ mongodb8_cis_python_packages[ansible_facts['distribution_major_version']] }}"
    state: present

- name: "PREPARE | PATCH | Install pymongo in its own virtualenv"
  ansible.builtin.pip:
    name: "{{ mongodb8_cis_pymongo_spec }}"
    virtualenv: "{{ mongodb8_cis_pymongo_venv }}"
    virtualenv_command: "{{ mongodb8_cis_python_bin }} -m venv"
```

Add to `tasks/main.yml` (after section 1). **For now the block imports only section 3**; section 2 comes in batch 8:
```yaml

- name: Prepare database access for sections 2 and 3 (pymongo, not a CIS rule)
  when: mongodb8_cis_section2 or mongodb8_cis_section3
  ansible.builtin.import_tasks:
    file: pymongo.yml

- name: Run sections 2 and 3 - Authentication and Authorization
  vars:
    # The database modules need pymongo, which lives in the role's virtualenv on the server.
    ansible_python_interpreter: "{{ mongodb8_cis_pymongo_venv }}/bin/python"
  module_defaults:
    community.mongodb.mongodb_info: &mongodb8_cis_login
      login_host: "{{ mongodb8_cis_shell_host }}"
      login_port: "{{ mongodb8_cis_shell_port | int }}"
      login_database: admin
      login_user: "{{ mongodb8_cis_admin_user if mongodb8_cis_shell_auth else omit }}"
      login_password: "{{ mongodb8_cis_admin_password if mongodb8_cis_shell_auth else omit }}"
      tls: "{{ mongodb8_cis_shell_tls }}"
      tlsCAFile: "{{ mongodb8_cis_shell_ca_file if mongodb8_cis_shell_tls and mongodb8_cis_shell_ca_file | length > 0 else omit }}"
      tlsCertificateKeyFile: "{{ mongodb8_cis_shell_client_cert if mongodb8_cis_shell_tls and mongodb8_cis_shell_client_cert | length > 0 else omit }}"
      # The role talks to its own mongod on this host; certificates are usually issued for the FQDN, not this address.
      connection_options: "{{ [{'tlsAllowInvalidHostnames': true}] if mongodb8_cis_shell_tls else omit }}"
    community.mongodb.mongodb_user: *mongodb8_cis_login
    community.mongodb.mongodb_shell:
      db: admin
      login_host: "{{ mongodb8_cis_shell_host }}"
      login_port: "{{ mongodb8_cis_shell_port | int }}"
      login_database: admin
      login_user: "{{ mongodb8_cis_admin_user if mongodb8_cis_shell_auth else omit }}"
      login_password: "{{ mongodb8_cis_admin_password if mongodb8_cis_shell_auth else omit }}"
      tls: "{{ mongodb8_cis_shell_tls }}"
      ssl_ca_certs: "{{ mongodb8_cis_shell_ca_file if mongodb8_cis_shell_tls and mongodb8_cis_shell_ca_file | length > 0 else omit }}"
      ssl_keyfile: "{{ mongodb8_cis_shell_client_cert if mongodb8_cis_shell_tls and mongodb8_cis_shell_client_cert | length > 0 else omit }}"
      additional_args: "{{ {'tlsAllowInvalidHostnames': true} if mongodb8_cis_shell_tls else omit }}"
  block:
    - name: Run section 3 - Authorization
      when: mongodb8_cis_section3
      ansible.builtin.import_tasks:
        file: section_3/main.yml
```

Add to `prelim.yml`, after "Keep the config mongod is running with":
```yaml
- name: "PRELIM | AUDIT | Check the database login values"
  when: mongodb8_cis_section2 or mongodb8_cis_section3
  ansible.builtin.assert:
    that:
      - not mongodb8_cis_shell_auth or (mongodb8_cis_admin_user | length > 0 and mongodb8_cis_admin_password | length > 0)
      - mongodb8_cis_admin_password is not search('[\\s\\x22\\x27\\x5c]')
    fail_msg: >-
      Sections 2 and 3 read the database. Authorization is on, so set mongodb8_cis_admin_user and
      mongodb8_cis_admin_password (or turn off mongodb8_cis_section2/3). The password must not contain spaces,
      quotes or backslashes (mongodb_shell passes it unquoted).
    quiet: true
```

`tasks/section_3/main.yml`
```yaml
---
- name: "SECTION | 3.1 | Ensure least privilege for database accounts"
  ansible.builtin.import_tasks:
    file: cis_3.1.yml

- name: "SECTION | 3.2 | Ensure that role-based access control is enabled and configured appropriately"
  ansible.builtin.import_tasks:
    file: cis_3.2.yml

- name: "SECTION | 3.3 | Ensure that MongoDB is run using a non-privileged, dedicated service account"
  ansible.builtin.import_tasks:
    file: cis_3.3.yml

- name: "SECTION | 3.4 | Ensure that each role for each MongoDB database is needed and grants only the necessary privileges"
  ansible.builtin.import_tasks:
    file: cis_3.4.yml

- name: "SECTION | 3.5 | Review Superuser/Admin Roles"
  ansible.builtin.import_tasks:
    file: cis_3.5.yml
```

`tasks/section_3/cis_3.1.yml`
```yaml
---
- name: "3.1 | AUDIT | Ensure least privilege for database accounts"
  when:
    - mongodb8_cis_rule_3_1
    - mongodb8_cis_level_1
  tags:
    - level1
    - manual
    - audit
    - rule_3.1
    - authorization
  vars:
    mongodb8_cis_3_1_roles: [dbOwner, userAdmin, userAdminAnyDatabase]
    # mongodb_info: users = {database: {user name: details}}; each user's roles come as a text string (str()).
    mongodb8_cis_3_1_users: "{{ discovered_3_1_info['users'] | dict2items | map(attribute='value') | map('dict2items') | flatten(levels=1) }}"
  block:
    - name: "3.1 | AUDIT | Ensure least privilege for database accounts | Read database users"
      community.mongodb.mongodb_info:
        filter: users
      register: discovered_3_1_info
      check_mode: false

    - name: "3.1 | AUDIT | Ensure least privilege for database accounts | Report each account with dbOwner, userAdmin, userAdminAnyDatabase in admin"
      when: >-
        item['value']['roles'] | from_yaml | selectattr('db', 'equalto', 'admin') | map(attribute='role')
        | intersect(mongodb8_cis_3_1_roles) | length > 0
      ansible.builtin.debug:
        msg: >-
          3.1 REVIEW: {{ item['value']['_id'] }} holds
          {{ item['value']['roles'] | from_yaml | selectattr('db', 'equalto', 'admin') | map(attribute='role') | intersect(mongodb8_cis_3_1_roles) | join(', ') }}
          in admin. CIS: drop these roles. (Users of databases that hold no data yet are not listed by mongodb_info.)
      loop: "{{ mongodb8_cis_3_1_users }}"
      loop_control:
        label: "{{ item['value']['_id'] }}"

    - name: "3.1 | AUDIT | Ensure least privilege for database accounts | Report PASS when there are none"
      when: >-
        mongodb8_cis_3_1_users | map(attribute='value.roles') | map('from_yaml') | flatten
        | selectattr('db', 'equalto', 'admin') | map(attribute='role') | intersect(mongodb8_cis_3_1_roles) | length == 0
      ansible.builtin.debug:
        msg: >-
          3.1 PASS: no account holds dbOwner, userAdmin or userAdminAnyDatabase in admin.
          (Users of databases that hold no data yet are not listed by mongodb_info.)
```

`tasks/section_3/cis_3.2.yml`
```yaml
---
- name: "3.2 | AUDIT | Ensure that role-based access control is enabled and configured appropriately"
  when:
    - mongodb8_cis_rule_3_2
    - mongodb8_cis_level_1
  tags:
    - level1
    - manual
    - audit
    - rule_3.2
    - authorization
  vars:
    mongodb8_cis_3_2_users: "{{ discovered_3_2_info['users'] | dict2items | map(attribute='value') | map('dict2items') | flatten(levels=1) }}"
  block:
    - name: "3.2 | AUDIT | Ensure that role-based access control is enabled and configured appropriately | Read database users"
      community.mongodb.mongodb_info:
        filter: users
      register: discovered_3_2_info
      check_mode: false

    - name: "3.2 | AUDIT | Ensure that role-based access control is enabled and configured appropriately | Report authorization"
      ansible.builtin.debug:
        msg: >-
          3.2 REVIEW: security.authorization = {{ discovered_mongod_running_conf['security']['authorization'] | default('disabled') }}
          (running config), {{ mongodb8_cis_3_2_users | length }} user(s). Check each user's roles below.

    - name: "3.2 | AUDIT | Ensure that role-based access control is enabled and configured appropriately | Report each user"
      ansible.builtin.debug:
        msg: >-
          3.2 REVIEW: {{ item['value']['_id'] }} roles:
          {{ item['value']['roles'] | from_yaml | map(attribute='role') | zip(item['value']['roles'] | from_yaml | map(attribute='db'))
             | map('join', '@') | join(', ') or 'none' }}.
      loop: "{{ mongodb8_cis_3_2_users }}"
      loop_control:
        label: "{{ item['value']['_id'] }}"
```

`tasks/section_3/cis_3.3.yml`
```yaml
---
- name: "3.3 | AUDIT | Ensure that MongoDB is run using a non-privileged, dedicated service account"
  when:
    - mongodb8_cis_rule_3_3
    - mongodb8_cis_level_1
  tags:
    - level1
    - manual
    - audit
    - rule_3.3
    - authorization
  vars:
    # An empty User= in the unit means systemd runs the service as root.
    mongodb8_cis_3_3_unit_user: "{{ discovered_mongod_service['status']['User'] | default('', true) or 'root' }}"
    mongodb8_cis_3_3_pid: "{{ discovered_mongod_service['status']['MainPID'] | default('0') | int }}"
  block:
    - name: "3.3 | AUDIT | Ensure that MongoDB is run using a non-privileged, dedicated service account | Get the mongod process owner"
      when: mongodb8_cis_3_3_pid | int > 0
      ansible.builtin.stat:
        path: "/proc/{{ mongodb8_cis_3_3_pid }}"
        get_checksum: false
      register: discovered_3_3_proc

    - name: "3.3 | AUDIT | Ensure that MongoDB is run using a non-privileged, dedicated service account | Report"
      vars:
        mongodb8_cis_3_3_owner: "{{ discovered_3_3_proc['stat']['pw_name'] | default(mongodb8_cis_3_3_unit_user) }}"
      ansible.builtin.debug:
        msg: >-
          3.3 {{ 'PASS' if mongodb8_cis_3_3_owner != 'root' and mongodb8_cis_3_3_unit_user != 'root' else 'FAIL' }}:
          unit User = {{ mongodb8_cis_3_3_unit_user }}, running mongod (PID {{ mongodb8_cis_3_3_pid }}) owned by {{ mongodb8_cis_3_3_owner }}.
          File permissions for this account are checked by 7.1 and 7.2.
```

`tasks/section_3/cis_3.4.yml`
```yaml
---
- name: "3.4 | AUDIT | Ensure that each role for each MongoDB database is needed and grants only the necessary privileges"
  when:
    - mongodb8_cis_rule_3_4
    - mongodb8_cis_level_1
  tags:
    - level1
    - manual
    - audit
    - rule_3.4
    - authorization
  block:
    - name: "3.4 | AUDIT | Ensure that each role for each MongoDB database is needed and grants only the necessary privileges | List user-defined roles"
      community.mongodb.mongodb_shell:
        eval: "db.getSiblingDB('admin').system.roles.find({}, {_id: 0, role: 1, db: 1, privileges: 1, roles: 1}).toArray()"
        transform: json
      register: discovered_3_4_roles
      changed_when: false
      check_mode: false

    - name: "3.4 | AUDIT | Ensure that each role for each MongoDB database is needed and grants only the necessary privileges | Report"
      ansible.builtin.debug:
        msg: >-
          3.4 REVIEW: {{ discovered_3_4_roles['transformed_output'] | length }} user-defined role(s); built-in roles are fixed by MongoDB.
          Check each role below is needed.

    - name: "3.4 | AUDIT | Ensure that each role for each MongoDB database is needed and grants only the necessary privileges | Report each role"
      ansible.builtin.debug:
        msg: >-
          3.4 REVIEW: {{ item['role'] }}@{{ item['db'] }} actions:
          {{ item['privileges'] | map(attribute='actions') | flatten | unique | join(', ') or 'none' }};
          inherits: {{ item['roles'] | map(attribute='role') | join(', ') or 'none' }}.
      loop: "{{ discovered_3_4_roles['transformed_output'] }}"
      loop_control:
        label: "{{ item['role'] }}@{{ item['db'] }}"
```

`tasks/section_3/cis_3.5.yml`
```yaml
---
- name: "3.5 | AUDIT | Review Superuser/Admin Roles"
  when:
    - mongodb8_cis_rule_3_5
    - mongodb8_cis_level_2
  tags:
    - level2
    - manual
    - audit
    - rule_3.5
    - authorization
  vars:
    mongodb8_cis_3_5_roles:
      - root
      - dbOwner
      - userAdmin
      - userAdminAnyDatabase
      - readWriteAnyDatabase
      - dbAdminAnyDatabase
      - clusterAdmin
      - hostManager
    mongodb8_cis_3_5_users: "{{ discovered_3_5_info['users'] | dict2items | map(attribute='value') | map('dict2items') | flatten(levels=1) }}"
  block:
    - name: "3.5 | AUDIT | Review Superuser/Admin Roles | Read database users"
      community.mongodb.mongodb_info:
        filter: users
      register: discovered_3_5_info
      check_mode: false

    - name: "3.5 | AUDIT | Review Superuser/Admin Roles | Report each user with a superuser or admin role"
      when: item['value']['roles'] | from_yaml | map(attribute='role') | intersect(mongodb8_cis_3_5_roles) | length > 0
      ansible.builtin.debug:
        msg: >-
          3.5 REVIEW: {{ item['value']['_id'] }} holds
          {{ item['value']['roles'] | from_yaml | map(attribute='role') | intersect(mongodb8_cis_3_5_roles) | join(', ') }}.
          Check this is needed.
      loop: "{{ mongodb8_cis_3_5_users }}"
      loop_control:
        label: "{{ item['value']['_id'] }}"
```

| Piece | Does | Why |
|-------|------|-----|
| `pymongo.yml` | `python3.12` (+pip) from the OS repos → venv `/opt/mongodb8_cis/venv` with `pymongo>=4.9,<5` | pymongo 4.9+ fully supports MongoDB 8.0 and needs Python ≥ 3.9 (RHEL 8's system Python is 3.6) |
| Block `vars: ansible_python_interpreter` | Runs sections 2/3 with the venv's Python | That's where pymongo is |
| `module_defaults` (`&mongodb8_cis_login` anchor) | Login/TLS settings written once for `mongodb_info` and `mongodb_user` (`*mongodb8_cis_login` reuses them); separate ones for `mongodb_shell` | Settings come from the **running** config, so they stay right while this run changes port/TLS |
| Prelim login check | If auth is already on, user/password must be set | The DB reads must be able to log in |
| 3.1 / 3.2 / 3.5 | `mongodb_info filter: users` → flatten `{db: {user: details}}` → each user's `roles` text → `from_yaml` | `mongodb_info` returns roles as a **string** (`str()` in its source) |
| 3.4 | `mongodb_shell` `rolesInfo`-style read with privileges | `mongodb_info` has no privileges; CIS 3.4 asks for them |
| 3.3 | process owner from systemd + `/proc` | No database needed |

**Known gap (accepted):** `mongodb_info` only lists databases that **hold data**, so users created for an empty database
are not reported by 3.1/3.2/3.5 (tested: `app.appuser` invisible). CIS's audit reads `admin.system.users`, which has all users.

**Test:** run → venv created (`/opt/mongodb8_cis/venv/bin/python -c 'import pymongo; print(pymongo.version)'` ≥ 4.9);
3.1 PASS, 3.2 "0 user(s)", 3.3 PASS, 3.4 "0 user-defined role(s)"; rerun `changed=0`.

---

## Batch 8 — Section 2: Authentication

Add the section 2 import **inside** the block in `tasks/main.yml`, before section 3:
```yaml
    - name: Run section 2 - Authentication
      when: mongodb8_cis_section2
      ansible.builtin.import_tasks:
        file: section_2/main.yml

```

`tasks/section_2/main.yml`
```yaml
---
- name: "SECTION | 2.1 | Ensure Authentication is configured"
  ansible.builtin.import_tasks:
    file: cis_2.1.yml

- name: "SECTION | 2.2 | Ensure that MongoDB does not bypass authentication via the localhost exception"
  ansible.builtin.import_tasks:
    file: cis_2.2.yml

- name: "SECTION | 2.3 | Ensure authentication is enabled in the sharded cluster"
  ansible.builtin.import_tasks:
    file: cis_2.3.yml
```

`tasks/section_2/cis_2.1.yml`
```yaml
---
- name: "2.1 | PATCH | Ensure Authentication is configured"
  when:
    - mongodb8_cis_rule_2_1
    - mongodb8_cis_level_1
    - discovered_mongod_conf['security']['authorization'] | default('disabled') != 'enabled'
  tags:
    - level1
    - automated
    - patch
    - rule_2.1
    - authentication
  vars:
    mongodb8_cis_2_1_settings:
      security:
        authorization: enabled
  block:
    - name: "2.1 | PATCH | Ensure Authentication is configured | Check the admin user values"
      ansible.builtin.assert:
        that:
          - mongodb8_cis_admin_user | length > 0
          - mongodb8_cis_admin_password | length > 0
        fail_msg: >-
          2.1: set mongodb8_cis_admin_user and mongodb8_cis_admin_password (keep the password in Ansible Vault).
          The admin user is created before authorization is enabled.

    # CIS remediation: create the user administrator first (role root on admin), then enable authorization.
    - name: "2.1 | PATCH | Ensure Authentication is configured | Create the admin user"
      community.mongodb.mongodb_user:
        name: "{{ mongodb8_cis_admin_user }}"
        password: "{{ mongodb8_cis_admin_password }}"
        database: admin
        roles:
          - db: admin
            role: root
        update_password: on_create
        state: present
      no_log: true

    - name: "2.1 | PATCH | Ensure Authentication is configured | Set security.authorization: enabled"
      ansible.builtin.copy:
        dest: "{{ mongodb8_cis_conf_path }}"
        content: "{{ discovered_mongod_conf | combine(mongodb8_cis_2_1_settings, recursive=true) | to_nice_yaml(indent=2) }}"
        owner: root
        group: root
        mode: "0644"
        backup: true
      notify: Restart mongod

    - name: "2.1 | PATCH | Ensure Authentication is configured | Update the parsed config"
      ansible.builtin.set_fact:
        discovered_mongod_conf: "{{ discovered_mongod_conf | combine(mongodb8_cis_2_1_settings, recursive=true) }}"
```

`tasks/section_2/cis_2.2.yml`
```yaml
---
- name: "2.2 | PATCH | Ensure that MongoDB does not bypass authentication via the localhost exception"
  when:
    - mongodb8_cis_rule_2_2
    - mongodb8_cis_level_1
    - discovered_mongod_conf['setParameter']['enableLocalhostAuthBypass'] | default(true) | bool
  tags:
    - level1
    - automated
    - patch
    - rule_2.2
    - authentication
  vars:
    mongodb8_cis_2_2_settings:
      setParameter:
        enableLocalhostAuthBypass: false
  block:
    - name: "2.2 | AUDIT | Ensure that MongoDB does not bypass authentication via the localhost exception | Read database users"
      community.mongodb.mongodb_info:
        filter: users
      register: discovered_2_2_info
      check_mode: false

    # In --check, 2.1 only simulates creating the admin user, so the count can still be 0.
    - name: "2.2 | PATCH | Ensure that MongoDB does not bypass authentication via the localhost exception | Check a user exists"
      vars:
        # users = {database: {user name: details}}: count the users of every database.
        mongodb8_cis_2_2_count: "{{ discovered_2_2_info['users'] | dict2items | map(attribute='value') | map('length') | sum }}"
      ansible.builtin.assert:
        that:
          - mongodb8_cis_2_2_count | int > 0 or ansible_check_mode
        fail_msg: >-
          2.2: no database user exists. Without the localhost exception nobody could log in once authorization is on.
          Create a user first (rule 2.1).

    - name: "2.2 | PATCH | Ensure that MongoDB does not bypass authentication via the localhost exception | Set enableLocalhostAuthBypass: false"
      ansible.builtin.copy:
        dest: "{{ mongodb8_cis_conf_path }}"
        content: "{{ discovered_mongod_conf | combine(mongodb8_cis_2_2_settings, recursive=true) | to_nice_yaml(indent=2) }}"
        owner: root
        group: root
        mode: "0644"
        backup: true
      notify: Restart mongod

    - name: "2.2 | PATCH | Ensure that MongoDB does not bypass authentication via the localhost exception | Update the parsed config"
      ansible.builtin.set_fact:
        discovered_mongod_conf: "{{ discovered_mongod_conf | combine(mongodb8_cis_2_2_settings, recursive=true) }}"
```

`tasks/section_2/cis_2.3.yml`
```yaml
---
- name: "2.3 | AUDIT | Ensure authentication is enabled in the sharded cluster"
  when:
    - mongodb8_cis_rule_2_3
    - mongodb8_cis_level_2
  tags:
    - level2
    - automated
    - audit
    - rule_2.3
    - authentication
  ansible.builtin.debug:
    msg: >-
      2.3 {{ 'REVIEW' if discovered_mongod_conf['sharding']['clusterRole'] is defined else 'NOT APPLICABLE (standalone, no sharding.clusterRole)' }}:
      security.clusterAuthMode = {{ discovered_mongod_conf['security']['clusterAuthMode'] | default('not set') }},
      security.keyFile = {{ discovered_mongod_conf['security']['keyFile'] | default('not set') }}.
      CIS: x509 in production, keyFile for development only.
```

| Rule | Type | Does |
|------|------|------|
| 2.1 L1 | PATCH, **off** | `mongodb_user` creates the admin (`root` on `admin`, as in CIS's own remediation; `update_password: on_create` keeps reruns at `changed=0`; `no_log`) → `security.authorization: enabled` |
| 2.2 L1 | PATCH, **off** | `mongodb_info` counts users (stops if none: lockout guard) → `enableLocalhostAuthBypass: false` |
| 2.3 L2 | REPORT, off | Sharded clusters only |

**Test (passed on a real mongod 8.0.32):** `rule_2_1`/`rule_2_2: true` + admin user/password (Vault) → admin created,
auth on, bypass off, one restart; rerun logs in → `changed=0`, 3.2 lists `admin.<user> roles: root@admin`;
`mongosh` without login → *"Command usersInfo requires authentication"*.

---

## Batch 9 — Section 4: Data Encryption

Add to `tasks/main.yml` (after the sections 2+3 block):
```yaml
- name: Run section 4 - Data Encryption
  when: mongodb8_cis_section4
  ansible.builtin.import_tasks:
    file: section_4/main.yml
```

Add to `prelim.yml`, after "Keep the config mongod is running with":
```yaml
- name: "PRELIM | AUDIT | Check mongod.conf uses net.tls, not the deprecated net.ssl"
  when: mongodb8_cis_section4
  ansible.builtin.assert:
    that:
      - discovered_mongod_conf['net']['ssl'] is not defined
    fail_msg: >-
      {{ mongodb8_cis_conf_path }} has a net.ssl block (deprecated since MongoDB 4.2). Move it to net.tls by hand
      before running Section 4: mongod refuses net.ssl and net.tls together.
    quiet: true
```

`tasks/section_4/main.yml`
```yaml
---
# 4.3 runs first: mongod refuses net.tls options (4.1, 4.2, 4.4) unless TLS is enabled (D25).
- name: "SECTION | 4.3 | Ensure Encryption of Data in Transit TLS or SSL (Transport Encryption)"
  ansible.builtin.import_tasks:
    file: cis_4.3.yml

- name: "SECTION | 4.1 | Ensure legacy TLS protocols are disabled"
  ansible.builtin.import_tasks:
    file: cis_4.1.yml

- name: "SECTION | 4.2 | Ensure Weak Protocols are Disabled"
  ansible.builtin.import_tasks:
    file: cis_4.2.yml

- name: "SECTION | 4.4 | Ensure Federal Information Processing Standard (FIPS) is enabled"
  ansible.builtin.import_tasks:
    file: cis_4.4.yml

- name: "SECTION | 4.5 | Ensure Encryption of Data at Rest"
  ansible.builtin.import_tasks:
    file: cis_4.5.yml
```

`tasks/section_4/cis_4.3.yml`
```yaml
---
- name: "4.3 | PATCH | Ensure Encryption of Data in Transit TLS or SSL (Transport Encryption)"
  when:
    - mongodb8_cis_rule_4_3
    - mongodb8_cis_level_1
    - discovered_mongod_conf['net']['tls']['mode'] | default('disabled') != 'requireTLS'
  tags:
    - level1
    - automated
    - patch
    - rule_4.3
    - tls
  vars:
    mongodb8_cis_4_3_key_file: >-
      {{ mongodb8_cis_tls_certificate_key_file or discovered_mongod_conf['net']['tls']['certificateKeyFile'] | default('') }}
    mongodb8_cis_4_3_ca_file: "{{ mongodb8_cis_tls_ca_file or discovered_mongod_conf['net']['tls']['CAFile'] | default('') }}"
  block:
    - name: "4.3 | PATCH | Ensure Encryption of Data in Transit TLS or SSL (Transport Encryption) | Get the PEM files"
      ansible.builtin.stat:
        path: "{{ item }}"
        get_checksum: false
      loop:
        - "{{ mongodb8_cis_4_3_key_file }}"
        - "{{ mongodb8_cis_4_3_ca_file }}"
      register: discovered_4_3_files

    - name: "4.3 | PATCH | Ensure Encryption of Data in Transit TLS or SSL (Transport Encryption) | Check the PEM files"
      ansible.builtin.assert:
        that:
          - mongodb8_cis_4_3_key_file | length > 0
          - mongodb8_cis_4_3_ca_file | length > 0
          - discovered_4_3_files['results'] | selectattr('stat.exists') | list | length == 2
        fail_msg: >-
          4.3: set mongodb8_cis_tls_certificate_key_file and mongodb8_cis_tls_ca_file to PEM files that exist on the host
          (found '{{ mongodb8_cis_4_3_key_file }}', '{{ mongodb8_cis_4_3_ca_file }}').
        quiet: true

    - name: "4.3 | PATCH | Ensure Encryption of Data in Transit TLS or SSL (Transport Encryption) | Set net.tls mode requireTLS"
      ansible.builtin.copy:
        dest: "{{ mongodb8_cis_conf_path }}"
        content: >-
          {{ discovered_mongod_conf | combine({'net': {'tls': {'mode': 'requireTLS', 'certificateKeyFile': mongodb8_cis_4_3_key_file,
          'CAFile': mongodb8_cis_4_3_ca_file}}}, recursive=true) | to_nice_yaml(indent=2) }}
        owner: root
        group: root
        mode: "0644"
        backup: true
      notify: Restart mongod

    - name: "4.3 | PATCH | Ensure Encryption of Data in Transit TLS or SSL (Transport Encryption) | Update the parsed config"
      ansible.builtin.set_fact:
        discovered_mongod_conf: >-
          {{ discovered_mongod_conf | combine({'net': {'tls': {'mode': 'requireTLS', 'certificateKeyFile': mongodb8_cis_4_3_key_file,
          'CAFile': mongodb8_cis_4_3_ca_file}}}, recursive=true) }}
```

`tasks/section_4/cis_4.1.yml`
```yaml
---
- name: "4.1 | PATCH | Ensure legacy TLS protocols are disabled"
  when:
    - mongodb8_cis_rule_4_1
    - mongodb8_cis_level_2
    - not (['TLS1_0', 'TLS1_1'] is subset(mongodb8_cis_4_1_disabled))
  tags:
    - level2
    - automated
    - patch
    - rule_4.1
    - tls
  vars:
    mongodb8_cis_4_1_disabled: "{{ (discovered_mongod_conf['net']['tls']['disabledProtocols'] | default('')).split(',') | map('trim') | select | list }}"
  block:
    - name: "4.1 | PATCH | Ensure legacy TLS protocols are disabled | Report when TLS is off"
      when: not mongodb8_cis_tls_enabled
      ansible.builtin.debug:
        msg: "4.1 FAIL: TLS is not enabled (net.tls.mode). mongod refuses disabledProtocols without it: configure TLS first (rule 4.3)."

    - name: "4.1 | PATCH | Ensure legacy TLS protocols are disabled | Set net.tls.disabledProtocols"
      when: mongodb8_cis_tls_enabled
      ansible.builtin.copy:
        dest: "{{ mongodb8_cis_conf_path }}"
        content: >-
          {{ discovered_mongod_conf | combine({'net': {'tls': {'disabledProtocols': (mongodb8_cis_4_1_disabled + ['TLS1_0', 'TLS1_1'])
          | unique | join(',')}}}, recursive=true) | to_nice_yaml(indent=2) }}
        owner: root
        group: root
        mode: "0644"
        backup: true
      notify: Restart mongod

    - name: "4.1 | PATCH | Ensure legacy TLS protocols are disabled | Update the parsed config"
      when: mongodb8_cis_tls_enabled
      ansible.builtin.set_fact:
        discovered_mongod_conf: >-
          {{ discovered_mongod_conf | combine({'net': {'tls': {'disabledProtocols': (mongodb8_cis_4_1_disabled + ['TLS1_0', 'TLS1_1'])
          | unique | join(',')}}}, recursive=true) }}
```

`tasks/section_4/cis_4.2.yml`
```yaml
---
- name: "4.2 | PATCH | Ensure Weak Protocols are Disabled"
  when:
    - mongodb8_cis_rule_4_2
    - mongodb8_cis_level_1
    - not (['TLS1_0', 'TLS1_1'] is subset(mongodb8_cis_4_2_disabled))
  tags:
    - level1
    - automated
    - patch
    - rule_4.2
    - tls
  vars:
    mongodb8_cis_4_2_disabled: "{{ (discovered_mongod_conf['net']['tls']['disabledProtocols'] | default('')).split(',') | map('trim') | select | list }}"
  block:
    - name: "4.2 | PATCH | Ensure Weak Protocols are Disabled | Report when TLS is off"
      when: not mongodb8_cis_tls_enabled
      ansible.builtin.debug:
        msg: "4.2 FAIL: TLS is not enabled (net.tls.mode). mongod refuses disabledProtocols without it: configure TLS first (rule 4.3)."

    - name: "4.2 | PATCH | Ensure Weak Protocols are Disabled | Set net.tls.disabledProtocols"
      when: mongodb8_cis_tls_enabled
      ansible.builtin.copy:
        dest: "{{ mongodb8_cis_conf_path }}"
        content: >-
          {{ discovered_mongod_conf | combine({'net': {'tls': {'disabledProtocols': (mongodb8_cis_4_2_disabled + ['TLS1_0', 'TLS1_1'])
          | unique | join(',')}}}, recursive=true) | to_nice_yaml(indent=2) }}
        owner: root
        group: root
        mode: "0644"
        backup: true
      notify: Restart mongod

    - name: "4.2 | PATCH | Ensure Weak Protocols are Disabled | Update the parsed config"
      when: mongodb8_cis_tls_enabled
      ansible.builtin.set_fact:
        discovered_mongod_conf: >-
          {{ discovered_mongod_conf | combine({'net': {'tls': {'disabledProtocols': (mongodb8_cis_4_2_disabled + ['TLS1_0', 'TLS1_1'])
          | unique | join(',')}}}, recursive=true) }}
```

`tasks/section_4/cis_4.4.yml`
```yaml
---
- name: "4.4 | PATCH | Ensure Federal Information Processing Standard (FIPS) is enabled"
  when:
    - mongodb8_cis_rule_4_4
    - mongodb8_cis_level_2
    - discovered_mongod_conf['net']['tls']['FIPSMode'] | default(false) is not true
  tags:
    - level2
    - automated
    - patch
    - rule_4.4
    - tls
  vars:
    mongodb8_cis_4_4_settings:
      net:
        tls:
          FIPSMode: true
  block:
    - name: "4.4 | PATCH | Ensure Federal Information Processing Standard (FIPS) is enabled | Report when TLS is off"
      when: not mongodb8_cis_tls_enabled
      ansible.builtin.debug:
        msg: "4.4 FAIL: net.tls.FIPSMode is not true. Not applied: TLS is not enabled (rule 4.3)."

    - name: "4.4 | PATCH | Ensure Federal Information Processing Standard (FIPS) is enabled | Set net.tls.FIPSMode: true"
      when: mongodb8_cis_tls_enabled
      block:
        - name: "4.4 | PATCH | Ensure Federal Information Processing Standard (FIPS) is enabled | Write mongod.conf"
          ansible.builtin.copy:
            dest: "{{ mongodb8_cis_conf_path }}"
            content: "{{ discovered_mongod_conf | combine(mongodb8_cis_4_4_settings, recursive=true) | to_nice_yaml(indent=2) }}"
            owner: root
            group: root
            mode: "0644"
            backup: true
          notify: Restart mongod

        - name: "4.4 | PATCH | Ensure Federal Information Processing Standard (FIPS) is enabled | Update the parsed config"
          ansible.builtin.set_fact:
            discovered_mongod_conf: "{{ discovered_mongod_conf | combine(mongodb8_cis_4_4_settings, recursive=true) }}"
```

`tasks/section_4/cis_4.5.yml`
```yaml
---
- name: "4.5 | AUDIT | Ensure Encryption of Data at Rest"
  when:
    - mongodb8_cis_rule_4_5
    - mongodb8_cis_level_2
  tags:
    - level2
    - manual
    - audit
    - rule_4.5
    - encryption
  ansible.builtin.debug:
    msg: >-
      4.5 {{ 'PASS' if discovered_mongod_conf['security']['enableEncryption'] | default(false) | bool else 'REVIEW' }}:
      security.enableEncryption = {{ discovered_mongod_conf['security']['enableEncryption'] | default('not set (false)') }};
      key management: {{ 'KMIP ' ~ discovered_mongod_conf['security']['kmip']['serverName']
      if discovered_mongod_conf['security']['kmip']['serverName'] is defined
      else ('keyfile ' ~ discovered_mongod_conf['security']['encryptionKeyFile']
      if discovered_mongod_conf['security']['encryptionKeyFile'] is defined else 'none') }}.
      CIS recommends KMIP; a local keyfile is the other option.
```

| Rule | Type | Does |
|------|------|------|
| 4.3 L1 | PATCH, **off**, runs first | PEM files must exist → `requireTLS` + key + CA |
| 4.1 L2 / 4.2 L1 | PATCH | Add `TLS1_0,TLS1_1` to `disabledProtocols` (FAIL message if TLS is off) |
| 4.4 L2 | PATCH, **off** | `FIPSMode: true` (needs TLS) |
| 4.5 L2 | REPORT | Encryption at rest + key management |

**Test:** put certs on the VM first (test project `prep-tls.yml`), then `rule_4_3: true` (+ `rule_4_4: true`) → requireTLS,
protocols off, FIPS log line; plain `mongosh` refused; rerun `changed=0`.

---

## Batch 10 — optional SELinux extra (not CIS)

Add at the end of `tasks/main.yml`:
```yaml

- name: Run the optional SELinux policy for mongod (not a CIS recommendation)
  when: mongodb8_cis_selinux_policy
  tags:
    - selinux_policy
  ansible.builtin.import_tasks:
    file: selinux.yml
```

`tasks/selinux.yml`
```yaml
---
# Optional extra, not a CIS recommendation (D17). MongoDB's policy module from github.com/mongodb/mongodb-selinux
# (commit 18181652, GPL-2.0-or-later) is copied in files/selinux/ and built on the host, as in the install docs (S1).

- name: "SELINUX | AUDIT | Report when SELinux is disabled"
  when: ansible_facts['selinux']['status'] | default('disabled') != 'enabled'
  ansible.builtin.debug:
    msg: "SELinux is disabled on {{ inventory_hostname }}: nothing to confine."

- name: "SELINUX | PATCH | Confine mongod with SELinux"
  when: ansible_facts['selinux']['status'] | default('disabled') == 'enabled'
  vars:
    mongodb8_cis_selinux_build: "{{ ansible_facts['distribution_major_version'] in mongodb8_cis_selinux_build_os_majors }}"
    mongodb8_cis_selinux_db_path: "{{ discovered_mongod_conf['storage']['dbPath'] | default(mongodb8_cis_default_db_path) }}"
    mongodb8_cis_selinux_log_dir: "{{ discovered_mongod_conf['systemLog']['path'] | default(mongodb8_cis_default_log_path) | dirname }}"
  block:
    - name: "SELINUX | PATCH | Confine mongod with SELinux | Install the SELinux tools"
      ansible.builtin.dnf:
        name: "{{ ['policycoreutils-python-utils'] + (['selinux-policy-devel'] if mongodb8_cis_selinux_build else []) }}"
        state: present

    - name: "SELINUX | AUDIT | Confine mongod with SELinux | Report RHEL 8"
      when: not mongodb8_cis_selinux_build
      ansible.builtin.debug:
        msg: >-
          RHEL {{ ansible_facts['distribution_major_version'] }}: the base policy's mongodb module already confines mongod
          (mongod_t); MongoDB's module does not build here. Only labels are added.

    - name: "SELINUX | PATCH | Confine mongod with SELinux | Build and load MongoDB's policy module"
      when: mongodb8_cis_selinux_build
      block:
        - name: "SELINUX | PATCH | Confine mongod with SELinux | Copy the module sources"
          ansible.builtin.copy:
            src: selinux/
            dest: "{{ mongodb8_cis_selinux_src_dir }}/"
            owner: root
            group: root
            mode: "0644"
            directory_mode: "0755"
          register: discovered_selinux_sources

        - name: "SELINUX | AUDIT | Confine mongod with SELinux | List loaded modules"
          ansible.builtin.command:
            cmd: semodule --list-modules=full
          register: discovered_selinux_modules
          changed_when: false
          check_mode: false

        - name: "SELINUX | PATCH | Confine mongod with SELinux | Build and load the module"
          when: >-
            discovered_selinux_sources['changed']
            or discovered_selinux_modules['stdout_lines'] | select('match', '^200 mongodb ') | list | length == 0
          block:
            - name: "SELINUX | PATCH | Confine mongod with SELinux | Build mongodb.pp"
              ansible.builtin.command:
                cmd: make -f /usr/share/selinux/devel/Makefile mongodb.pp
                chdir: "{{ mongodb8_cis_selinux_src_dir }}"
              changed_when: true

            - name: "SELINUX | PATCH | Confine mongod with SELinux | Load mongodb.pp at priority 200"
              ansible.builtin.command:
                cmd: semodule --priority 200 --install {{ mongodb8_cis_selinux_src_dir }}/mongodb.pp
              changed_when: true
              notify: Restart mongod

            # RHEL 9 base module labels mongod's socket files mongod_tmp_t; MongoDB's module has no such type, so the
            # restarted mongod may not unlink the old socket and aborts (fassert 40486). mongod recreates them on start.
            - name: "SELINUX | PATCH | Confine mongod with SELinux | Find mongod socket files left by the old policy"
              ansible.builtin.find:
                paths: "{{ discovered_mongod_conf['net']['unixDomainSocket']['pathPrefix'] | default('/tmp') }}"
                patterns: "mongodb-*.sock"
                file_type: any
              register: discovered_selinux_sockets

            - name: "SELINUX | PATCH | Confine mongod with SELinux | Remove mongod socket files left by the old policy"
              ansible.builtin.file:
                path: "{{ item['path'] }}"
                state: absent
              loop: "{{ discovered_selinux_sockets['files'] }}"
              loop_control:
                label: "{{ item['path'] }}"

    - name: "SELINUX | PATCH | Confine mongod with SELinux | Label a non-default dbPath and log directory"
      when: item['path'] != item['default']
      community.general.sefcontext:
        target: "{{ item['path'] }}(/.*)?"
        setype: "{{ item['type'] }}"
        state: present
      loop:
        - {path: "{{ mongodb8_cis_selinux_db_path }}", default: "{{ mongodb8_cis_default_db_path }}", type: mongod_var_lib_t}
        - {path: "{{ mongodb8_cis_selinux_log_dir }}", default: "{{ mongodb8_cis_default_log_path | dirname }}", type: mongod_log_t}
      loop_control:
        label: "{{ item['path'] }}"
      notify: Restart mongod

    - name: "SELINUX | PATCH | Confine mongod with SELinux | Label a non-default port"
      when: mongodb8_cis_listen_port | int not in mongodb8_cis_selinux_default_ports
      community.general.seport:
        ports: "{{ mongodb8_cis_listen_port | int }}"
        proto: tcp
        setype: mongod_port_t
        state: present

    - name: "SELINUX | PATCH | Confine mongod with SELinux | Restore file labels"
      ansible.builtin.command:
        cmd: >-
          restorecon -R -v /usr/bin/mongod /usr/lib/systemd/system/mongod.service
          {{ mongodb8_cis_selinux_db_path }} {{ mongodb8_cis_selinux_log_dir }}
      register: discovered_selinux_restorecon
      changed_when: discovered_selinux_restorecon['stdout'] | length > 0
      notify: Restart mongod

    - name: "SELINUX | AUDIT | Confine mongod with SELinux | Read the running mongod domain"
      when: discovered_mongod_service['status']['MainPID'] | default('0') | int > 0
      ansible.builtin.slurp:
        src: "/proc/{{ discovered_mongod_service['status']['MainPID'] }}/attr/current"
      register: discovered_selinux_domain

    - name: "SELINUX | AUDIT | Confine mongod with SELinux | Report"
      vars:
        mongodb8_cis_selinux_context: "{{ (discovered_selinux_domain['content'] | default('') | b64decode).strip('\x00\n') or 'not running' }}"
      ansible.builtin.debug:
        msg: >-
          SELinux {{ ansible_facts['selinux']['mode'] | default('') }}: mongod runs as {{ mongodb8_cis_selinux_context }}
          (read before this run's restart). Expected domain: mongod_t.
```

**Test:** `mongodb8_cis_selinux_policy: true` on RHEL 9 and 10 → `semodule -lfull | grep mongodb` shows `200 mongodb`,
mongod runs as `mongod_t`; RHEL 8 → message only; rerun `changed=0`.

---

## Appendix — final `tasks/main.yml` (compare when done)

`tasks/main.yml`
```yaml
---
- name: Run preliminary checks and discovery
  ansible.builtin.import_tasks:
    file: prelim.yml
  tags:
    - always

- name: Run section 1 - Installation and Patching
  when: mongodb8_cis_section1
  ansible.builtin.import_tasks:
    file: section_1/main.yml

- name: Prepare database access for sections 2 and 3 (pymongo, not a CIS rule)
  when: mongodb8_cis_section2 or mongodb8_cis_section3
  ansible.builtin.import_tasks:
    file: pymongo.yml

- name: Run sections 2 and 3 - Authentication and Authorization
  vars:
    # The database modules need pymongo, which lives in the role's virtualenv on the server.
    ansible_python_interpreter: "{{ mongodb8_cis_pymongo_venv }}/bin/python"
  module_defaults:
    community.mongodb.mongodb_info: &mongodb8_cis_login
      login_host: "{{ mongodb8_cis_shell_host }}"
      login_port: "{{ mongodb8_cis_shell_port | int }}"
      login_database: admin
      login_user: "{{ mongodb8_cis_admin_user if mongodb8_cis_shell_auth else omit }}"
      login_password: "{{ mongodb8_cis_admin_password if mongodb8_cis_shell_auth else omit }}"
      tls: "{{ mongodb8_cis_shell_tls }}"
      tlsCAFile: "{{ mongodb8_cis_shell_ca_file if mongodb8_cis_shell_tls and mongodb8_cis_shell_ca_file | length > 0 else omit }}"
      tlsCertificateKeyFile: "{{ mongodb8_cis_shell_client_cert if mongodb8_cis_shell_tls and mongodb8_cis_shell_client_cert | length > 0 else omit }}"
      # The role talks to its own mongod on this host; certificates are usually issued for the FQDN, not this address.
      connection_options: "{{ [{'tlsAllowInvalidHostnames': true}] if mongodb8_cis_shell_tls else omit }}"
    community.mongodb.mongodb_user: *mongodb8_cis_login
    community.mongodb.mongodb_shell:
      db: admin
      login_host: "{{ mongodb8_cis_shell_host }}"
      login_port: "{{ mongodb8_cis_shell_port | int }}"
      login_database: admin
      login_user: "{{ mongodb8_cis_admin_user if mongodb8_cis_shell_auth else omit }}"
      login_password: "{{ mongodb8_cis_admin_password if mongodb8_cis_shell_auth else omit }}"
      tls: "{{ mongodb8_cis_shell_tls }}"
      ssl_ca_certs: "{{ mongodb8_cis_shell_ca_file if mongodb8_cis_shell_tls and mongodb8_cis_shell_ca_file | length > 0 else omit }}"
      ssl_keyfile: "{{ mongodb8_cis_shell_client_cert if mongodb8_cis_shell_tls and mongodb8_cis_shell_client_cert | length > 0 else omit }}"
      additional_args: "{{ {'tlsAllowInvalidHostnames': true} if mongodb8_cis_shell_tls else omit }}"
  block:
    - name: Run section 2 - Authentication
      when: mongodb8_cis_section2
      ansible.builtin.import_tasks:
        file: section_2/main.yml

    - name: Run section 3 - Authorization
      when: mongodb8_cis_section3
      ansible.builtin.import_tasks:
        file: section_3/main.yml

- name: Run section 4 - Data Encryption
  when: mongodb8_cis_section4
  ansible.builtin.import_tasks:
    file: section_4/main.yml

- name: Run section 5 - Audit Logging
  when: mongodb8_cis_section5
  ansible.builtin.import_tasks:
    file: section_5/main.yml

- name: Run section 6 - Operating System Hardening
  when: mongodb8_cis_section6
  ansible.builtin.import_tasks:
    file: section_6/main.yml

- name: Run section 7 - File Permissions
  when: mongodb8_cis_section7
  ansible.builtin.import_tasks:
    file: section_7/main.yml

- name: Run the optional SELinux policy for mongod (not a CIS recommendation)
  when: mongodb8_cis_selinux_policy
  tags:
    - selinux_policy
  ansible.builtin.import_tasks:
    file: selinux.yml
```
