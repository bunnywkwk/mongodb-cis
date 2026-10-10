# Build guide: type the tasks

Every file under `tasks/` for `mongodb8_cis`, in the order to type them. Each file has its purpose, why it is
written this way, then the code. The code was run on a fresh Rocky 9 container (2026-10-10, ansible-core 2.20,
Level 2, risky rules on): first run `changed=14`, second run `changed=0`, no failures. Since then
changed after review (not yet run): 4.5 back to report only, TLS file copy (4.3), risky rules on by default with
`NOT APPLIED` reports when a value is missing. The role's other files
(`defaults/`, `vars/`, `handlers/`, `files/`, `meta/`) are already in place.

## The one pattern to understand first

`mongod.conf` is one YAML file, so the rules don't each write it. `prelim.yml` reads it into `mongodb8_cis_conf`;
every rule that changes the config adds its setting with

```yaml
ansible.builtin.set_fact:
  mongodb8_cis_conf: "{{ mongodb8_cis_conf | combine({'systemLog': {'logAppend': true}}, recursive=true) }}"
```

and `post.yml` writes the file once and restarts mongod once. `combine(recursive=true)` merges the new keys into the
existing ones, so the site's own settings (dbPath, bindIp, replication...) stay. This is the same pattern as
`chrome_cis` (each rule adds its policy, one task writes the file).

Other rules (users, roles) talk to the database with `community.mongodb.mongodb_shell`. Reads use
`changed_when: false`; changes run only for what you listed in `group_vars`.

## Every rule has the same shape

- Name `"<ID> | AUDIT or PATCH | <CIS title>"`; steps add ` | <what this step does>`.
- `when:` the rule switch and its level.
- `tags:` level, `automated`/`manual`, `patch`/`audit`, `rule_<id>`, a topic.
- Manual rules report with `debug`; their optional PATCH steps get their own `patch` tag.

## Check as you go

```bash
yamllint . && ansible-lint          # in mongodb-cis/
```

Then run it from the test project: fresh VM (base snapshot) → full run → second run must show `changed=0`.

## Files

### `tasks/main.yml`

Orchestration: prelim, then the 7 sections, the optional SELinux extra, then post.

- `prelim` and `post` are tagged `always`, so they also run with `--tags rule_5.4`.
- `module_defaults` gives every `mongodb_shell` in sections 2 and 3 the same login (host, port, admin, TLS), so the rules don't repeat it.
- Each section has a switch (`mongodb8_cis_section<N>`).

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

- name: Run sections 2 and 3 - Authentication and Authorization
  module_defaults:
    community.mongodb.mongodb_shell:
      db: admin
      login_host: "{{ mongodb8_cis_shell_host }}"
      login_port: "{{ mongodb8_cis_shell_port | int }}"
      login_database: admin
      login_user: "{{ mongodb8_cis_admin_user if mongodb8_cis_shell_auth else omit }}"
      login_password: "{{ mongodb8_cis_admin_password if mongodb8_cis_shell_auth else omit }}"
      tls: "{{ mongodb8_cis_shell_tls }}"
      ssl_ca_certs: "{{ mongodb8_cis_shell_ca_file if mongodb8_cis_shell_tls else omit }}"
      ssl_keyfile: "{{ mongodb8_cis_shell_client_cert if mongodb8_cis_shell_tls else omit }}"
      # The role talks to its own mongod on this host; certificates are usually issued for the FQDN, not this address.
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

- name: Write mongod.conf
  ansible.builtin.import_tasks:
    file: post.yml
  tags:
    - always
```

### `tasks/prelim.yml`

Checks and discovery that every rule needs.

- One assert for ansible-core, OS and requested version: fail fast with one clear message.
- Install is imported only when `mongodb8_cis_install: true`.
- Not installed and no install: the host ends cleanly (no failure).
- `discovered_mongod_conf` = the config mongod runs with now (read-only). `mongodb8_cis_conf` starts as the same config; each rule adds its settings to it, and `post.yml` writes it.
- The password check exists because `mongodb_shell` passes the password unquoted on the command line.

```yaml
---
- name: "PRELIM | AUDIT | Gather minimal facts if the play skipped them"
  when: ansible_facts['os_family'] is not defined
  ansible.builtin.setup:
    gather_subset:
      - min

- name: "PRELIM | AUDIT | Check ansible-core, OS and MongoDB version"
  ansible.builtin.assert:
    that:
      - ansible_version.full is version('2.16.1', '>=')
      - ansible_facts['os_family'] == 'RedHat'
      - ansible_facts['distribution_major_version'] in mongodb8_cis_supported_os_majors
      - mongodb8_cis_version in mongodb8_cis_supported_versions
    fail_msg: >-
      Needs ansible-core >= 2.16.1, RHEL-family {{ mongodb8_cis_supported_os_majors | join('/') }} and
      mongodb8_cis_version {{ mongodb8_cis_supported_versions | join(', ') }}. Found ansible-core {{ ansible_version.full }},
      {{ ansible_facts['distribution'] }} {{ ansible_facts['distribution_version'] }}, mongodb8_cis_version {{ mongodb8_cis_version }}.

- name: "PRELIM | PATCH | Install MongoDB"
  when: mongodb8_cis_install
  ansible.builtin.import_tasks:
    file: install.yml

- name: "PRELIM | AUDIT | Gather installed packages"
  ansible.builtin.package_facts:
    manager: rpm

- name: "PRELIM | AUDIT | Stop when MongoDB is not installed"
  when: mongodb8_cis_server_package not in ansible_facts['packages']
  block:
    - name: "PRELIM | AUDIT | Stop when MongoDB is not installed | Report"
      ansible.builtin.debug:
        msg: "{{ mongodb8_cis_server_package }} is not installed on {{ inventory_hostname }}: nothing to harden (mongodb8_cis_install: false)."

    - name: "PRELIM | AUDIT | Stop when MongoDB is not installed | End this host"
      ansible.builtin.meta: end_host

