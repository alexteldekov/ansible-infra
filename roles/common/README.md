# common

Ansible role for baseline system configuration applied to every host in the infrastructure.

## What it does

- Sets the FQDN entry in `/etc/hosts` (the `127.0.1.1` line) from `fqdn` or `inventory_hostname`, with the short hostname as an alias.
- Sets the system hostname (short name) via the `hostname` module, which updates `/etc/hostname` and notifies `systemd-hostnamed`.
- Sets the timezone via the `timezone` module.
- Installs diagnostic and security packages: `htop`, `iotop`, `ufw`, `fail2ban`.
- Enables UFW (unless `ufw_enabled: false`).
- Grants passwordless sudo to the `sudo` group via `/etc/sudoers.d/sudo-group-nopasswd`.

## Variables

This role has no `defaults/main.yml`; variables are expected to be defined in group/host vars.

| Variable | Default | Description |
|---|---|---|
| `fqdn` | `inventory_hostname` | Fully qualified domain name written to `/etc/hosts` and used as the hostname source |
| `timezone` | — | System timezone, e.g. `Europe/Moscow` (required) |
| `ufw_enabled` | `true` | Whether to enable UFW (set to `false` on hosts where the firewall is managed elsewhere) |

## Tags

| Tag | Tasks |
|---|---|
| `common` | All tasks in the role |
| `hostname` | `/etc/hosts` and system hostname |
| `timezone` | Timezone configuration |
| `packages` | Diagnostic and security package installation |
| `ufw` | UFW enablement |
| `sudo` | Passwordless sudo for the `sudo` group |

## Handlers

| Handler | Action |
|---|---|
| `restart cron` | Restarts the `cron` service (notified on timezone change) |
| `restart networking services` | Restarts `systemd-hostnamed` (notified on hostname/hosts change) |

## Usage

The role is applied to `all` hosts by `playbooks/deploy_common.yml`:

```bash
ansible-playbook playbooks/deploy_common.yml
```

It is also the first role in the umbrella playbook `playbooks/site.yml`, applied to `all` hosts.

To run only specific parts of the role:

```bash
# Only hostname-related tasks
ansible-playbook playbooks/deploy_common.yml --tags hostname

# Skip package installation
ansible-playbook playbooks/deploy_common.yml --skip-tags packages
```
