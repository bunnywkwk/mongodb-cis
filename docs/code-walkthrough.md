# Code walkthrough — `mongodb8_cis` (rebuild, branch `mongodb8-cis-rebuild`)

What each file and block does, why it's there, and where that comes from. Written as the role is rebuilt by hand,
batch by batch. Each section ends with **Sources**: official documentation that backs the choice.

| # | File | Status |
|---|------|--------|
| 1 | `.yamllint`, `.ansible-lint` | done |
| 2 | `tasks/main.yml` (prelim only) | done |
| 3 | `tasks/prelim.yml`: version and OS checks | done |
| 4 | `tasks/prelim.yml`: detect MongoDB, read `mongod.conf`, read the service | next |

---

## 1. Linting: `.yamllint` and `.ansible-lint`

Two tools check the code **before** it runs on a server. Run both inside the role folder after every batch:

```bash
cd ~/ansible-cis/mongodb_cis
yamllint . && ansible-lint --offline
```

| Tool | Checks | Example of what it catches |
|------|--------|----------------------------|
| **yamllint** | The **YAML file format**: indentation, blank lines, spacing, `true`/`false` style | Two blank lines in a row, `yes` instead of `true`, a tab instead of spaces |
| **ansible-lint** | **Ansible best practices**: module names, task names, risky patterns, deprecated features. It also runs yamllint's rules | A task without a name, `command:` without `changed_when`, `shell` where a module exists, missing FQCN (`copy` instead of `ansible.builtin.copy`) |

Neither tool runs anything on a host. They can't catch a **misspelled variable name**: that only fails when the task runs (see section 3).

### `.yamllint`

| Setting | Does | Why |
|---------|------|-----|
| `extends: default` | Start from yamllint's standard rules | Change only what we must |
| `ignore: .ansible/, molecule/` | Skip ansible-lint's cache and Molecule files | Not our role code |
| `braces` / `brackets` `max-spaces-inside: 1` | Allows `{{ var }}` and `[ a ]` | **Required by ansible-lint**: it refuses a yamllint config without it |
| `comments: min-spaces-from-content: 1` | `key: value # note` with one space is fine | Lockdown's setting |
| `comments-indentation: disable` | Comments may be indented differently from code | Lockdown's setting |
| `empty-lines: max: 1` | At most one blank line in a row | Compact files |
| `indentation: spaces: 2, indent-sequences: consistent` | 2-space indent, list style the same within a file | Project style; Lockdown's setting |
| `line-length: disable` | No 80-character limit | CIS titles in task names are long; Lockdown disables it too |
| `octal-values: forbid-implicit/explicit-octal` | Rejects unquoted `mode: 0644` | Unquoted, YAML can read it as a number and set **wrong file permissions**. Always write `mode: "0644"` |
| `truthy: allowed-values: ["true", "false"]` | Only `true`/`false`, not `yes`/`on` | Avoids YAML 1.1 surprises; the role's switches must be real booleans (D19) |

### `.ansible-lint`

| Setting | Does | Why |
|---------|------|-----|
| `profile: production` | The strictest of ansible-lint's rule sets (`min` → `basic` → `moderate` → `safety` → `shared` → `production`) | The role must be production-grade. Any lower profile would accept weaker code |
| `exclude_paths: .ansible/, docs/` | Don't lint the cache or the Markdown docs | Not role code |
| `--offline` (command line) | Don't try to download the collections from `requirements.yml` | They're already installed; avoids network errors |

**Sources**
- yamllint configuration and rules: https://yamllint.readthedocs.io/en/stable/configuration.html, https://yamllint.readthedocs.io/en/stable/rules.html
- ansible-lint profiles: https://ansible.readthedocs.io/projects/lint/profiles/
- ansible-lint configuration (`exclude_paths`, `offline`): https://ansible.readthedocs.io/projects/lint/configuring/
- ansible-lint's yamllint requirements (`braces`, `octal-values`, `truthy`): https://ansible.readthedocs.io/projects/lint/rules/yaml/
- Lockdown's own `.yamllint` (same settings): https://github.com/ansible-lockdown/RHEL9-CIS/blob/devel/.yamllint

---

## 2. `tasks/main.yml`: the table of contents

```yaml
- name: Run preliminary checks and discovery
  ansible.builtin.import_tasks:
    file: prelim.yml
  tags: always
```

| Key | Does | Why |
|-----|------|-----|
| `ansible.builtin.import_tasks` | Inserts `prelim.yml`'s tasks here when the playbook is **read** (static) | Static import lets `tags` and `when` apply to every task inside, and `--list-tasks` shows them. Lockdown imports its sections the same way |
| `tags: always` | Runs prelim even when you choose tasks with `--tags` (e.g. `--tags rule_5.4`) | Every rule needs what prelim reads; without it, a tagged run would fail. `always` is a special Ansible tag for exactly this |

Each CIS section is added below this one when it's built.

**Sources**
- `import_tasks`: https://docs.ansible.com/ansible/latest/collections/ansible/builtin/import_tasks_module.html
- Import vs include (static vs dynamic): https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse.html#comparing-includes-and-imports-dynamic-and-static-re-use
- Special tags `always` / `never`: https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_tags.html#special-tags-always-and-never

---

## 3. `tasks/prelim.yml`: checks before anything else

Every task here is named `PRELIM | AUDIT | …`: it only reads or checks, and **never changes the host**.

| Task | Does | Why |
|------|------|-----|
| Gather minimal facts if the play skipped them | `ansible.builtin.setup` with `gather_subset: min`, only when `ansible_facts['os_family']` is missing | The checks below need the OS facts. This keeps the role working when a play sets `gather_facts: false`; `min` collects only the basics, so it's fast |
| Check the ansible-core version | `assert` that `ansible_version.full is version('2.16.1', '>=')` | ansible-core 2.16 is the only version that manages RHEL 8 (Python 3.6) **and** RHEL 9/10. 2.16.1 is Lockdown's floor (D15) |
| Check supported OS | `assert` RedHat family, major version in `mongodb8_cis_supported_os_majors` (8/9/10), architecture `x86_64` | Fail fast with a clear message instead of failing halfway on an unsupported system |
| Check the requested MongoDB version | `assert` that `mongodb8_cis_version` is in `mongodb8_cis_supported_versions` (`8.0`) | The role implements the CIS MongoDB **8** Benchmark; another version needs another benchmark |

How `assert` works: `that:` lists conditions that must all be **true** to continue. If one is false, the run stops for that host and prints `fail_msg`.

`quiet` was left out on purpose: it only shortens the output of a **passing** assert. It doesn't hide any values, so it's purely cosmetic.

**Sources**
- `setup` module, `gather_subset`: https://docs.ansible.com/ansible/latest/collections/ansible/builtin/setup_module.html
- `assert` module (`that`, `fail_msg`, `quiet`): https://docs.ansible.com/ansible/latest/collections/ansible/builtin/assert_module.html
- Version comparison test `version(…, '>=')`: https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_tests.html#comparing-versions
- `ansible_version` magic variable: https://docs.ansible.com/ansible/latest/reference_appendices/special_variables.html
- ansible-core target Python per version (why 2.16): https://docs.ansible.com/ansible/latest/reference_appendices/release_and_maintenance.html
- CIS MongoDB 8 Benchmark v2.0.0, *Target Technology Details*: "MongoDB version/s 8.x"
