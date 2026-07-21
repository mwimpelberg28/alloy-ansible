# Architecture

## Overview

This playbook installs Grafana Alloy and points it at Grafana Cloud Fleet
Management. Once running, each host manages almost nothing locally — it fetches
its telemetry pipeline from the server and keeps it up to date.

```
                        control node (ansible-core 2.16)
                                     │
                     ansible-playbook install-alloy.yml
                                     │
        ┌────────────────────────────┼────────────────────────────┐
        ▼                            ▼                            ▼
   RHEL 8 host                  RHEL 9 host                 Ubuntu host
        │                            │                            │
   installs alloy pkg          installs alloy pkg          installs alloy pkg
   /etc/alloy/config.alloy     (same)                      (same)
   /etc/sysconfig/alloy        (same)                      /etc/default/alloy
        │                            │                            │
        └──────────────── alloy.service (systemd) ───────────────┘
                                     │
                          remotecfg { url = ... }
                                     │  poll every 60s
                                     ▼
                    Grafana Cloud Fleet Management
                                     │  delivers pipeline
                                     ▼
              remote_write to Grafana Cloud (metrics/logs)
              authenticated with sys.env("GCLOUD_RW_API_KEY")
```

## The two credentials

There are **two distinct auth paths**, and confusing them is the most common
source of trouble:

1. **Config-plane (`remotecfg` basic_auth).** Alloy authenticates to Fleet
   Management to *pull its configuration*. Username = Fleet Management
   instance/stack ID, password = an access token. Configured directly in the
   `remotecfg` block in `config.alloy`.

2. **Data-plane (`GCLOUD_RW_API_KEY`).** The pipeline that Fleet Management
   *delivers* writes metrics/logs to Grafana Cloud. That delivered config reads
   its write credential from the environment via `sys.env("GCLOUD_RW_API_KEY")`.
   We supply it through the systemd env file — **not** through `config.alloy`.

If the config-plane token is wrong, the host can't fetch config at all. If the
data-plane key is wrong or missing, the host fetches config fine but every
remote_write returns `401` — see
[troubleshooting](troubleshooting.md#401-on-remote_write).

## What the playbook does

`install-alloy.yml` runs in three phases:

### 1. `pre_tasks`
- **`python3-apt` bootstrap** (Debian/Ubuntu only, via `raw`) so the `apt`
  module can run even on minimal images.
- **Swap provisioning** on hosts under 2 GiB RAM — the package modules peak
  around ~500 MiB and get OOM-killed on tiny instances.
- **Assertions** that the host is a supported OS family and that the
  remotecfg credentials are configured.

### 2. `roles: grafana.grafana.alloy`
The upstream role installs the Alloy package (RPM via `dnf`, `.deb` via `apt`,
downloaded from GitHub releases), templates `config.alloy` and the env file,
and manages the `alloy` systemd service.

### 3. `post_tasks`
- Tightens the env file (which now carries `GCLOUD_RW_API_KEY`) to `0600`.

## On-host files

| Path | Purpose |
|------|---------|
| `/etc/alloy/config.alloy` | The bootstrap config (logging + `remotecfg`). |
| `/etc/sysconfig/alloy` (RHEL) / `/etc/default/alloy` (Ubuntu) | systemd `EnvironmentFile`; carries `GCLOUD_RW_API_KEY`. |
| `alloy.service` | systemd unit shipped by the package. |

`config.alloy` intentionally stays tiny — everything else is delivered by Fleet
Management at runtime, so day-2 changes happen centrally, not by re-running
Ansible.
