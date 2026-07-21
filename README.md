# alloy-ansible

Install and configure [**Grafana Alloy**](https://grafana.com/docs/alloy/latest/)
across a fleet of Linux hosts with Ansible, wiring each collector to
[**Grafana Cloud Fleet Management**](https://grafana.com/docs/grafana-cloud/send-data/fleet-management/)
through a `remotecfg` block so its pipeline is managed centrally.

The heavy lifting is done by the official
[`grafana.grafana.alloy`](https://github.com/grafana/grafana-ansible-collection)
role — this repo wraps it with a batteries-included playbook: credentials in
Ansible Vault, a distro-aware Python interpreter, low-memory handling, and the
`remotecfg` + remote_write environment wiring.

---

## Features

- **`remotecfg`-first.** Hosts register with Fleet Management on boot and pull
  their full pipeline centrally — the on-host config stays a few lines.
- **RHEL 8/9 and Ubuntu (incl. 18.04).** One playbook, one run, mixed fleet.
- **Secrets in Ansible Vault.** No plaintext tokens in the repo.
- **Remote_write credentials via the systemd env file.** `GCLOUD_RW_API_KEY`
  is delivered through `/etc/default/alloy` (or `/etc/sysconfig/alloy`) and
  locked down to `0600`.
- **Low-memory safe.** Auto-provisions swap on small instances so the package
  install doesn't get OOM-killed.

## Supported platforms

| OS | System Python | Notes |
|----|---------------|-------|
| RHEL 8 | 3.6 | requires control-node ansible-core 2.16 |
| RHEL 9 | 3.9 | |
| Ubuntu 18.04 | 3.6 | requires control-node ansible-core 2.16 |
| Ubuntu 20.04+ | 3.8+ | |

> **Control node must run ansible-core 2.16.x** (`>=2.16,<2.17`). See
> [docs/troubleshooting.md](docs/troubleshooting.md#why-ansible-core-216) for
> the full reasoning — in short, RHEL 8 and Ubuntu 18.04 ship Python 3.6, and
> the package modules must run on the system Python where their bindings live,
> but ansible-core 2.17+ dropped Python 3.6 support.

## Repository layout

```
requirements.txt                         # control-node ansible-core pin (<2.17)
requirements.yml                         # grafana.grafana collection + deps
ansible.cfg                              # inventory, become, vault password file
inventory.ini.example                    # template; copy to inventory.ini (git-ignored)
install-alloy.yml                        # the playbook
group_vars/alloy_nodes/
  vars.yml                               # Alloy config, remotecfg block, env vars
  vault.yml                              # encrypted credentials (git-ignored)
  vault.yml.example                      # template for the vault
docs/                                    # architecture, configuration, ops, troubleshooting
```

## Quick start

```bash
# 1. Control node: pinned ansible-core + collection
python3 -m venv .venv-ansible216
.venv-ansible216/bin/pip install -r requirements.txt
.venv-ansible216/bin/ansible-galaxy collection install -r requirements.yml -p collections

# 2. Secrets
openssl rand -base64 24 > .vault_pass && chmod 600 .vault_pass
cp group_vars/alloy_nodes/vault.yml.example group_vars/alloy_nodes/vault.yml
# fill in real values, then:
.venv-ansible216/bin/ansible-vault encrypt group_vars/alloy_nodes/vault.yml

# 3. Point vars.yml at your Fleet Management stack; add hosts to your inventory
cp inventory.ini.example inventory.ini   # then edit (inventory.ini is git-ignored)

# 4. Run
.venv-ansible216/bin/ansible-playbook -i inventory.ini install-alloy.yml
```

## Documentation

- [**Architecture**](docs/architecture.md) — how remotecfg, the env file, and
  the role fit together, with the data flow.
- [**Configuration**](docs/configuration.md) — every variable, the vault, and
  the `remotecfg` block explained.
- [**Operations**](docs/operations.md) — running, adding hosts, upgrading,
  uninstalling, rotating tokens.
- [**Troubleshooting**](docs/troubleshooting.md) — the real failure modes
  (ansible-core, OOM `rc 137`, `401` on remote_write) and their fixes.

## Security

- Credentials live only in the Ansible Vault (`vault.yml`, git-ignored). The
  `.vault_pass` file and `collections/` are git-ignored too.
- The systemd env file that carries `GCLOUD_RW_API_KEY` is tightened to `0600`.
- The token used here should be treated as sensitive — rotate it if it is ever
  exposed. See [docs/operations.md](docs/operations.md#rotating-credentials).
