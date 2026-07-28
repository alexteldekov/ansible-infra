# logwatch

Ansible role to install **logwatch** and configure a local **postfix** MTA for daily report delivery.

## What it does

- Installs `debconf-utils` and pre-seeds postfix debconf values (mailer type, mailname, relayhost, myorigin, protocols) for non-interactive installation.
- Installs `logwatch` and `postfix` packages (or just `logwatch` when `postfix_install: false`).
- Explicitly sets `inet_protocols = ipv4` in `/etc/postfix/main.cf` via `lineinfile`. This is necessary because debconf values are only applied by the package's postinst script during install/reconfigure — if postfix is already installed, `apt: state: present` is a no-op and the debconf `postfix/protocols` value never reaches `main.cf` (it only ends up in `main.cf.proto`).
- Ensures postfix is started and enabled.
- Configures root mail forwarding in `/etc/aliases` (root → `logwatch_email`) and rebuilds the aliases database.
- Deploys `/etc/logwatch/conf/override.conf` from a template, sending daily reports to `root` (which is then forwarded via aliases) in HTML format.

## Variables

See `defaults/main.yml`:

| Variable | Default | Description |
|---|---|---|
| `logwatch_email` | `alext@t7g.org` | Address root mail is forwarded to (report recipient) |
| `logwatch_detail` | `Med` | Report detail level: `Low`, `Med`, `High` |
| `logwatch_archive` | `No` | Whether to archive old reports (`Yes`/`No`) |
| `postfix_install` | `true` | Install and configure postfix as the MTA |
| `postfix_type` | `Internet Site` | Postfix mailer type (`Internet Site` / `Satellite` / `Local only`) |
| `postfix_relayhost` | `""` | Relay host, e.g. `smtp:[mail.example.com]:587` |
| `postfix_myorigin` | `localhost` | Envelope sender origin |

## Handlers

| Handler | Action |
|---|---|
| `restart postfix` | Restarts the postfix service (notified when `inet_protocols` changes) |
| `rebuild aliases` | Runs `newaliases` to rebuild `/etc/aliases.db` |
| `restart logwatch cron` | Restarts the `cron` service (logwatch runs via cron) |

## Usage

The role is applied to `vpn_servers` by `playbooks/deploy_logwatch.yml`:

```bash
ansible-playbook playbooks/deploy_logwatch.yml
```

It is also included in the umbrella playbook `playbooks/site.yml` under the `vpn_servers` group.