- name: "PRELIM | AUDIT | Check the installed MongoDB is 8.0"
  ansible.builtin.assert:
    that:
      - discovered_mongodb_version.startswith(mongodb8_cis_version ~ '.')
    fail_msg: "MongoDB {{ discovered_mongodb_version }} is installed; this role implements the CIS MongoDB 8 Benchmark (8.0 only)."

- name: "PRELIM | AUDIT | Read mongod.conf"
  ansible.builtin.slurp:
    src: "{{ mongodb8_cis_conf_path }}"
  register: discovered_mongod_conf_raw

# discovered_mongod_conf = the config mongod runs with now; mongodb8_cis_conf = what the rules want (written in post.yml).
- name: "PRELIM | AUDIT | Parse mongod.conf"
  ansible.builtin.set_fact:
    discovered_mongod_conf: "{{ discovered_mongod_conf_raw['content'] | b64decode | from_yaml | default({}, true) }}"
    mongodb8_cis_conf: "{{ discovered_mongod_conf_raw['content'] | b64decode | from_yaml | default({}, true) }}"

# mongodb_shell passes the password on the command line without quotes.
- name: "PRELIM | AUDIT | Check the admin login values"
  when: mongodb8_cis_section2 or mongodb8_cis_section3
  ansible.builtin.assert:
    that:
      - not mongodb8_cis_shell_auth or (mongodb8_cis_admin_user | length > 0 and mongodb8_cis_admin_password | length > 0)
      - mongodb8_cis_admin_password is not search('[\\s\\x22\\x27\\x5c]')
    fail_msg: >-
      Login is on: set mongodb8_cis_admin_user and mongodb8_cis_admin_password (no spaces, quotes or backslashes),
      or turn off mongodb8_cis_section2 and mongodb8_cis_section3.

- name: "PRELIM | AUDIT | Read the mongod service"
  ansible.builtin.systemd_service:
    name: "{{ mongodb8_cis_service }}"
  register: discovered_mongod_service
```

### `tasks/install.yml`
Opt-in install of MongoDB Enterprise from MongoDB's repo, then start mongod.

- Key, repo, package, start: all declarative, so a second run reports `ok`.
- mongod is started here because sections 2 and 3 talk to it.

```yaml
---
- name: "INSTALL | PATCH | Import the MongoDB package signing key"
  ansible.builtin.rpm_key:
    key: "{{ mongodb8_cis_repo_gpgkey }}"
    state: present

- name: "INSTALL | PATCH | Add the MongoDB Enterprise repository"
  ansible.builtin.yum_repository:
    name: "{{ mongodb8_cis_repo_name }}"
    description: MongoDB Enterprise Repository
    baseurl: "{{ mongodb8_cis_repo_baseurl }}"
    gpgkey: "{{ mongodb8_cis_repo_gpgkey }}"
    gpgcheck: true
    enabled: true

- name: "INSTALL | PATCH | Install MongoDB"
  ansible.builtin.dnf:
    name: "{{ mongodb8_cis_package }}"
    state: present

- name: "INSTALL | PATCH | Start and enable mongod"
  ansible.builtin.systemd_service:
    name: "{{ mongodb8_cis_service }}"
    state: started
    enabled: true
```

### `tasks/post.yml`

Writes `mongod.conf` once, from `mongodb8_cis_conf`.

- One write and one restart for the whole run, instead of one per rule.
- `copy` compares the content itself: if nothing changed, it reports `ok` (second run `changed=0`).
- `backup: true` keeps the previous file next to it.

```yaml
---
# Every rule above added its settings to mongodb8_cis_conf; mongod.conf is written once, and mongod restarted once.
- name: "POST | PATCH | Write mongod.conf"
  ansible.builtin.copy:
    dest: "{{ mongodb8_cis_conf_path }}"
    content: "{{ mongodb8_cis_conf | to_nice_yaml(indent=2) }}"
    owner: root
    group: root
    mode: "0644"
    backup: true
  notify: Restart mongod
```

### `tasks/section_1/main.yml`

Imports the rule files of section 1, in benchmark order.

```yaml
---
- name: "SECTION | 1.1 | Ensure the appropriate MongoDB software version/patches are installed"
  ansible.builtin.import_tasks:
    file: cis_1.1.yml
```

### `tasks/section_1/cis_1.1.yml`

1.1 (L1, Manual): report only.

- Upgrades restart the database, so they belong in a change window, not in a hardening run.

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
      1.1 REVIEW: MongoDB {{ discovered_mongodb_version }} is installed. Compare it with the latest {{ mongodb8_cis_version }}.x
      release and https://www.mongodb.com/alerts. The role never upgrades.
```

### `tasks/section_2/main.yml`

Imports the rule files of section 2.

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

### `tasks/section_2/cis_2.1.yml`

2.1 (L1, Automated): create the admin, then turn on login.

- The admin is created while login is still off (the order CIS gives), otherwise nobody could log in.
- `no_log` hides the password in the output.
- The last task only adds `security.authorization: enabled` to the config; `post.yml` writes it.
- On by default. No admin user/password set: it reports `NOT APPLIED` and skips (the run goes on).

