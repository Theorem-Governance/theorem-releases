# First-Boot Networking Contract

## Ownership Model

Network configuration is owned by FreeBSD `rc.conf`, **not** `theorem-init`.

`theorem-init` handles:

- Loopback interface
- DNS resolver (points to localhost via unbound)
- chronyd (NTS time synchronization)
- unbound (DNS-over-TLS recursive resolver)

The NIC/IP/gateway configuration is handled by standard FreeBSD `rc(8)` **before** `theorem-init` runs. theorem-init assumes a working network stack is already present.

## Default Configuration

Source: `freebsd/rc.conf.theoremos`

- **DHCP** on all interfaces (`ifconfig_DEFAULT="DHCP"`)
- **SSH** enabled (`sshd_enable="YES"`)
- **PF firewall** enabled with theorem anchor
- **Serial console** at 115200 baud

## Static IP Provisioning

Three methods are supported:

### 1. At Image Build Time

```sh
NETWORK_CONFIG=static \
STATIC_IP=x.x.x.x \
NETMASK=y.y.y.y \
GATEWAY=z.z.z.z \
./freebsd/build-image.sh
```

### 2. At Install Time

Provide a `first-boot.conf` with the following fields, then run `install.sh`:

```
NETWORK_MODE=static
STATIC_IP=x.x.x.x
NETMASK=y.y.y.y
GATEWAY=z.z.z.z
```

### 3. Post-Install

Edit `/etc/rc.conf` directly. Replace:

```
ifconfig_DEFAULT="DHCP"
```

with the appropriate static configuration:

```
ifconfig_DEFAULT="inet x.x.x.x netmask y.y.y.y"
defaultrouter="z.z.z.z"
```

## Remote Access

- **SSH** is enabled by default on port 22.
- Key-based authentication can be provisioned via the `SSH_AUTHORIZED_KEYS` field in `first-boot.conf`.
- **theorem-node HTTP API** listens on port **3170**.

## DNS

`theorem-init` starts **unbound** as a local recursive resolver on `127.0.0.1`.

Default upstreams (DNS-over-TLS):

- `1.1.1.1@853`
- `9.9.9.9@853`

Override via:

- `dns_upstream` in `theorem.conf`
- `DNS_UPSTREAM` in `first-boot.conf`

## Time Synchronization

`theorem-init` starts **chronyd** with NTS (Network Time Security).

Default servers:

- `time.cloudflare.com`
- `nts.netnod.se`

Override via:

- `nts_servers` in `theorem.conf`
- `NTS_SERVERS` in `first-boot.conf`

## Firewall

PF is enabled with a theorem anchor. Firewall rules are managed by `theorem-net`.

Per-workload rules are configured via:

- `allowed_egress` — CIDR list for outbound traffic
- `allowed_ingress` — CIDR list for inbound traffic
- `dns_allowed` — flag controlling DNS access

## Serial Console

115200 baud, always enabled for headless server access.

## Deterministic Remote Access Guarantee

- **DHCP boot**: After first boot with DHCP, the host is reachable via SSH on whatever IP DHCP assigns.
- **Static IP boot**: After first boot with a static IP from `first-boot.conf`, the host is reachable at that IP via SSH.
- **In both cases**: `theorem-node` API is reachable at `:3170/health`.
