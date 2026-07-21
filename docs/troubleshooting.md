# Troubleshooting

Real failure modes seen bringing this up, with the fix for each.

## Why ansible-core 2.16

**Symptom** (on RHEL 8 or Ubuntu 18.04, with a newer ansible-core):

```
SyntaxError: future feature annotations is not defined
MODULE FAILURE: No start of json char found
```

**Cause.** The upstream role installs Alloy with the `dnf`/`apt` module. Those
modules must run on the managed node's **system Python**, because that's the
only interpreter with the package-manager bindings (`libdnf` / `python3-apt`).
On RHEL 8 and Ubuntu 18.04 that system Python is **3.6**. ansible-core **2.17+**
dropped the ability to run modules on Python 3.6, so its module code fails to
even parse there.

**Fix.** Run the control node on **ansible-core 2.16.x**, which supports Python
3.6–3.12 on managed nodes — covering RHEL 8 (3.6), RHEL 9 (3.9), and Ubuntu
18.04 (3.6) in a single run. This repo pins it in `requirements.txt`; always
invoke via the venv:

```bash
.venv-ansible216/bin/ansible-playbook -i inventory.ini install-alloy.yml
```

There is no way to keep the upstream role *and* use 2.17+ on those distros —
the package bindings simply don't exist for a newer Python there.

## `rc 137` during package install (OOM)

**Symptom:**

```
TASK [grafana.grafana.alloy : DNF - Install Alloy from remote URL]
fatal: [host]: FAILED! => {"changed": false, "module_stderr": "...", "rc": 137}
```

Empty output, `rc 137` = process killed by SIGKILL. Checking the host:

```
Out of memory: Killed process NNNN (platform-python) ... anon-rss:525688kB
```

**Cause.** The `dnf`/`apt` module loads package metadata into memory and peaks
around ~500 MiB. On a <1 GiB instance with no swap, the kernel OOM-kills it.

**Fix.** The playbook auto-provisions a 1 GiB swap file on hosts under 2 GiB
RAM (skipped on larger hosts). If you hit this on a host that somehow slipped
past, add swap manually:

```bash
sudo dd if=/dev/zero of=/swapfile bs=1M count=1024
sudo chmod 600 /swapfile && sudo mkswap /swapfile && sudo swapon /swapfile
```

## `401` on remote_write

**Symptom.** Alloy is running and `remotecfg` log lines are present, but logs/
metrics fail:

```
level=error msg="final error sending batch, no retries left, dropping data"
  component=endpoint host=logs-prod-006.grafana.net status=401
  error="authentication error: invalid token"
```

**Cause.** This is the **data-plane**, not remotecfg. The pipeline delivered by
Fleet Management reads its write credential from `sys.env("GCLOUD_RW_API_KEY")`.
If that env var is missing or the token lacks metrics/logs **write** scope, every
remote_write is rejected with `401`. remotecfg itself is fine — it already pulled
the config.

**Fix.**
1. Ensure `alloy_env_file_vars.GCLOUD_RW_API_KEY` is set in `vars.yml` (it is by
   default, sourced from the vault).
2. Ensure the token has the required write scopes (not just
   `fleet-management:read`). If your write token differs from the config-plane
   token, add a separate vault var and point `GCLOUD_RW_API_KEY` at it.
3. Re-run the playbook and confirm on the host:
   ```bash
   sudo grep GCLOUD_RW_API_KEY /etc/default/alloy   # or /etc/sysconfig/alloy
   ```

## `python3-apt` missing (Ubuntu)

**Symptom.** The `apt` module fails to import its Python bindings on a minimal
Ubuntu image.

**Fix.** The playbook installs `python3-apt` via a `raw` pre-task before any
apt-based task. If you removed that task, restore it or install the package
manually.

## Readiness check fails behind a corporate proxy

**Symptom.** The "Wait for Alloy to report ready" task (or the role's own
readiness check) fails/times out on some hosts even though Alloy is running and
`curl http://127.0.0.1:12345/-/ready` returns `200` on the host itself. Often it
fails only on corporate hosts and passes on cloud VMs.

**Cause.** The host has `http_proxy`/`https_proxy` set, and the `uri` module
routes the `127.0.0.1` request *through the proxy*, which refuses it.

**Fix.** This repo bypasses the proxy for the readiness check:
- our post-task runs a local `curl --noproxy '*'` on the host, and
- `alloy_readiness_check_use_proxy: false` disables it for the role's check.

If you still see it, confirm the proxy env on the host and that `no_proxy`
includes `127.0.0.1,localhost`:

```bash
ssh <user>@<host> 'env | grep -i proxy; systemctl show alloy -p Environment'
```

## Host unreachable

**Symptom:** `ssh: connect to host X port 22: No route to host`.

**Cause.** The host is on a private network (e.g. a `10.x` address) not
reachable from the control node.

**Fix.** Run the playbook from a control node with network access to the target
(VPN, bastion/jump host via `ansible_ssh_common_args`, or an in-VPC runner).

## `Invalid callback for stdout specified: yaml`

**Cause.** A `stdout_callback` from `community.general` that is too new for
ansible-core 2.16, or the collection isn't on the collections path.

**Fix.** This repo uses the default callback. If you re-add a community callback,
pin `community.general` to an 8.x release (see `requirements.yml`).