```yaml
---
- name: "2.1 | PATCH | Ensure Authentication is configured"
  when:
    - mongodb8_cis_rule_2_1
    - mongodb8_cis_level_1
  tags:
    - level1
    - automated
    - patch
    - rule_2.1
    - authentication
  block:
    - name: "2.1 | AUDIT | Ensure Authentication is configured | Report when the admin values are missing"
      when: mongodb8_cis_admin_user | length == 0 or mongodb8_cis_admin_password | length == 0
      ansible.builtin.debug:
        msg: "2.1 NOT APPLIED: set mongodb8_cis_admin_user and mongodb8_cis_admin_password (the admin is created before login is turned on)."

    - name: "2.1 | AUDIT | Ensure Authentication is configured | Check whether the admin exists"
      when: mongodb8_cis_admin_user | length > 0 and mongodb8_cis_admin_password | length > 0
      community.mongodb.mongodb_shell:
        eval: "db.getSiblingDB('admin').getUser({{ mongodb8_cis_admin_user | to_json }}) != null"
        transform: raw
      register: discovered_2_1_admin
      changed_when: false

    # CIS remediation: create a user administrator, then enable authorization.
    - name: "2.1 | PATCH | Ensure Authentication is configured | Create the admin"
      when:
        - discovered_2_1_admin is not skipped
        - discovered_2_1_admin['transformed_output'] != 'true'
      community.mongodb.mongodb_shell:
        eval: >-
          db.getSiblingDB('admin').createUser({user: {{ mongodb8_cis_admin_user | to_json }},
          pwd: {{ mongodb8_cis_admin_password | to_json }}, roles: [{role: 'root', db: 'admin'}]})
      no_log: true

    - name: "2.1 | PATCH | Ensure Authentication is configured | Set security.authorization: enabled"
      when: discovered_2_1_admin is not skipped
      ansible.builtin.set_fact:
        mongodb8_cis_conf: "{{ mongodb8_cis_conf | combine({'security': {'authorization': 'enabled'}}, recursive=true) }}"
```

### `tasks/section_2/cis_2.2.yml`

2.2 (L1, Automated): no localhost login without a user.

- Counts users first: without a user and without the localhost exception, nobody could log in.
- No user yet: it reports `NOT APPLIED` and skips, instead of stopping the run.

```yaml
---
- name: "2.2 | PATCH | Ensure that MongoDB does not bypass authentication via the localhost exception"
  when:
    - mongodb8_cis_rule_2_2
    - mongodb8_cis_level_1
  tags:
    - level1
    - automated
    - patch
    - rule_2.2
    - authentication
  block:
    - name: "2.2 | AUDIT | Ensure that MongoDB does not bypass authentication via the localhost exception | Count users"
      community.mongodb.mongodb_shell:
        eval: "db.getSiblingDB('admin').system.users.countDocuments({})"
        transform: raw
      register: discovered_2_2_users
      changed_when: false

    # Without the localhost exception and without a user, nobody could log in.
    - name: "2.2 | AUDIT | Ensure that MongoDB does not bypass authentication via the localhost exception | Report when no user exists"
      when: discovered_2_2_users['transformed_output'] | int == 0
      ansible.builtin.debug:
        msg: "2.2 NOT APPLIED: no database user exists yet. Set the 2.1 admin first."

    - name: "2.2 | PATCH | Ensure that MongoDB does not bypass authentication via the localhost exception | Set enableLocalhostAuthBypass: false"
      when: discovered_2_2_users['transformed_output'] | int > 0
      ansible.builtin.set_fact:
        mongodb8_cis_conf: "{{ mongodb8_cis_conf | combine({'setParameter': {'enableLocalhostAuthBypass': false}}, recursive=true) }}"
```

### `tasks/section_2/cis_2.3.yml`

2.3 (L2, Automated): cluster members authenticate with x509.

- Standalone: report N/A, nothing changes.
- Member (replica set, shard or config server): CIS's production method, `clusterAuthMode: x509` + `clusterFile` (default: the server certificate).
- x509 needs TLS, which is rule 4.3, so the rule checks 4.3 is on.
- On a member without TLS ready (4.3 off or no certificate files): `NOT APPLIED`, because x509 without TLS stops mongod from starting.

```yaml
---
- name: "2.3 | PATCH | Ensure authentication is enabled in the sharded cluster"
  when:
    - mongodb8_cis_rule_2_3
    - mongodb8_cis_level_2
  tags:
    - level2
    - automated
    - patch
    - rule_2.3
    - authentication
  block:
    - name: "2.3 | AUDIT | Ensure authentication is enabled in the sharded cluster | Report a standalone"
      when: not mongodb8_cis_cluster_member
      ansible.builtin.debug:
        msg: "2.3 NOT APPLICABLE: standalone mongod (no sharding.clusterRole or replication.replSetName)."

    - name: "2.3 | AUDIT | Ensure authentication is enabled in the sharded cluster | Report when TLS will not be on"
      when: mongodb8_cis_cluster_member and not mongodb8_cis_tls_ready
      ansible.builtin.debug:
        msg: "2.3 NOT APPLIED: x509 member authentication needs TLS, and rule 4.3 is off or has no certificate files."

    # CIS remediation (production): x509 member authentication over TLS. TLS itself is rule 4.3.
    - name: "2.3 | PATCH | Ensure authentication is enabled in the sharded cluster | Set clusterAuthMode x509"
      when: mongodb8_cis_cluster_member and mongodb8_cis_tls_ready
      ansible.builtin.set_fact:
        mongodb8_cis_conf: >-
          {{ mongodb8_cis_conf | combine({'security': {'clusterAuthMode': 'x509'},
             'net': {'tls': {'clusterFile': mongodb8_cis_cluster_file or mongodb8_cis_tls_certificate_key_file}}}, recursive=true) }}
```

### `tasks/section_3/main.yml`

Imports the rule files of section 3.

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

### `tasks/section_3/cis_3.1.yml`

3.1 (L1, Manual): report accounts with dbOwner / userAdmin / userAdminAnyDatabase in admin; optional revoke list.

