# paqet-plus

A packet-level proxy that tunnels encrypted [KCP](https://github.com/xtaci/kcp-go) over raw TCP packets. Built for networks where VPN protocols are blocked or throttled by DPI.

paqet-plus is a fork of [hanselime/paqet](https://github.com/hanselime/paqet). It adds DPI-evasion tuning, a one-line installer and control panel, connection autoscaling, send pacing, and a faster Linux packet path. It stays wire-compatible with paqet.

> **Status:** alpha. It runs in production for the maintainer, but expect changes between releases.

## How it works

paqet-plus builds TCP frames itself and puts them straight on the wire, bypassing the kernel's TCP stack. Each frame carries KCP segments, encrypted and multiplexed with [smux](https://github.com/xtaci/smux), so the traffic looks like ordinary TCP rather than a known tunnel protocol. Header fields such as TTL, window, MSS and timestamps vary per packet to avoid a fixed fingerprint.

```
 Client                                       Server
┌──────────────┐   raw TCP frames         ┌──────────────┐
│ SOCKS5       │ ───────────────────────> │              │
│ or           │   KCP + smux + cipher    │   Internet   │
│ port forward │ <─────────────────────── │   gateway    │
└──────────────┘                          └──────────────┘
```

On Linux, packets go through raw AF_PACKET sockets: sends are batched with `sendmmsg`, receives with `recvmmsg`, and libpcap only compiles the port filter. Windows and macOS use libpcap/Npcap.

## Features

- **Raw TCP transport** invisible to kernel connection tracking, with per-packet header randomization
- **KCP over smux**: many proxied connections share a few tunnel sessions
- **Send pacing** at each session's window rate, which smooths the bursts KCP sends with congestion control off
- **Connection autoscaling** between `conn` and `2 × conn` sessions, driven by the number of active users
- **SOCKS5 proxy** (TCP and UDP, optional username/password) and **static port forwarding** (TCP/UDP)
- **Encryption** with `aes-128-gcm` by default, the only cipher that also detects tampering
- **Optional FEC** (Reed-Solomon) for lossy links
- **One-line installer**, systemd service, firewall rules, and the `paqet-plus-ctl` control panel
- **Static multi-arch releases**: Linux amd64, arm64, arm32, mips, mipsle, mips64, mips64le; Windows amd64; macOS amd64 and arm64

## Performance

Measured between a client in Iran and a server in Germany, 78 ms apart, with the default settings (`conn: 8`, mode `normal`):

| traffic | upload | download |
|---|---|---|
| 1 connection | ~105 Mbit/s | ~105 Mbit/s |
| 4 connections | ~400 Mbit/s | ~385 Mbit/s |
| 16 connections | ~650–740 Mbit/s | ~600–690 Mbit/s |

- A single connection is limited to about 100 Mbit/s by its session's window rate (`sndwnd × MTU ÷ RTT`). Many connections share the total.
- Round-trip latency through the tunnel is about the path RTT: 74 ms median on that 78 ms path.
- Batched sends cut the sender's CPU by about 18% per Mbit/s. At ~600 Mbit/s a 2-vCPU server ran at about 1.3 cores.

## Quick install (Linux)

Run on both the server and the client:

```bash
curl -fsSL https://raw.githubusercontent.com/LivingG0D/paqet-plus/master/setup-paqet-plus.sh | sudo bash
```

The installer:

1. installs dependencies (`libpcap-dev`, `iproute2`, `curl`, `jq`)
2. downloads the latest release for your architecture
3. runs a configuration wizard (server or client) that writes the recommended settings
4. creates the `paqet-plus` systemd service
5. adds the required iptables rules on a server
6. installs the `paqet-plus-ctl` control panel

Run the same command again to update. Pick **1) Update** to keep your config.

### Control panel

```bash
sudo paqet-plus-ctl
```

It shows logs, live stats and bottleneck alerts, takes a diagnostics snapshot, edits the config, tunes performance (mode, MTU, connections, buffers, cipher), restarts or stops the service, and updates paqet-plus.

## Configuration

Configs are YAML. Annotated examples for both roles are in [`example/`](example/). The KCP `key`, `block` cipher and FEC settings must match on both ends.

### Server

```yaml
role: "server"
log:
  level: "info"
network:
  interface: "eth0"
  ipv4:
    addr: "10.0.0.100:9999"      # the server listens on this port
    router_mac: "aa:bb:cc:dd:ee:ff"
  pcap:
    sockbuf: 16777216
transport:
  protocol: "kcp"
  kcp:
    key: "your-secret-key"       # generate one with: paqet-plus secret
```

### Client

```yaml
role: "client"
log:
  level: "info"
socks5:
  - listen: "127.0.0.1:1080"
forward:
  - listen: "127.0.0.1:8080"
    target: "127.0.0.1:80"       # dialed by the server
    protocol: "tcp"
network:
  interface: "eth0"
  ipv4:
    addr: "192.168.1.100:0"      # port 0 = random
    router_mac: "aa:bb:cc:dd:ee:ff"
server:
  addr: "10.0.0.100:9999"
transport:
  protocol: "kcp"
  kcp:
    key: "your-secret-key"
```

### IPv6

paqet-plus speaks IPv6 as well as IPv4, on the raw path and in the config. Add an `ipv6` block next to (or instead of) `ipv4`; at least one is required, and when both are set they must use the same port.

```
network:
  interface: "eth0"
  ipv4:
    addr: "10.0.0.100:9999"
    router_mac: "aa:bb:cc:dd:ee:ff"
  ipv6:
    addr: "[2001:db8::100]:9999"   # global address, in brackets
    router_mac: "aa:bb:cc:dd:ee:01" # MAC of the IPv6 gateway
```

- **Server:** listens on every family you configure (dual stack works).
- **Client:** the family of `server.addr` decides which local block is used, so a client that dials `[2001:db8::100]:9999` needs an `ipv6` block, and one that dials `203.0.113.7:9999` needs `ipv4`. Port `0` still means a random port.
- **Gateway MAC:** IPv6 uses NDP, not ARP. Find it with `ip -6 route show default` (the gateway is usually a `fe80::` address), then `ping -6 -c1 -I eth0 <gateway>` and `ip -6 neigh show <gateway> dev eth0`.
- **Firewall:** on a server, repeat the three rules with `ip6tables` (the installer does this for you):

```
sudo ip6tables -t raw    -A PREROUTING -p tcp --dport 9999 -j NOTRACK
sudo ip6tables -t raw    -A OUTPUT     -p tcp --sport 9999 -j NOTRACK
sudo ip6tables -t mangle -A OUTPUT     -p tcp --sport 9999 --tcp-flags RST RST -j DROP
```

- Use a global (not temporary, not `fd00::/8`) address, and keep both ends on the same interface.
- Forwards and SOCKS5 can listen on or target IPv6 too: `listen: "[::1]:1080"`.

### Defaults

A key left out of the config gets the value that measured best on a shared 78 ms path, the same the installer writes:

| key | default | notes |
|---|---|---|
| `transport.conn` | `8` | 4 lost a third of the throughput; 16 gained nothing |
| `kcp.mode` | `normal` | `fast3` lost 25–38% with 36% retransmits |
| `kcp.mtu` | `1250` | lower it if large packets get dropped |
| `kcp.sndwnd` / `kcp.rcvwnd` | `1024` | 2048 lost 13–21% and raised retransmits to 20–25% |
| `kcp.smuxbuf` / `kcp.streambuf` | 16 MB / 4 MB | a stream can't exceed `streambuf ÷ RTT` |
| `kcp.block` | `aes-128-gcm` | |
| FEC | off | `dshard: 10`, `pshard: 1` to enable |

### Server firewall (required)

The kernel would otherwise reset the connections paqet-plus builds. The installer adds these rules; add them yourself on a manual install:

```bash
sudo iptables -t raw    -A PREROUTING -p tcp --dport 9999 -j NOTRACK
sudo iptables -t raw    -A OUTPUT     -p tcp --sport 9999 -j NOTRACK
sudo iptables -t mangle -A OUTPUT     -p tcp --sport 9999 --tcp-flags RST RST -j DROP
```

Replace `9999` with the port in `network.ipv4.addr`. If the tunnel connects but no data flows, check these rules first.

## Using it behind Xray / x-ui

A common setup puts paqet-plus between two Xray servers: users connect to Xray on the client host, and Xray sends their traffic through the tunnel.

1. On the client, add a forward that listens on loopback only, for example `127.0.0.1:17108` → `127.0.0.1:17108`.
2. On the server, run an Xray inbound on `127.0.0.1:17108`. Plain VLESS without encryption is enough, since paqet-plus already encrypts the tunnel.
3. On the client, point the Xray outbound at `127.0.0.1:17108`.

Keep both ends on `127.0.0.1`. A forward listening on `0.0.0.0` lets anyone on the internet reach the server's local services through the tunnel.

## Commands

| command | what it does |
|---|---|
| `paqet-plus run -c config.yaml` | start the server or client |
| `paqet-plus secret` | generate a random 32-byte key |
| `paqet-plus ping` | send a single raw TCP packet to test the path |
| `paqet-plus dump` | log TCP payloads for a port (debugging) |
| `paqet-plus version` | show version and build info |

`run`, `ping` and `dump` need root, because they open raw sockets.

## KCP modes

| mode | behaviour | when to use |
|---|---|---|
| `normal` | 40 ms update interval, flushes on write | default; best measured throughput on shared paths |
| `fast` | 30 ms interval, nodelay on | low-traffic links that want slightly faster recovery |
| `fast2` | 20 ms interval, immediate ACKs | real-time traffic, at higher overhead |
| `fast3` | 15 ms interval, immediate ACKs | lowest latency; lost throughput in tests |
| `manual` | your `nodelay`/`interval`/`resend`/`nocongestion` values | full control |

## Encryption

`aes-128-gcm` (default) · `aes` · `aes-128` · `aes-192` · `salsa20` · `blowfish` · `twofish` · `cast5` · `3des` · `tea` · `xtea` · `sm4` · `xor` · `none` · `null`

Only `aes-128-gcm` detects tampering. The others carry a CRC32 that an on-path attacker can forge. Use `none` only when an outer layer already encrypts.

## Upgrading

- **From paqet** (`/opt/paqet`, `paqet.service`, `paqet-ctl`): the installer, or **Update** in the old `paqet-ctl`, moves the install to `/opt/paqet-plus`, `paqet-plus.service` and `paqet-plus-ctl`, keeping the config, backups and service drop-ins.
- **From paqet2:** run the installer and pick **1) Update**. It remaps removed KCP modes (`turbo` → `fast2`, `eco` → `normal`), moves `pcap:` under `network:`, converts `buffer_mb` to `sockbuf`, and drops deprecated keys.
- **Cipher default:** the default changed from `salsa20` to `aes-128-gcm`. A config without an explicit `block:` must pin `block: "salsa20"`, or both ends must upgrade together, otherwise each end drops the other's packets. The installer's update pins it for you.

