# Control node setup — `mongodb8_cis`

One Ansible environment that runs this role against RHEL 8, 9 and 10 (D15).

## Why ansible-core 2.16

| | ansible-core 2.16 | 2.17 and newer |
|---|---|---|
| Target Python | 2.7, **3.6**–3.12 | 3.7+ (no Python 3.6) |
| RHEL 8 (system Python 3.6) | ✅ | ❌ (`dnf` bindings exist only for 3.6) |
| RHEL 9 (3.9), RHEL 10 (3.12) | ✅ | ✅ |
| Controller Python | **3.10–3.12** | newer |

So 2.16 is the one version that manages all three. It reached upstream end of life in July 2025 (no more security
fixes); keep it in its own virtual environment, used only for this work.

## 1. Python 3.10–3.12 on the control node

| Control node OS | Command |
|-----------------|---------|
| Fedora (system Python is newer) | `sudo dnf install python3.11` |
| RHEL / Rocky / Alma 9 | `sudo dnf install python3.11` (or `python3.12`) |
| RHEL / Rocky / Alma 10 | system `python3` is 3.12, nothing to install |

Check: `python3.11 --version` (or `python3.12 --version`).

## 2. ansible-core 2.16 in a virtual environment

```bash
python3.11 -m venv ~/.ansible-2.16.1-env               # use python3.12 if that is the one you have
~/.ansible-2.16.1-env/bin/pip install --upgrade pip
~/.ansible-2.16.1-env/bin/pip install 'ansible-core>=2.16.1,<2.17'

source ~/.ansible-2.16.1-env/bin/activate              # every new shell
ansible --version                                      # ansible [core 2.16.x], python version = 3.11.x
```

## 3. Collections the role needs

```bash
git clone https://github.com/bunnywkwk/mongodb-cis.git ~/mongodb-cis
ansible-galaxy collection install -r ~/mongodb-cis/requirements.yml
ansible-galaxy collection list | grep -E 'community\.(mongodb|general)'
# community.general 11.x   (12.x needs ansible-core 2.17+)
# community.mongodb 1.8.x
```

If `~/.ansible/collections` already holds a newer `community.general` (12.x or later) for another Ansible version,
install into the project instead, `-p ./collections`, and set `collections_path = ./collections` in its `ansible.cfg`.

## 4. Use the role from a project

```bash
mkdir -p ~/mongodb8-cis-test/roles
ln -s ~/mongodb-cis ~/mongodb8-cis-test/roles/mongodb8_cis    # the role is named after the folder it sits in
```

`ansible.cfg` in the project (roles are looked up next to the playbook unless `roles_path` says otherwise):

```ini
[defaults]
inventory = sysconfig/inventory.yml
roles_path = ./roles
```

Check before the first run:

```bash
ansible-playbook playbooks/site.yml --syntax-check     # playbook: playbooks/site.yml
ansible all -m ping                                    # pong from every target (Python found)
```

## 5. Targets

- SSH key login for a user with sudo (`ssh-copy-id <user>@<host>`); run with `-K` if sudo asks for a password.
- Registered to Red Hat (BaseOS/AppStream reachable) and internet access to `repo.mongodb.com` for the install.
- VMs: CPU type must expose **x86-64-v3 and AVX** (Proxmox: CPU type `host`; libvirt: `host-passthrough`). RHEL 10 does
  not boot without x86-64-v3, and MongoDB 8 does not start without AVX.