- The query reads `admin.system.users` (all users of all databases).
- Revoke only for accounts in `mongodb8_cis_revoke_admin_roles`; the account stays.

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
  block:
    - name: "3.1 | AUDIT | Ensure least privilege for database accounts | Find accounts with dbOwner, userAdmin or userAdminAnyDatabase in admin"
      community.mongodb.mongodb_shell:
        eval: >-
          db.getSiblingDB('admin').system.users.find({roles: {$elemMatch: {db: 'admin',
          role: {$in: ['dbOwner', 'userAdmin', 'userAdminAnyDatabase']}}}}, {_id: 1, user: 1, db: 1}).toArray()
        transform: json
      register: discovered_3_1_users
      changed_when: false

    - name: "3.1 | AUDIT | Ensure least privilege for database accounts | Report"
      ansible.builtin.debug:
        msg: >-
          3.1 {{ 'PASS' if discovered_3_1_users['transformed_output'] | length == 0 else 'REVIEW' }}:
          accounts with dbOwner, userAdmin or userAdminAnyDatabase in admin:
          {{ discovered_3_1_users['transformed_output'] | map(attribute='_id') | join(', ') or 'none' }}.

    # CIS remediation: drop these roles. Only from the accounts the site lists; the account itself stays.
    - name: "3.1 | PATCH | Ensure least privilege for database accounts | Revoke the roles from the listed accounts"
      when: item['_id'] in mongodb8_cis_revoke_admin_roles
      community.mongodb.mongodb_shell:
        eval: >-
          db.getSiblingDB({{ item['db'] | to_json }}).revokeRolesFromUser({{ item['user'] | to_json }},
          [{role: 'dbOwner', db: 'admin'}, {role: 'userAdmin', db: 'admin'}, {role: 'userAdminAnyDatabase', db: 'admin'}])
      loop: "{{ discovered_3_1_users['transformed_output'] }}"
      loop_control:
        label: "{{ item['_id'] }}"
      tags:
        - patch
```

### `tasks/section_3/cis_3.2.yml`

3.2 (L1, Manual): report every account and its roles; optional `mongodb8_cis_users`.

- Create: only listed accounts that don't exist yet (with their roles).
- Add missing roles: only listed accounts that already exist; `difference` = roles in your list that the account doesn't have.
- Nothing is ever removed, passwords are not changed, unlisted accounts are not touched. Empty list = report only.

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
  block:
    - name: "3.2 | AUDIT | Ensure that role-based access control is enabled and configured appropriately | List users and roles"
      community.mongodb.mongodb_shell:
        eval: "db.getSiblingDB('admin').system.users.find({}, {_id: 0, user: 1, db: 1, roles: 1}).toArray()"
        transform: json
      register: discovered_3_2_users
      changed_when: false

    - name: "3.2 | AUDIT | Ensure that role-based access control is enabled and configured appropriately | Report"
      ansible.builtin.debug:
        msg: >-
          3.2 REVIEW: authorization {{ discovered_mongod_conf['security']['authorization'] | default('disabled') }}.
          Check each account has only the roles it needs: {{ discovered_3_2_users['transformed_output'] | to_json }}

    # CIS remediation: assign the appropriate users to each role. Creates the listed accounts that don't exist yet.
    - name: "3.2 | PATCH | Ensure that role-based access control is enabled and configured appropriately | Create the listed accounts"
      when: >-
        discovered_3_2_users['transformed_output'] | selectattr('user', 'equalto', item['user'])
        | selectattr('db', 'equalto', item['db']) | list | length == 0
      community.mongodb.mongodb_shell:
        eval: >-
          db.getSiblingDB({{ item['db'] | to_json }}).createUser({user: {{ item['user'] | to_json }},
          pwd: {{ item['password'] | to_json }}, roles: {{ item['roles'] | to_json }}})
      loop: "{{ mongodb8_cis_users }}"
      loop_control:
        label: "{{ item['user'] }}@{{ item['db'] }}"
      no_log: true
      tags:
        - patch

    # Existing accounts get the listed roles they are missing. Nothing is removed and passwords are not changed.
    - name: "3.2 | PATCH | Ensure that role-based access control is enabled and configured appropriately | Add missing roles"
      vars:
        mongodb8_cis_3_2_current_roles: >-
          {{ discovered_3_2_users['transformed_output'] | selectattr('user', 'equalto', item['user'])
             | selectattr('db', 'equalto', item['db']) | map(attribute='roles') | first | default(none) }}
      when:
        - mongodb8_cis_3_2_current_roles is not none
        - item['roles'] | difference(mongodb8_cis_3_2_current_roles) | length > 0
      community.mongodb.mongodb_shell:
        eval: >-
          db.getSiblingDB({{ item['db'] | to_json }}).grantRolesToUser({{ item['user'] | to_json }},
          {{ item['roles'] | difference(mongodb8_cis_3_2_current_roles) | to_json }})
      loop: "{{ mongodb8_cis_users }}"
      loop_control:
        label: "{{ item['user'] }}@{{ item['db'] }}"
      tags:
        - patch
```

### `tasks/section_3/cis_3.3.yml`

3.3 (L1, Manual): report the account mongod runs as.

- Reads systemd's `User=` (prelim). Empty means root, so FAIL. The RPM sets `mongod`, so a fresh install passes.

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
    # No User= in the unit means systemd runs the service as root.
    mongodb8_cis_3_3_user: "{{ discovered_mongod_service['status']['User'] | default('root', true) }}"
  ansible.builtin.debug:
    msg: >-
      3.3 {{ 'PASS' if mongodb8_cis_3_3_user != 'root' else 'FAIL' }}: the mongod service runs as {{ mongodb8_cis_3_3_user }}
      (systemd User=). File permissions for this account are checked by 7.1 and 7.2.
```

### `tasks/section_3/cis_3.4.yml`

3.4 (L1, Manual): report custom roles; optional drop list.

- Built-in roles are fixed by MongoDB; only custom roles (`admin.system.roles`) are listed.

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
    - name: "3.4 | AUDIT | Ensure that each role for each MongoDB database is needed and grants only the necessary privileges | List custom roles"
      community.mongodb.mongodb_shell:
        eval: "db.getSiblingDB('admin').system.roles.find({}, {_id: 1, role: 1, db: 1, privileges: 1, roles: 1}).toArray()"
        transform: json
      register: discovered_3_4_roles
      changed_when: false

    - name: "3.4 | AUDIT | Ensure that each role for each MongoDB database is needed and grants only the necessary privileges | Report"
      ansible.builtin.debug:
        msg: >-
          3.4 REVIEW: {{ discovered_3_4_roles['transformed_output'] | length }} custom role(s) (built-in roles are fixed by MongoDB).
          Check each is needed: {{ discovered_3_4_roles['transformed_output'] | to_json }}

    # CIS: eliminate unneeded roles. Only the custom roles the site lists; users that held the role lose it.
    - name: "3.4 | PATCH | Ensure that each role for each MongoDB database is needed and grants only the necessary privileges | Drop the listed roles"
      when: item['_id'] in mongodb8_cis_drop_custom_roles
      community.mongodb.mongodb_shell:
        eval: "db.getSiblingDB({{ item['db'] | to_json }}).dropRole({{ item['role'] | to_json }})"
      loop: "{{ discovered_3_4_roles['transformed_output'] }}"
      loop_control:
        label: "{{ item['_id'] }}"
      tags:
        - patch
```

