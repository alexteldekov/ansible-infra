# vcgnat

Ansible role to install and configure NFWare VCGNAT on **Ubuntu 20.04 Focal** (hard-pinned; the role fails on any other OS).

## What it does

- Installs virtualization packages (`libvirt-daemon-system`, `qemu-kvm`, `msr-tools`) and `snmpd`.
- Downloads the VCGNAT QCOW2 image (MD5-verified) and the hw-configuration archive.
- Installs local `.deb` packages from hw-configuration and **holds** them via `dpkg_selections` so Ubuntu repo upgrades don't override them.
- Configures `snmpd` to listen on the network and answer only to configured Zabbix sources.

## Variables

See `defaults/main.yml`. Key ones:

| Variable | Default | Description |
|---|---|---|
| `vcgnat_libvirt_dir` | `/var/lib/libvirt/images` | Where images and hw-configuration are stored |
| `vcgnat_qcow2_url` | NFWare URL | QCOW2 image URL |
| `vcgnat_qcow2_md5` | pinned | MD5 checksum for the QCOW2 |
| `snmpd_listen_ip` | `0.0.0.0` | Address for snmpd to listen on |
| `snmpd_allowed_sources` | `[]` | List of Zabbix server IPs allowed to query snmpd |
| `snmpd_community` | `changeme` | SNMP community string (override in vault) |
| `snmpd_sys_location` | `Russia` | sysLocation value |
| `snmpd_sys_contact` | `root@localhost` | sysContact value |

`snmpd_community` should be set in `inventory/production/group_vars/all/vault.yml` (encrypted).

## Tags

Tasks are tagged so you can run only part of the role:

| Tag | Tasks |
|---|---|
| `always` | OS version check (runs on every invocation) |
| `install` | apt packages (libvirt, snmpd) |
| `snmpd` | snmpd.conf template |
| `download` | QCOW2 + hw-configuration downloads, libvirt dir |
| `hwconfig` | Extract archive, install + hold local debs |

Run only the snmpd configuration:

```bash
ansible-playbook playbooks/deploy_vcgnat.yml --tags snmpd
```

## Verifying snmpd from another host

Use `snmpwalk` (from the `snmp` package) to confirm snmpd answers from a remote host:

```bash
# Install the client (Debian/Ubuntu)
sudo apt install snmp

# Query the system tree
snmpwalk -v 2c -c <community> <vcgnat-host-ip> system
```

Example with the default community and a host at `194.8.47.5`:

```bash
snmpwalk -v 2c -c public 194.8.47.5 system
```

A successful response lists `SNMPv2-MIB::sysDescr.0` and the rest of the `system` tree. If you get `Timeout: No Response`, check that:

- `snmpd_allowed_sources` in your inventory includes your client IP,
- the community string matches,
- UDP port 161 is reachable (no firewall in between).
