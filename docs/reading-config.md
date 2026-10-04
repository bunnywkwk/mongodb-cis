# Reading and changing `mongod.conf` — how the role handles config values

How the role reads `/etc/mongod.conf`, looks up values, fills in missing ones, and writes changes back. Every
example below was run with a `debug`-only playbook on localhost (ansible-core 2.16.19 and 2.20.7, 2026-10-03);
outputs are copied from those runs.

## 1. From file to dictionary (prelim)

```yaml
- name: "PRELIM | AUDIT | Read mongod.conf"
  ansible.builtin.slurp:
    src: "{{ mongodb8_cis_conf_path }}"
  register: discovered_mongod_conf_raw

- name: "PRELIM | AUDIT | Parse mongod.conf"
  ansible.builtin.set_fact:
    discovered_mongod_conf: "{{ discovered_mongod_conf_raw['content'] | b64decode | from_yaml | default({}, true) }}"
```

| Stage | What it looks like |
|-------|--------------------|
| File on the host (`cat /etc/mongod.conf`) | text: `storage:` / `  dbPath: /var/lib/mongo` / `net:` / `  port: 27017` |
| `slurp` result (`['content']`) | base64 text, e.g. `c3RvcmFnZToKICBkYlBhdGg6...` (slurp always returns base64) |
| `\| b64decode` | the same text as `cat` |
| `\| from_yaml` | a **dictionary**: `{'storage': {'dbPath': '/var/lib/mongo'}, 'net': {'port': 27017}}` |
| `\| default({}, true)` | `{}` when the file is empty (see 3.2) |

`discovered_mongod_conf` is a **snapshot** in memory. It has no live link to the file: when a task rewrites the
file, the dictionary does not change by itself (section 5).

## 2. Looking up a value: `[...][...]`

Each `['key']` goes one level deeper into the YAML, like following the indentation:

```yaml
storage:                 # discovered_mongod_conf['storage']
  dbPath: /var/lib/mongo # discovered_mongod_conf['storage']['dbPath']  -> /var/lib/mongo
net:
  tls:
    mode: requireTLS     # discovered_mongod_conf['net']['tls']['mode'] -> requireTLS
```

The role uses `['key']` (bracket) instead of `.key` (dot) everywhere: brackets work for every key name, while dot
notation can clash with Python dict methods (e.g. a key called `items` or `keys`).

## 3. Missing values: `| default(...)`

### 3.1 A key that isn't in the file

`mongod.conf` only contains what someone wrote in it. Looking up a key that isn't there **fails the task**:

```text
{{ conf['auditLog']['destination'] }}
-> FAILED: 'dict object' has no attribute 'auditLog'
```

`default` gives a fallback instead. It also works when a **parent** key is missing (`auditLog` here):

```text
{{ conf['auditLog']['destination'] | default('not set') }}
-> not set
```

Rule: **every lookup of a setting that might not be in the file ends with `| default(...)`.** Choose the fallback by
what the code does next:

| Fallback | Meaning | Example in the role |
|----------|---------|---------------------|
| `default('')` | "not set"; test it with `\| length == 0` / `> 0` | 5.1 `when`: no `auditLog.destination` yet |
| `default(<real default>)` | what MongoDB/the RPM uses when the key is absent | `['net']['port'] \| default(27017)` |
| `default(<var>)` | a role fallback from `vars/main.yml` | `['storage']['dbPath'] \| default(mongodb8_cis_default_db_path)` |
| `default('disabled')` | the value that means "off" | `['net']['tls']['mode'] \| default('disabled')` |

### 3.2 `default(x)` vs `default(x, true)`

Plain `default` only replaces **undefined**. An empty file parses to `None`, which *is* defined, so it is not
replaced:

```text
{{ ('' | from_yaml) | default({}) }}        -> (empty: None stays None)
{{ ('' | from_yaml) | default({}, true) }}  -> {}
```

The second argument `true` means "also replace empty or false values" (`None`, `''`, `[]`, `{}`). Use it when an
empty value should count as missing: prelim's parse (empty file → `{}`), and `mongodb8_cis_service_user`
(systemd returns an empty `User=` when the unit runs as root → fall back to `mongod`).

## 4. Writing a change: `combine(..., recursive=true)`

A rule never writes a whole new config. It merges **only its own keys** into what is already there:

```yaml
content: "{{ discovered_mongod_conf | combine({'auditLog': {'destination': 'syslog'}}, recursive=true) | to_nice_yaml(indent=2) }}"
```

`recursive=true` is required. Without it, `combine` replaces a whole top-level section:

| Merge `{'net': {'tls': {'mode': 'requireTLS'}}}` into `{'net': {'port': 27017}, ...}` | Result |
|---|---|
| `combine(...)` | `net: {tls: {mode: requireTLS}}` → **`port: 27017` is lost** |
| `combine(..., recursive=true)` | `net: {port: 27017, tls: {mode: requireTLS}}` ✓ |

`to_nice_yaml(indent=2)` turns the dictionary back into YAML text for `copy`. Side effect: comments and key order
of the original file are not kept (`backup: true` keeps the previous file next to it).

## 5. Keeping the snapshot in sync: `set_fact` after every write

Because the dictionary is a snapshot (section 1), each rule that writes the file updates it right after:

```yaml
- name: "5.1 | PATCH | ... | Update the parsed config"
  ansible.builtin.set_fact:
    discovered_mongod_conf: "{{ discovered_mongod_conf | combine(mongodb8_cis_5_1_settings, recursive=true) }}"
```

Without it, the next rule builds its file from the old snapshot and erases the previous rule's change:

```text
prelim         file: {storage, net}             dict: {storage, net}
5.1 copy       file: {storage, net, auditLog}   dict: {storage, net}             <- stale
5.2 copy       file: {storage, net, auditLog: {filter}}                          <- 5.1's destination lost

with set_fact:
5.1            file = dict = {storage, net, auditLog: {destination}}
5.2            file = dict = {storage, net, auditLog: {destination, filter}}     ✓
```

Re-reading the file (`slurp` + parse again) after each write would also work, but costs two tasks and a round
trip per rule; `set_fact` does the same in memory.

## 6. Two copies of the config, two jobs

| Variable | Set in | Changes during the run? | Used for |
|----------|--------|-------------------------|----------|
| `discovered_mongod_conf` | prelim, then every config rule | yes (section 5) | building the next file write, AUDIT checks of the file |
| `discovered_mongod_running_conf` | prelim only | no | what mongod is **running** with until the restart handler: the role's own DB connection (sections 2/3 host, port, TLS, auth) |

## 7. Cheat sheet

```text
read     slurp -> b64decode -> from_yaml -> default({}, true)
look up  conf['a']['b'] | default(<fallback>)
empty?   value | default('') | length == 0
write    copy content: conf | combine(new, recursive=true) | to_nice_yaml(indent=2)
sync     set_fact conf: conf | combine(new, recursive=true)
```
