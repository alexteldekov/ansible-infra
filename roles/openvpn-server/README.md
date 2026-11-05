# openvpn-server

Ansible role to install and configure a full **OpenVPN** server with **Easy-RSA** as the certificate authority, including client config bundle (`.ovpn`) generation for two client types.

## What it does

- Installs `openvpn` and `easy-rsa` packages.
- Initializes an Easy-RSA PKI at `{{ easyrsa_dir }}` and builds the CA (idempotent — skipped if `ca.crt` already exists).
- Generates the server certificate, DH parameters, CRL, and a TLS auth key.
- Generates client certificates for every entry in `openvpn_clients`.
- Deploys `/usr/local/sbin/openvpn-add-client` — a helper script to create clients directly on the server, without Ansible (see below).
- Deploys `/etc/openvpn/server.conf` from a template: TCP/443, `subnet` topology, pushes a route to `home_network`. Optional IPv6 in the tunnel (`server-ipv6` with a ULA prefix + NAT66 masquerade on exit).
- Deploys a Python script (`/usr/local/sbin/update_openvpn_routes.py`) + systemd timer that daily fetches routes from [antifilter.download](https://antifilter.download/list/allyouneed.lst) and a locally-maintained `routes_custom.lst`, generates `routes.inc`, and restarts OpenVPN if routes changed.
- Creates the CCD directory (`{{ ccd_dir }}`) and per-client config files (static IPs + `iroute` for router-clients).
- Generates ready-to-use `.ovpn` client bundles in `{{ client_config_dir }}` with embedded certificates and keys.
- Enables IP forwarding via `sysctl`.
- Opens the OpenVPN port in UFW and allows forwarded traffic from `tun0` (VPN clients → internet/LAN). UFW's default forward policy is DROP, so without the route rule clients can connect but get no traffic through.
- Adds NAT (`MASQUERADE`) for the VPN subnet via `/etc/ufw/before.rules` — without it, client packets leave the server with `10.8.2.x` source addresses and replies never come back.
- Enables and starts the `openvpn@server` service.

## Client types

Two client types are supported via the `type` field:

### `router` — Keenetic/Netcraze

A router with a home network behind it. Does **not** redirect the default route through the VPN — only specific subnets are routed (home network + subnets behind other router-clients). The `.ovpn` bundle contains explicit `route` directives and no `redirect-gateway`.

### `mobile` — iPhone (OpenVPN Connect) / macOS (Tunnelblick)

A mobile device that tunnels **all** traffic through the VPN. The `.ovpn` bundle contains `redirect-gateway def1 bypass-dhcp`, DNS server options, `block-outside-dns`, and `block-ipv6` (redirect-gateway only covers IPv4 — without blocking, dual-stack services would leak around the VPN via native IPv6).

## Variables

See `defaults/main.yml`:

| Variable | Default | Description |
|---|---|---|
| `openvpn_protocol` | `tcp` | Transport protocol (`tcp` or `udp`) |
| `openvpn_port` | `443` | Listening port |
| `openvpn_topology` | `subnet` | Network topology |
| `openvpn_network` | `10.8.2.0` | VPN subnet address |
| `openvpn_netmask` | `255.255.255.0` | VPN subnet netmask |
| `openvpn_ipv6_enabled` | `true` | IPv6 in the tunnel (`server-ipv6`, NAT66, `route-ipv6` pushes). Disable if the server has no IPv6 connectivity |
| `openvpn_ipv6_prefix` | `fd00:10:8:2::/64` | ULA prefix for tunnel IPv6 (NAT66 on exit) |
| `home_network` | `192.168.0.0` | Home network pushed to clients |
| `home_netmask` | `255.255.0.0` | Home network netmask |
| `easyrsa_dir` | `/etc/openvpn/easyrsa` | Easy-RSA working directory |
| `ccd_dir` | `/etc/openvpn/ccd` | Client config directory |
| `client_config_dir` | `/etc/openvpn/client-configs` | Output dir for `.ovpn` bundles |
| `openvpn_remote` | `ansible_host` | Remote endpoint in client configs (IP, not DNS — avoids DNS interception) |
| `openvpn_remote_port` | `openvpn_port` | Remote port in client configs |
| `openvpn_remote_proto` | `openvpn_protocol` | Remote protocol in client configs |
| `openvpn_dns_servers` | `[8.8.8.8, 1.1.1.1]` | DNS servers pushed to mobile clients |
| `openvpn_search_domain` | `domain` | Search domain for mobile clients |
| `openvpn_server_name` | `server` | Server certificate name |
| `easyrsa_key_size` | `2048` | RSA key size |
| `easyrsa_ca_expire` | `3650` | CA certificate expiry (days) |
| `easyrsa_cert_expire` | `3650` | Client/server cert expiry (days) |
| `easyrsa_crl_expire` | `3650` | CRL expiry (days) |
| `openvpn_cipher` | `AES-256-GCM` | Data channel cipher |
| `openvpn_auth` | `SHA256` | Auth digest |
| `openvpn_tls_auth_key` | `/etc/openvpn/ta.key` | TLS auth key path |
| `openvpn_clients` | `[]` | List of clients (see below) |
| `openvpn_ip_forward` | `true` | Whether to enable IP forwarding |
| `openvpn_nat_subnet` | `10.8.2.0/24` | VPN subnet for NAT masquerade (must match `openvpn_network`/`openvpn_netmask`) |
| `openvpn_routes_script` | `/usr/local/sbin/update_openvpn_routes.py` | Path to the route update script |
| `openvpn_routes_inc` | `/etc/openvpn/routes.inc` | Path to generated routes file (included by `server.conf`) |
| `openvpn_routes_custom_list` | `/etc/openvpn/routes_custom.lst` | Admin-maintained custom routes file |
| `openvpn_antifilter_url` | `https://antifilter.download/list/allyouneed.lst` | External route list URL |
| `openvpn_iplist_services` | `[]` | Services from iplist.opencck.org to route via VPN (e.g. `[youtube.com, discord.com]`) |
| `openvpn_iplist_base_url` | `https://iplist.opencck.org` | Base URL of the iplist portal |
| `openvpn_iplist_batch_size` | `20` | Services per HTTP request (batching) |
| `openvpn_iplist_include_ip4` | `false` | Include individual service IPs as `/32` routes (large services have tens of thousands) |
| `openvpn_iplist_include_ip6` | `false` | Include individual service IPv6 addresses as `/128` `route-ipv6` pushes |
| `openvpn_russia_exclude_enabled` | `true` | Push Russian networks via `net_gateway` (ISP route) instead of VPN |
| `openvpn_russia_exclude_url` | `https://russia.iplist.opencck.org/?format=json&data=cidr4` | Russian networks list URL (IPv4) |
| `openvpn_russia_exclude_v6_url` | `https://russia.iplist.opencck.org/?format=json&data=cidr6` | Russian networks list URL (IPv6, used as a filter) |

### `openvpn_clients` format

```yaml
openvpn_clients:
  # Router: Keenetic with a home subnet behind it, no default route
  - name: router1
    type: router
    static_ip: 10.8.2.20
    client_subnet: 192.168.10.0    # subnet behind the router
    client_netmask: 255.255.255.0

  # Mobile: iPhone, full tunnel
  - name: iphone-alex
    type: mobile
    static_ip: 10.8.2.30

  # Mobile: macOS Tunnelblick, full tunnel, dynamic IP
  - name: macbook-jane
    type: mobile

  # Router without IPv6 (v4-only client or client with no IPv6 support)
  - name: gw-old
    type: router
    static_ip: 10.8.2.6
    ipv6: false
```

Optional per-client field `ipv6: false` disables IPv6 for that client when `openvpn_ipv6_enabled` is `true` — the CCD file gets `push-remove ifconfig-ipv6` (drops the tunnel v6 address from the `server-ipv6` pool) and `push-remove route-ipv6` (drops all v6 route pushes from `routes.inc`). IPv6 cannot be enabled for a single client when it is disabled globally.

For router-clients (`client_subnet` + `client_netmask`), the role generates both:
- `iroute` in the CCD file — tells OpenVPN which client owns the subnet
- `route` in `server.conf` — creates the kernel route via the tun interface

## Tags

| Tag | Tasks |
|---|---|
| `openvpn` | All tasks in the role |
| `packages` | Package installation |
| `pki` | Easy-RSA PKI, CA, server cert, DH, CRL, TLS auth key, client certs |
| `clients` | Client certificate generation, CCD files, `.ovpn` bundles |
| `config` | Server config, routes script, systemd units, CCD files |
| `routes` | Route update script, custom list, systemd service/timer |
| `network` | IP forwarding sysctl |
| `ufw` | UFW firewall rule |
| `service` | OpenVPN service enable/start |

## Handlers

| Handler | Action |
|---|---|
| `restart openvpn` | Restarts the `openvpn@server` service |
| `reload systemd` | Reloads systemd daemon (after unit file changes) |
| `reload ufw` | Reloads UFW (after `before.rules` NAT changes) |

## Usage

The role is applied to `vpn_servers` by `playbooks/deploy_openvpn.yml`:

```bash
ansible-playbook playbooks/deploy_openvpn.yml
```

It is also included in the umbrella playbook `playbooks/site.yml` under the `vpn_servers` group.

To run only specific parts:

```bash
# Only regenerate client configs
ansible-playbook playbooks/deploy_openvpn.yml --tags clients

# Only update server config
ansible-playbook playbooks/deploy_openvpn.yml --tags config
```

### Fetching client configs

Generated `.ovpn` bundles are stored on the VPN server in `{{ client_config_dir }}` (default: `/etc/openvpn/client-configs/`). Fetch them with:

```bash
scp root@<vpn-server>:/etc/openvpn/client-configs/<name>.ovpn .
```

## Notes

- PKI initialization and certificate generation are **idempotent**: each `command` task uses `creates:` to skip if the output file already exists. Re-running the playbook will not regenerate existing certificates.
- `.ovpn` bundles are **re-generated on every run** (they are cheap to render and always reflect current server config).
- To revoke a client, run `./easyrsa revoke <name>` manually in `{{ easyrsa_dir }}` and regenerate the CRL with `./easyrsa gen-crl`.
- Private keys are embedded in the `.ovpn` bundles — the `client-configs` directory is mode `0750` and bundle files are mode `0600`.

## Dynamic routes (`routes.inc`)

`server.conf` includes `config routes.inc`, which is generated daily by `/usr/local/sbin/update_openvpn_routes.py` (runs via `update-openvpn-routes.timer`).

### Sources

1. **iplist.opencck.org services** — `openvpn_iplist_services` (default: `[]`) lists service names as they appear on the site (`youtube.com`, `discord.com`, ...). Services are fetched in batches of `openvpn_iplist_batch_size` (default: `20`) per HTTP request (`?format=json&site=a&site=b&...`) from `openvpn_iplist_base_url` (default: `https://iplist.opencck.org`). Only `cidr4` zones are used; individual `ip4` addresses can be included as `/32` routes via `openvpn_iplist_include_ip4` (default: `false` — large services have tens of thousands of them). With `openvpn_ipv6_enabled` the `cidr6` zones are also pushed as `push "route-ipv6 <cidr>"` (and `ip6` as `/128` via `openvpn_iplist_include_ip6`). A failed batch is skipped with a warning; the rest are still updated.
2. **antifilter.download** — `{{ openvpn_antifilter_url }}` returns a list of CIDR networks (`223.130.134.0/24`). These are converted to `push "route <ip> <netmask> vpn_gateway"` lines. Disabled by default (`openvpn_antifilter_enabled: false`); if enabled, a download failure aborts the update (existing routes are preserved).
3. **`/etc/openvpn/routes_custom.lst`** — admin-maintained file, one entry per line:
   - `149.154.164.0/255.255.252.0` — network/netmask
   - `149.154.164.0/22` — CIDR
   - `apps.apple.com` — hostname (resolved to IPv4, each address becomes a `/32` route)
   - Lines starting with `#` are ignored
4. **Russian networks exclusion** — `openvpn_russia_exclude_enabled` (default: `true`): IPv4 zones from `openvpn_russia_exclude_url` (default: `https://russia.iplist.opencck.org/?format=json&data=cidr4`) are pushed as `push "route <ip> <netmask> net_gateway"` — via the client's original default gateway (mainly relevant for mobile clients with `redirect-gateway`). For IPv6, `route-ipv6` on `tun` cannot express `net_gateway`, so zones from `openvpn_russia_exclude_v6_url` (default: `...&data=cidr6`) act as a **filter**: any service/custom IPv6 route overlapping a Russian network is not pushed at all — clients reach it via their native ISP route. Networks already routed via VPN (sources 1-3) are not duplicated in the v4 exclusions. A download failure aborts the update (existing routes are preserved).

If the generated `routes.inc` differs from the current one, OpenVPN is restarted so clients receive the updated routes. If the antifilter download fails, the update is aborted (existing routes are preserved).

### Manual update

```bash
/usr/local/sbin/update_openvpn_routes.py
```

## Creating clients without Ansible (`openvpn-add-client`)

For ad-hoc clients, run the helper script directly on the VPN server:

```bash
# Router client with a subnet behind it and a fixed IP
openvpn-add-client keenetic2 router 10.8.2.21 192.168.20.0 255.255.255.0

# Mobile client, auto-picked free IP
openvpn-add-client ipad-guest mobile

# Router client without IPv6
openvpn-add-client gw-old router 10.8.2.22 --no-ipv6
```

The script generates the certificate, the CCD file, and a self-contained `.ovpn` bundle in `{{ client_config_dir }}` — identical in format to the Ansible-managed ones.

Caveats:

- Such clients are **unmanaged**: Ansible doesn't know about them and won't update their CCD/`.ovpn` (it won't delete them either). To bring a client under Ansible management, add it to `openvpn_clients` — the existing certificate will be reused.
- For a router-client's `client_subnet`, the script writes only `iroute` (no kernel `route` in `server.conf`, since it is Ansible-managed). The subnet is still reachable from other VPN clients via `client-to-client`, but the server itself won't route to it. Add the client to `openvpn_clients` if a kernel route is needed.