## Building from source

Requires Go 1.27+, libpcap headers (`libpcap-dev` on Debian/Ubuntu) and a C compiler:

```bash
git clone https://github.com/LivingG0D/paqet-plus.git
cd paqet-plus
CGO_ENABLED=1 go build -o paqet-plus ./cmd
CGO_ENABLED=1 go test ./...
```

Release binaries are static builds made by [`.github/workflows/build-release.yml`](.github/workflows/build-release.yml) inside musl containers.

## Credits

- [hanselime/paqet](https://github.com/hanselime/paqet), the project this fork is based on
- [xtaci/kcp-go](https://github.com/xtaci/kcp-go) and [xtaci/smux](https://github.com/xtaci/smux)
- [gopacket](https://github.com/gopacket/gopacket)

## Donate

If paqet-plus is useful to you, donations help keep it maintained:

| network | address |
|---|---|
| TRON | `TR3pJMPvNdACK2oNNgjyyDTPrzCvkoGg4L` |
| USDT (BEP20) | `0xd360aEb1A32adcc3d5E1dFc043d68650C136D6D5` |
| GRAM (TON) | `UQCeLZhcJUwY-7ymm1ZESdj1so91LGJNVcAG_KzGvZUnBOgR` |

## License

[MIT](LICENSE)