### `tasks/section_3/cis_3.5.yml`

3.5 (L2, Manual): report superuser/admin accounts; optional revoke list.

- The assert stops the run if the role's own admin is listed: the role logs in with it.

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
    mongodb8_cis_3_5_roles: [root, dbOwner, userAdmin, userAdminAnyDatabase, readWriteAnyDatabase, dbAdminAnyDatabase, clusterAdmin, hostManager]
  block:
    - name: "3.5 | AUDIT | Review Superuser/Admin Roles | Find accounts with superuser or admin roles"
      community.mongodb.mongodb_shell:
        eval: >-
          db.getSiblingDB('admin').system.users.find({'roles.role': {$in: {{ mongodb8_cis_3_5_roles | to_json }}}},
          {_id: 1, user: 1, db: 1, roles: 1}).toArray()
        transform: json
      register: discovered_3_5_users
      changed_when: false

    - name: "3.5 | AUDIT | Review Superuser/Admin Roles | Report"
      ansible.builtin.debug:
        msg: >-
          3.5 REVIEW: accounts holding {{ mongodb8_cis_3_5_roles | join(', ') }}:
          {{ discovered_3_5_users['transformed_output'] | map(attribute='_id') | join(', ') or 'none' }}. Check each one is needed.

    # The role logs in as its own admin; revoking its roles would lock the role out.
    - name: "3.5 | PATCH | Review Superuser/Admin Roles | Check the role's own admin is not listed"
      ansible.builtin.assert:
        that:
          - ('admin.' ~ mongodb8_cis_admin_user) not in mongodb8_cis_revoke_superuser_roles
        fail_msg: "3.5: remove admin.{{ mongodb8_cis_admin_user }} (the role's own admin) from mongodb8_cis_revoke_superuser_roles."
      tags:
        - patch

    # CIS: superuser/admin roles only where needed. Only from the accounts the site lists; the account stays.
    - name: "3.5 | PATCH | Review Superuser/Admin Roles | Revoke the roles from the listed accounts"
      when: item['_id'] in mongodb8_cis_revoke_superuser_roles
      community.mongodb.mongodb_shell:
        eval: >-
          db.getSiblingDB({{ item['db'] | to_json }}).revokeRolesFromUser({{ item['user'] | to_json }},
          {{ item['roles'] | selectattr('role', 'in', mongodb8_cis_3_5_roles) | list | to_json }})
      loop: "{{ discovered_3_5_users['transformed_output'] }}"
      loop_control:
        label: "{{ item['_id'] }}"
      tags:
        - patch
```

### `tasks/section_4/main.yml`
Imports section 4. 4.3 first: the other TLS rules need TLS on.

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

### `tasks/section_4/cis_4.3.yml`

4.3 (L1, Automated): require TLS.

- Two files, as mongod expects: the server certificate **with** its private key in one PEM (`certificateKeyFile`), and the CA certificate (`CAFile`).
- The site gives the files with the two `_src` variables; the role copies them to `/etc/pki/mongodb/server.pem` and `ca.pem` (fixed in `vars/main.yml`), owner mongod, `0600`.
- A renewed certificate is just a new source file: the copy changes and mongod restarts.
- The third file is optional: the role's own client certificate, for companies whose server certificates are not allowed for client use (mongod asks every client for a certificate signed by the CA).
- On by default. `_src` not set: `NOT APPLIED`, nothing changes.

```yaml
---
- name: "4.3 | PATCH | Ensure Encryption of Data in Transit TLS or SSL (Transport Encryption)"
  when:
    - mongodb8_cis_rule_4_3
    - mongodb8_cis_level_1
  tags:
    - level1
    - automated
    - patch
    - rule_4.3
    - tls
  vars:
    # Files the site gives on the control node (src) and where they go on the server (dest).
    mongodb8_cis_4_3_files:
      - {src: "{{ mongodb8_cis_tls_certificate_key_src }}", dest: "{{ mongodb8_cis_tls_certificate_key_file }}", owner: "{{ mongodb8_cis_service_user }}"}
      - {src: "{{ mongodb8_cis_tls_ca_src }}", dest: "{{ mongodb8_cis_tls_ca_file }}", owner: "{{ mongodb8_cis_service_user }}"}
      - {src: "{{ mongodb8_cis_shell_tls_certificate_key_src }}", dest: "{{ mongodb8_cis_shell_client_cert_path }}", owner: root}
  block:
    - name: "4.3 | AUDIT | Ensure Encryption of Data in Transit TLS or SSL (Transport Encryption) | Report when there are no certificate files"
      when: not mongodb8_cis_tls_ready
      ansible.builtin.debug:
        msg: >-
          4.3 NOT APPLIED: set mongodb8_cis_tls_certificate_key_src (server certificate + key) and
          mongodb8_cis_tls_ca_src (CA certificate).

    - name: "4.3 | PATCH | Ensure Encryption of Data in Transit TLS or SSL (Transport Encryption) | Create the certificate folders"
      when: mongodb8_cis_tls_ready and item['src'] | length > 0
      ansible.builtin.file:
        path: "{{ item['dest'] | dirname }}"
        state: directory
        owner: root
        group: root
        mode: "0755"
      loop: "{{ mongodb8_cis_4_3_files }}"
      loop_control:
        label: "{{ item['dest'] | dirname }}"

    # mongod reads the server and CA files as its own user; a renewed certificate is copied and mongod restarted.
    - name: "4.3 | PATCH | Ensure Encryption of Data in Transit TLS or SSL (Transport Encryption) | Copy the certificate files"
      when: mongodb8_cis_tls_ready and item['src'] | length > 0
      ansible.builtin.copy:
        src: "{{ item['src'] }}"
        dest: "{{ item['dest'] }}"
        owner: "{{ item['owner'] }}"
        group: "{{ item['owner'] }}"
        mode: "0600"
      loop: "{{ mongodb8_cis_4_3_files }}"
      loop_control:
        label: "{{ item['dest'] }}"
      notify: Restart mongod

    - name: "4.3 | PATCH | Ensure Encryption of Data in Transit TLS or SSL (Transport Encryption) | Set net.tls mode requireTLS"
      when: mongodb8_cis_tls_ready
      ansible.builtin.set_fact:
        mongodb8_cis_conf: >-
          {{ mongodb8_cis_conf | combine({'net': {'tls': {'mode': 'requireTLS',
             'certificateKeyFile': mongodb8_cis_tls_certificate_key_file, 'CAFile': mongodb8_cis_tls_ca_file}}}, recursive=true) }}
