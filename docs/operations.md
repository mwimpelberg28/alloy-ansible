# Operations

All commands assume the pinned venv. If `.vault_pass` exists and
`ansible.cfg` points at it, no `--ask-vault-pass` is needed.

## Run / update

```bash
.venv-ansible216/bin/ansible-playbook -i inventory.ini install-alloy.yml
```

The playbook is idempotent — re-running reconciles state (`changed=0` when
nothing needs to change). Day-2 *pipeline* changes are made in Fleet Management,
not here; re-run Ansible only to change the install, the `remotecfg` block, or
the env file.

## Target a subset of hosts

```bash
.venv-ansible216/bin/ansible-playbook -i inventory.ini install-alloy.yml --limit ubuntu-1
```

## Add a host

1. Add it to `inventory.ini` with the right `ansible_user`
   (`ec2-user` for RHEL, `ubuntu` for Ubuntu).
2. Re-run the playbook (optionally `--limit` to just the new host).

## Pin / change the Alloy version

Set `alloy_version` in `vars.yml` (e.g. `v1.10.0`), then re-run. Use a pinned
version in production for reproducible rollouts.

## Upgrade Alloy

Bump `alloy_version` (or leave `latest`) and re-run. The role installs the new
package and restarts the service via its handler.

## Uninstall

```bash
.venv-ansible216/bin/ansible-playbook -i inventory.ini install-alloy.yml -e "alloy_uninstall=true"
```

The role removes the package, service, config, and env files.

## Rotating credentials

The access token is sensitive and should be rotated if it is ever exposed
(committed in plaintext, pasted into a chat/ticket, etc.).

1. In Grafana Cloud → **Administration → Access Policies**, revoke the current
   token / access policy.
2. Create a new token with the required scopes (`fleet-management:read`, plus
   metrics/logs write scopes for remote_write).
3. Update the vault:
   ```bash
   ansible-vault edit group_vars/alloy_nodes/vault.yml
   ```
4. Re-run the playbook. Only the env file changes — `config.alloy` holds no
   token — and the role's handler restarts Alloy so both planes pick up the new
   `GCLOUD_RW_API_KEY`.

## Verify a host

```bash
ssh <user>@<host> '
  alloy --version
  systemctl is-active alloy
  sudo journalctl -u alloy --no-pager | grep -i remotecfg | tail
  sudo grep -c GCLOUD_RW_API_KEY /etc/default/alloy /etc/sysconfig/alloy 2>/dev/null
'
```

- `remotecfg` log lines mean the config-plane is working (pulling pipelines).
- `401` errors on remote_write mean the **data-plane** key is wrong/missing —
  see [troubleshooting](troubleshooting.md#401-on-remote_write).
