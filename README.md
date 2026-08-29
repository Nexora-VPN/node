<div align="center">
  <img src="https://avatars.githubusercontent.com/u/304456640?s=200&v=4" width="96" alt="Nexora">
  <h1>Nexora Node</h1>
  <p><strong>The data plane.</strong> Releases, container images and documentation.</p>
</div>

---

Nexora is a proxy management platform made of a **panel**
([`nexora-vpn/panel`](https://github.com/nexora-vpn/panel)) and any number of
**nodes** (this repository). A node is a stateless agent running the Nexora proxy
engine: it has no database and no configuration of its own worth backing up. The
panel pushes everything to it over HTTPS with mutual TLS, and takes it back on a
full resync whenever anything looks out of step.

This repository publishes the node: its release binaries, container images and
documentation. The source is not public.

## Installing a node

Nodes are not installed from here. Register the node in your panel and it hands
you a one-line command to run on the new server:

```bash
curl -fsSL https://PANEL/install-node.sh | bash -s -- --panel PANEL --token TOKEN
```

That command carries a **single-use install token**. Running it downloads the
node binary from your panel, exchanges the token for the panel's mTLS client
certificate, generates the node's own server certificate, installs a systemd
service and starts it. The panel pins the node's certificate on first connection
and refuses anything else afterwards.

The binaries the panel serves are the ones this repository publishes — the panel
installer stages them under `/var/opt/nexora/bin/` at install time.

For an air-gapped server, a non-systemd host, or an architecture your panel has
no staged binary for, see the manual install:

- [English](docs/en/install.md)
- [فارسی](docs/fa/install.md)
- [Русский](docs/ru/install.md)
- [中文](docs/zh/install.md)

## Firewall

A node listens for the panel on TCP **62050**. That port speaks mTLS and accepts
the panel's client certificate and nothing else, but there is no reason to leave
it open to the internet — restrict it to the panel's address:

```bash
ufw allow from PANEL_IP to any port 62050 proto tcp
```

Client traffic arrives on whatever ports the inbounds assigned to this node use.
Those are chosen in the panel, so open them there first and here second.

## Panel and node on the same server

It works — separate binaries, separate services, separate ports (2095 and
62050), no path collision — and it is **not recommended**. A node's address ends
up in every subscription link and every client config, so co-locating publishes
your panel's address to everyone you hand a config to; and a node saturating the
box takes the panel, and with it every other node's subscriptions, down too.
Install it with `--listen 127.0.0.1:62050` and add it in the panel as
`127.0.0.1` if you do it anyway — the [install guide](docs/en/install.md#panel-and-node-on-the-same-server)
has the full caveats, including the uninstall command that would delete the
panel's database.

## Docker

```bash
git clone https://github.com/nexora-vpn/node
cd node
mkdir -p certs && cp /path/to/panel_ca.pem certs/
docker compose up -d
```

Images: `ghcr.io/nexora-vpn/node` (`linux/amd64`, `linux/arm64`), published with
every tagged release.

## Releases

`nexora-node-linux-{amd64,arm64,armv5,armv6,armv7,386,s390x,riscv64}.tar.gz` —
each containing the `nexora-node` binary and the systemd unit.

The NaïveProxy outbound is the only protocol that is not pure Go; it links
Chromium's cronet, which is bundled statically for `amd64`, `arm64`, `armv7` and
`386`. The other architectures build without it and everything else works.

The panel and the node are versioned independently — a node `v1.4.0` does not
imply a panel `v1.4.0`. Any node release is driven by any panel release of the
same or newer minor version, so a fleet can be upgraded node by node.

## Support

- **Bugs and feature requests** — [open an issue here](https://github.com/nexora-vpn/node/issues).
  Panel issues belong in [`nexora-vpn/panel`](https://github.com/nexora-vpn/panel/issues).
- **Security vulnerabilities** — report them privately through
  [the organisation's security advisories](https://github.com/nexora-vpn/.github/security/advisories/new).
  Please do not open a public issue for those.

## Licence

Nexora Node is licensed per installation as part of Nexora. It needs no key of
its own — the Panel's licence caps how many nodes it will drive.