```

### `tasks/section_4/cis_4.1.yml`

4.1 (L2, Automated): no TLS 1.0 / 1.1.

- `mongodb8_cis_tls_enabled` in `when`: mongod refuses TLS options without TLS.

```yaml
---
# mongod refuses net.tls options unless TLS is on (rule 4.3).
- name: "4.1 | PATCH | Ensure legacy TLS protocols are disabled"
  when:
    - mongodb8_cis_rule_4_1
    - mongodb8_cis_level_2
    - mongodb8_cis_tls_enabled
  tags:
    - level2
    - automated
    - patch
    - rule_4.1
    - tls
  ansible.builtin.set_fact:
    mongodb8_cis_conf: "{{ mongodb8_cis_conf | combine({'net': {'tls': {'disabledProtocols': 'TLS1_0,TLS1_1'}}}, recursive=true) }}"
```

### `tasks/section_4/cis_4.2.yml`

4.2 (L1, Automated): weak protocols off. Same setting as 4.1 (CIS lists it twice, at L1 and L2).

```yaml
---
# mongod refuses net.tls options unless TLS is on (rule 4.3).
- name: "4.2 | PATCH | Ensure Weak Protocols are Disabled"
  when:
    - mongodb8_cis_rule_4_2
    - mongodb8_cis_level_1
    - mongodb8_cis_tls_enabled
  tags:
    - level1
    - automated
    - patch
    - rule_4.2
    - tls
  ansible.builtin.set_fact:
    mongodb8_cis_conf: "{{ mongodb8_cis_conf | combine({'net': {'tls': {'disabledProtocols': 'TLS1_0,TLS1_1'}}}, recursive=true) }}"
```

### `tasks/section_4/cis_4.4.yml`

4.4 (L2, Automated): FIPS mode.
- On by default, but only applied when TLS is on and the OS runs in FIPS mode (`ansible_facts['fips']`); otherwise `NOT APPLIED` with the reason. mongod does not start in FIPS mode on a non-FIPS OS.

```yaml
---
- name: "4.4 | PATCH | Ensure Federal Information Processing Standard (FIPS) is enabled"
  when:
    - mongodb8_cis_rule_4_4
    - mongodb8_cis_level_2
  tags:
    - level2
    - automated
    - patch
    - rule_4.4
    - tls
  block:
    # mongod refuses FIPSMode without TLS (4.3), and does not start unless the OS itself runs in FIPS mode.
    - name: "4.4 | AUDIT | Ensure Federal Information Processing Standard (FIPS) is enabled | Report when not possible"
      when: not mongodb8_cis_tls_enabled or not ansible_facts['fips'] | default(false)
      ansible.builtin.debug:
        msg: >-
          4.4 NOT APPLIED: needs TLS (4.3: {{ mongodb8_cis_tls_enabled }}) and the OS in FIPS mode
          ({{ ansible_facts['fips'] | default(false) }}; RHEL: fips-mode-setup --enable, then reboot).

    - name: "4.4 | PATCH | Ensure Federal Information Processing Standard (FIPS) is enabled | Set net.tls.FIPSMode: true"
      when: mongodb8_cis_tls_enabled and ansible_facts['fips'] | default(false)
      ansible.builtin.set_fact:
        mongodb8_cis_conf: "{{ mongodb8_cis_conf | combine({'net': {'tls': {'FIPSMode': true}}}, recursive=true) }}"
```

### `tasks/section_4/cis_4.5.yml`
4.5 (L2, Manual): report encryption at rest only.

- MongoDB cannot encrypt existing data, and the key (KMIP or keyfile) is the organization's design, so the organization
  enables it by hand (docs/manual-remediation.md 4.5). The role reports whether it is on and which key management is used.

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
      security.enableEncryption = {{ discovered_mongod_conf['security']['enableEncryption'] | default(false) }},
      key: {{ discovered_mongod_conf['security']['kmip'] | default(discovered_mongod_conf['security']['encryptionKeyFile'] | default('none')) }}.
      CIS recommends KMIP. Enabling it is done by hand: docs/manual-remediation.md 4.5.
```

### `tasks/section_5/main.yml`

Imports section 5.

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

### `tasks/section_5/cis_5.1.yml`

5.1 (L1, Automated): turn on auditing.

- Only when `auditLog` is missing: an existing one is kept as the site set it.
- `mongodb8_cis_audit_log` is written as-is (default `destination: syslog`).

```yaml
---
- name: "5.1 | PATCH | Ensure that system activity is audited"
  when:
    - mongodb8_cis_rule_5_1
    - mongodb8_cis_level_1
    - mongodb8_cis_conf['auditLog'] is not defined   # an existing auditLog is kept as it is
  tags:
    - level1
    - automated
    - patch
    - rule_5.1
    - logging
  ansible.builtin.set_fact:
    mongodb8_cis_conf: "{{ mongodb8_cis_conf | combine({'auditLog': mongodb8_cis_audit_log}) }}"
```

