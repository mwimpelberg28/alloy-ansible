# Configuration

All tunables live in `group_vars/alloy_nodes/`. Non-secret settings go in
`vars.yml`; credentials go in the encrypted `vault.yml`.

## `vars.yml`

| Variable | Default | Purpose |
|----------|---------|---------|
| `ansible_python_interpreter` | `auto` | Let Ansible discover the system Python per distro (`platform-python` on RHEL, `/usr/bin/python3` on Ubuntu). |
| `alloy_version` | `latest` | Alloy release to install. Pin to a tag (e.g. `v1.10.0`) for reproducibility. |
| `alloy_remotecfg_url` | — | Fleet Management endpoint for your stack. |
| `alloy_remotecfg_poll_frequency` | `60s` | How often the collector polls for config. |
| `alloy_remotecfg_username` | vault ref | Fleet Management instance/stack ID. |
| `alloy_remotecfg_password` | vault ref | Access token (config-plane). |
| `alloy_env_file_vars` | see below | Extra env vars written to the systemd env file. |
| `alloy_config` | see below | The rendered `config.alloy` contents. |

### `alloy_env_file_vars`

Merged by the role into the systemd env file and loaded as process environment:

```yaml
alloy_env_file_vars:
  GCLOUD_RW_API_KEY: "{{ vault_alloy_remotecfg_password }}"
```

`GCLOUD_RW_API_KEY` is the **data-plane** credential the Fleet-Management-
delivered pipeline uses for remote_write. It is referenced inside delivered
configs as `sys.env("GCLOUD_RW_API_KEY")`. The playbook locks this file to
`0600` in a `post_task`.

### The `remotecfg` block

Rendered from `alloy_config`:

```alloy
remotecfg {
  url = "https://fleet-management-prod-NNN.grafana.net"

  basic_auth {
    username = "<instance id>"
    password = "<access token>"
  }

  id             = constants.hostname   // how the host appears in Fleet Mgmt
  poll_frequency = "60s"

  attributes = {
    "platform" = "ubuntu",              // set from ansible_facts.distribution
    "env"      = "production",
  }
}
```

- `id` uses `constants.hostname` so each collector is identifiable in the Fleet
  Management UI.
- `attributes` are how you target collectors when assigning pipelines centrally.
  `platform` is filled from the host's distribution automatically; add your own
  (region, role, team, …) as needed.

## The vault (`vault.yml`)

Holds the real credentials, encrypted with Ansible Vault:

```yaml
vault_alloy_remotecfg_username: "123456"
vault_alloy_remotecfg_password: "glc_xxxxxxxxxxxxxxxxxxxxxxxx"
```

- `vault_alloy_remotecfg_username` — Fleet Management instance/stack ID.
- `vault_alloy_remotecfg_password` — access token. Used for both the config-plane
  `basic_auth` and (via `alloy_env_file_vars`) the data-plane
  `GCLOUD_RW_API_KEY`. If your setup needs a *separate* write token, add a second
  vault var and point `GCLOUD_RW_API_KEY` at it.

Create and encrypt it:

```bash
cp group_vars/alloy_nodes/vault.yml.example group_vars/alloy_nodes/vault.yml
# edit values
ansible-vault encrypt group_vars/alloy_nodes/vault.yml
```

`ansible.cfg` sets `vault_password_file = .vault_pass`, so runs pick up the
passphrase automatically. Both `.vault_pass` and `vault.yml`'s decrypted form
are git-ignored — only the encrypted `vault.yml` is safe to commit.

## Inventory

`ansible_user` is per host because RHEL AMIs log in as `ec2-user` and Ubuntu
AMIs as `ubuntu`:

```ini
[alloy_nodes]
rhel-1     ansible_host=10.0.0.11 ansible_user=ec2-user
ubuntu-1   ansible_host=10.0.0.50 ansible_user=ubuntu
```