### `tasks/section_5/cis_5.2.yml`

5.2 (L2, Manual): report the audit filter; optional `mongodb8_cis_audit_filter`.

- Only when auditing is on (5.1).

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
          5.2 REVIEW: auditLog.filter = {{ discovered_mongod_conf['auditLog']['filter'] | default('not set (every event is audited)') }}.
          Check it against the organization's audit requirements.

    - name: "5.2 | PATCH | Ensure that audit filters are configured properly | Set auditLog.filter"
      when:
        - mongodb8_cis_audit_filter | length > 0
        - mongodb8_cis_conf['auditLog'] is defined   # auditing must be on (5.1)
      ansible.builtin.set_fact:
        mongodb8_cis_conf: "{{ mongodb8_cis_conf | combine({'auditLog': {'filter': mongodb8_cis_audit_filter}}, recursive=true) }}"
      tags:
        - patch
```

### `tasks/section_5/cis_5.3.yml`

5.3 (L2, Automated): `systemLog.quiet: false`.

- Written explicitly, because CIS's audit greps for it.

```yaml
---
- name: "5.3 | PATCH | Ensure that logging captures as much information as possible"
  when:
    - mongodb8_cis_rule_5_3
    - mongodb8_cis_level_2
  tags:
    - level2
    - automated
    - patch
    - rule_5.3
    - logging
  ansible.builtin.set_fact:
    mongodb8_cis_conf: "{{ mongodb8_cis_conf | combine({'systemLog': {'quiet': false}}, recursive=true) }}"
```

### `tasks/section_5/cis_5.4.yml`

5.4 (L2, Automated): `systemLog.logAppend: true`.

```yaml
---
- name: "5.4 | PATCH | Ensure that new entries are appended to the end of the log file"
  when:
    - mongodb8_cis_rule_5_4
    - mongodb8_cis_level_2
  tags:
    - level2
    - automated
    - patch
    - rule_5.4
    - logging
  ansible.builtin.set_fact:
    mongodb8_cis_conf: "{{ mongodb8_cis_conf | combine({'systemLog': {'logAppend': true}}, recursive=true) }}"
```

### `tasks/section_6/main.yml`

Imports section 6.

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

### `tasks/section_6/cis_6.1.yml`

6.1 (L1, Automated): non-default port.

- With SELinux on, mongod may only listen on ports labelled `mongod_port_t`, so the port is labelled first.
- On by default. No port set: `NOT APPLIED` (CIS leaves the port to the organization).

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
    - name: "6.1 | AUDIT | Ensure that MongoDB uses a non-default port | Report when no port is set"
      when: mongodb8_cis_port | string | length == 0
      ansible.builtin.debug:
        msg: "6.1 NOT APPLIED: set mongodb8_cis_port (1024-65535, not 27017; every client must use the new port)."

    - name: "6.1 | PATCH | Ensure that MongoDB uses a non-default port | Apply the port"
      when: mongodb8_cis_port | string | length > 0
      block:
        - name: "6.1 | PATCH | Ensure that MongoDB uses a non-default port | Check the port value"
          ansible.builtin.assert:
            that:
              - mongodb8_cis_port | int >= 1024
              - mongodb8_cis_port | int <= 65535
              - mongodb8_cis_port | int != 27017
            fail_msg: "6.1: set mongodb8_cis_port to 1024-65535, not 27017 (found '{{ mongodb8_cis_port }}')."

        # SELinux confines mongod (mongod_t): it can only listen on ports labelled mongod_port_t.
        - name: "6.1 | PATCH | Ensure that MongoDB uses a non-default port | Label the port for SELinux"
          when:
            - ansible_facts['selinux']['status'] | default('disabled') == 'enabled'
            - mongodb8_cis_port | int not in mongodb8_cis_selinux_default_ports
          block:
            - name: "6.1 | PATCH | Ensure that MongoDB uses a non-default port | Install the SELinux tools"
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
          ansible.builtin.set_fact:
            mongodb8_cis_conf: "{{ mongodb8_cis_conf | combine({'net': {'port': mongodb8_cis_port | int}}, recursive=true) }}"
```

### `tasks/section_6/cis_6.2.yml`

6.2 (L2, Manual): report systemd limits; optional drop-in.

- A drop-in (`mongod.service.d/`) leaves the RPM's unit file untouched. `copy` is declarative: no change, no restart.

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
  block:
    - name: "6.2 | AUDIT | Ensure that operating system resource limits are set for MongoDB | Report each limit"
      ansible.builtin.debug:
        msg: >-
          6.2 {{ 'PASS' if discovered_mongod_service['status'][item.key] | default('') == item.value | string else 'REVIEW' }}:
          {{ item.key }} = {{ discovered_mongod_service['status'][item.key] | default('unknown') }} (CIS: {{ item.value }}).
      loop: "{{ mongodb8_cis_resource_limits | dict2items }}"
      loop_control:
        label: "{{ item.key }}"

    # CIS remediation: set the limits, then restart mongod. Written as a systemd drop-in; the RPM's unit stays untouched.
    - name: "6.2 | PATCH | Ensure that operating system resource limits are set for MongoDB | Create the drop-in folder"
      when: mongodb8_cis_fix_resource_limits
      ansible.builtin.file:
        path: "{{ mongodb8_cis_limits_dropin | dirname }}"
        state: directory
        owner: root
        group: root
        mode: "0755"
      tags:
        - patch

    - name: "6.2 | PATCH | Ensure that operating system resource limits are set for MongoDB | Write the limits"
      when: mongodb8_cis_fix_resource_limits
      ansible.builtin.copy:
        dest: "{{ mongodb8_cis_limits_dropin }}"
        content: |
          [Service]
          {% for name, value in mongodb8_cis_resource_limits.items() %}
          {{ name }}={{ value }}
          {% endfor %}
        owner: root
        group: root
        mode: "0644"
      notify: Restart mongod
      tags:
        - patch
```

### `tasks/section_6/cis_6.3.yml`

6.3 (L2, Manual): report server-side JavaScript; `mongodb8_cis_javascript_needed: false` turns it off.

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
  block:
    - name: "6.3 | AUDIT | Ensure that server-side scripting is disabled if not needed | Report"
      ansible.builtin.debug:
        msg: >-
          6.3 REVIEW: security.javascriptEnabled = {{ discovered_mongod_conf['security']['javascriptEnabled'] | default('not set (true)') }};
          mongodb8_cis_javascript_needed = {{ mongodb8_cis_javascript_needed }}.

    - name: "6.3 | PATCH | Ensure that server-side scripting is disabled if not needed | Set security.javascriptEnabled: false"
      when: not mongodb8_cis_javascript_needed
      ansible.builtin.set_fact:
        mongodb8_cis_conf: "{{ mongodb8_cis_conf | combine({'security': {'javascriptEnabled': false}}, recursive=true) }}"
      tags:
        - patch
```

### `tasks/section_7/main.yml`

Imports section 7.

```yaml
---
- name: "SECTION | 7.1 | Ensure appropriate key file permissions are set"
  ansible.builtin.import_tasks:
    file: cis_7.1.yml

- name: "SECTION | 7.2 | Ensure appropriate database file permissions are set."
  ansible.builtin.import_tasks:
    file: cis_7.2.yml
```

### `tasks/section_7/cis_7.1.yml`

7.1 (L1, Manual): report key/certificate file permissions; optional fix to 0600 owner mongod.

- Reads the files from `mongodb8_cis_conf`, so files set by 4.3 in this run are included.

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
    # Key and certificate files mongod.conf points to (after this run's changes).
    mongodb8_cis_7_1_files: >-
      {{ [mongodb8_cis_conf['security']['keyFile'] | default(''),
          mongodb8_cis_conf['net']['tls']['certificateKeyFile'] | default(''),
          mongodb8_cis_conf['net']['tls']['CAFile'] | default('')] | select | list }}
  block:
    - name: "7.1 | AUDIT | Ensure appropriate key file permissions are set | Get the files"
      ansible.builtin.stat:
        path: "{{ item }}"
      loop: "{{ mongodb8_cis_7_1_files }}"
      register: discovered_7_1_files

    - name: "7.1 | AUDIT | Ensure appropriate key file permissions are set | Report when there are none"
      when: mongodb8_cis_7_1_files | length == 0
      ansible.builtin.debug:
        msg: "7.1 NOT APPLICABLE: mongod.conf has no keyFile, certificateKeyFile or CAFile."

    - name: "7.1 | AUDIT | Ensure appropriate key file permissions are set | Report each file"
      ansible.builtin.debug:
        msg: >-
          7.1 {{ 'PASS' if item['stat']['mode'] | default('') in ['0600', '0400'] and item['stat']['pw_name'] | default('') == mongodb8_cis_service_user
          else 'FAIL' }}: {{ item['item'] }} is {{ item['stat']['mode'] | default('missing') }} {{ item['stat']['pw_name'] | default('') }}
          (CIS: 0600, owner {{ mongodb8_cis_service_user }}).
      loop: "{{ discovered_7_1_files['results'] }}"
      loop_control:
        label: "{{ item['item'] }}"

    # CIS remediation: chmod 600, owned by the mongod user.
    - name: "7.1 | PATCH | Ensure appropriate key file permissions are set | Set 0600 and owner"
      when:
        - mongodb8_cis_fix_key_file_permissions
        - item['stat']['exists']
      ansible.builtin.file:
        path: "{{ item['item'] }}"
        owner: "{{ mongodb8_cis_service_user }}"
        group: "{{ mongodb8_cis_service_user }}"
        mode: "0600"
      loop: "{{ discovered_7_1_files['results'] }}"
      loop_control:
        label: "{{ item['item'] }}"
      tags:
        - patch
```

### `tasks/section_7/cis_7.2.yml`

7.2 (L1, Manual): report dbPath permissions; optional fix to 0770 owner mongod.

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
    mongodb8_cis_7_2_db_path: "{{ mongodb8_cis_conf['storage']['dbPath'] | default(mongodb8_cis_default_db_path) }}"
  block:
    - name: "7.2 | AUDIT | Ensure appropriate database file permissions are set. | Get dbPath"
      ansible.builtin.stat:
        path: "{{ mongodb8_cis_7_2_db_path }}"
      register: discovered_7_2_db_path

    - name: "7.2 | AUDIT | Ensure appropriate database file permissions are set. | Report"
      ansible.builtin.debug:
        msg: >-
          7.2 {{ 'PASS' if discovered_7_2_db_path['stat']['mode'] == '0770' and discovered_7_2_db_path['stat']['pw_name'] == mongodb8_cis_service_user
          else 'FAIL' }}: {{ mongodb8_cis_7_2_db_path }} is {{ discovered_7_2_db_path['stat']['mode'] }}
          {{ discovered_7_2_db_path['stat']['pw_name'] }} (CIS: 0770, owner {{ mongodb8_cis_service_user }}).

    # CIS remediation: chmod 770, owned by the mongod user.
    - name: "7.2 | PATCH | Ensure appropriate database file permissions are set. | Set 0770 and owner"
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

### `tasks/selinux.yml`

Optional extra, not CIS (`mongodb8_cis_selinux_policy`). Unchanged from before.

- RHEL 9/10 build MongoDB's own module; RHEL 8's base policy already confines mongod.
- Labels a non-default dbPath, log folder and port.

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
    mongodb8_cis_selinux_db_path: "{{ mongodb8_cis_conf['storage']['dbPath'] | default(mongodb8_cis_default_db_path) }}"
    mongodb8_cis_selinux_log_dir: "{{ mongodb8_cis_conf['systemLog']['path'] | default(mongodb8_cis_default_log_path) | dirname }}"
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
                paths: "{{ mongodb8_cis_conf['net']['unixDomainSocket']['pathPrefix'] | default('/tmp') }}"
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
