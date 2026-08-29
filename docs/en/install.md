# Installing a Nexora node

A node is the data plane: a stateless agent that terminates client traffic and
does what the panel tells it. It has no database, no admin interface and no
settings file worth backing up — its only state is a pair of certificates. That
is deliberate: a lost node is re-registered in the panel and comes back
identical, and nothing on it needs to be migrated or restored.

You almost never install a node from this page. **The panel installs nodes for
you**, and the one-line command it gives you is the supported path. This page
covers that path in detail, and then the manual one for the cases the panel
cannot reach.

## Requirements

- A Linux server with systemd. amd64, arm64, armv5/v6/v7, 386, s390x and riscv64
  are all published.
- Root access.
- Outbound access to the panel, and TCP **62050** reachable *from the panel*.
- Whatever ports the inbounds you assign to this node will listen on.

A node does not need a licence key of its own. The panel's licence caps how many
nodes it will drive.

## The normal install

In the panel, add the node and copy the command it shows:

```bash
curl -fsSL https://PANEL/install-node.sh | bash -s -- --panel PANEL --token TOKEN
```

Run it as root on the new server. In under a minute it:

1. Detects the architecture and downloads the node binary **from your panel**,
   not from the internet.
2. Exchanges `TOKEN` for the panel's mTLS client certificate. The token is
   single-use and expires; the panel consumes it on this request and it never
   works again.
3. Generates the node's own server certificate and key.
4. Writes `/opt/nexora-node/config.json` and a systemd unit, then starts the
   service.

The panel connects, records the node's certificate fingerprint on that first
connection, and from then on accepts only that certificate — trust on first use.
The node in turn accepts only the panel's client certificate. Nothing else on the
internet can talk to port 62050 in a way that gets past the TLS handshake.

Options you can add to the command:

| Flag | Effect |
| --- | --- |
| `--listen ADDR` | bind the control API somewhere other than `0.0.0.0:62050` |
| `--binary-url URL` | take the binary from somewhere other than the panel |

If the panel has no binary staged for this server's architecture, the install
fails at step 1 with `node binary not provisioned`. Stage it on the **panel**
host and re-run — do not fall back to a manual install just for that:

```bash
# on the panel server
curl -fsSL -o /tmp/node.tar.gz \
  https://github.com/nexora-vpn/node/releases/latest/download/nexora-node-linux-armv7.tar.gz
tar -C /tmp -xzf /tmp/node.tar.gz
install -m 0755 /tmp/nexora-node/nexora-node /var/opt/nexora/bin/nexora-node-linux-armv7
```

## Firewall

Port 62050 speaks mTLS and rejects everyone but the panel, but there is no reason
to advertise it:

```bash
ufw allow from PANEL_IP to any port 62050 proto tcp
```

Client-facing ports are a separate matter. They are whatever the inbounds
assigned to this node use, so they are chosen in the panel — open them here after
you have assigned the template, not before.

## Panel and node on the same server

**Not recommended.** It works, and nothing in Nexora forbids it — but read why
you probably do not want it before you do it.

The two do not collide. The panel listens on 2095 and the node on 62050; they
install into `/opt/nexora-panel` and `/opt/nexora-node`, run as separate systemd
services, and share only the parent of their state directories —
`/var/opt/nexora/nexora.db` and `/var/opt/nexora/bin/` are the panel's,
`/var/opt/nexora/certs/` is the node's.

To do it, run the panel's one-line install command on the panel server itself,
and bind the control API to loopback, since the panel is already there:

```bash
curl -fsSL https://PANEL/install-node.sh | bash -s -- \
  --panel PANEL --token TOKEN --listen 127.0.0.1:62050
```

Then add the node in the panel with the address `127.0.0.1`. Port 62050 never
leaves the machine, so skip the `ufw` rule above entirely.

In Docker, the panel repository ships this arrangement as a ready stack:
[`docker/panel-and-node`](https://github.com/nexora-vpn/panel/tree/main/docker/panel-and-node).

### Why it is not recommended

- **It publishes the panel's address.** A node's IP goes into every subscription
  link and every client config you hand out. Co-locating means every user — and
  anyone who collects those configs — learns where your panel lives. If that
  address is later filtered or blacklisted for carrying proxy traffic, you lose
  the panel and every subscription URL with it, not just one node.
- **One machine, one fate.** Client traffic is bursty and unbounded: it eats
  CPU, memory, file descriptors and bandwidth. A node under load takes the panel
  down with it — and a panel that is down takes the subscriptions of *every*
  other node's users down too.
- **It fuses the disposable with the irreplaceable.** A node is meant to be
  thrown away and re-registered; the panel's database is the one thing in the
  whole fleet worth backing up. On a shared server, reinstalling the node means
  working on top of that database.
- **The uninstall is a footgun.** `rm -rf /var/opt/nexora` in the uninstall
  section below deletes the panel's database. On a shared server remove only
  `/opt/nexora-node` and `/var/opt/nexora/certs`.
- **It still costs a node.** A co-located node counts against your licence's
  node cap exactly like any other.

It is a reasonable choice for a lab, a demo, or a small single-server deployment
where you accept that the panel and the proxy share one address and one fate.
For anything you would be upset to lose, give the panel its own server.

## Manual install

For an air-gapped server, a host without systemd, or a panel you would rather not
have reach out from, the same four steps done by hand.

**1. Install the binary.**

```bash
ARCH=amd64   # or arm64, armv7, armv6, armv5, 386, s390x, riscv64
curl -fsSL -o /tmp/node.tar.gz \
  https://github.com/nexora-vpn/node/releases/latest/download/nexora-node-linux-${ARCH}.tar.gz
tar -C /tmp -xzf /tmp/node.tar.gz
mkdir -p /opt/nexora-node /var/opt/nexora/certs
install -m 0755 /tmp/nexora-node/nexora-node /opt/nexora-node/nexora-node
```

**2. Get the panel's client certificate.** Copy it from the node install page in
the panel and save it as `/var/opt/nexora/certs/panel_ca.pem`. This is a public
certificate, not a secret — a node that has it still cannot do anything to the
panel.

**3. Generate the node's own certificate.**

```bash
/opt/nexora-node/nexora-node gencerts -dir /var/opt/nexora/certs
```

**4. Configure and start.**

```bash
cat > /opt/nexora-node/config.json <<'JSON'
{
  "listen": "0.0.0.0:62050",
  "cert_file": "/var/opt/nexora/certs/ssl_cert.pem",
  "key_file": "/var/opt/nexora/certs/ssl_key.pem",
  "client_ca_file": "/var/opt/nexora/certs/panel_ca.pem"
}
JSON

cp /tmp/nexora-node/nexora-node.service /etc/systemd/system/
systemctl daemon-reload
systemctl enable --now nexora-node
```

Then add the node in the panel with this server's address and port 62050. The
panel pins the certificate on its first connection exactly as it would have after
the scripted install.

## Docker

```bash
git clone https://github.com/nexora-vpn/node
cd node
mkdir -p certs
cp /path/to/panel_ca.pem certs/
docker compose up -d
```

The compose file uses host networking on purpose. A node terminates client
traffic on whatever ports its inbounds use, and those are chosen in the panel
afterwards — with bridged networking every new inbound would mean editing the
compose file and recreating the container.

## Updating

Nodes are updated from the panel, which pushes the new binary and restarts the
service. Nothing needs to be run on the node itself.

By hand, replace the binary and restart:

```bash
systemctl stop nexora-node
install -m 0755 /tmp/nexora-node/nexora-node /opt/nexora-node/nexora-node
systemctl start nexora-node
```

The certificates are untouched by an update, so the panel's pin still matches and
the node comes back without being re-registered.

In Docker: `docker compose pull && docker compose up -d`.

## Uninstall

```bash
systemctl disable --now nexora-node
rm -f /etc/systemd/system/nexora-node.service
systemctl daemon-reload
rm -rf /opt/nexora-node /var/opt/nexora
```

If the panel is installed on this same server, do **not** remove
`/var/opt/nexora` — that is where its database lives. Remove
`/var/opt/nexora/certs` instead.

Delete the node in the panel too, or it will keep being counted against your
licence and keep showing as offline.

## Where things live

| Path | What |
| --- | --- |
| `/opt/nexora-node/nexora-node` | the binary |
| `/opt/nexora-node/config.json` | listen address and certificate paths — nothing else |
| `/var/opt/nexora/certs/ssl_cert.pem` | the node's own server certificate |
| `/var/opt/nexora/certs/ssl_key.pem` | its private key |
| `/var/opt/nexora/certs/panel_ca.pem` | the panel's client certificate, which is the only one accepted |
| `/etc/systemd/system/nexora-node.service` | the service unit |

Everything else — inbounds, outbounds, endpoints, routing, users — is pushed by
the panel and held in memory. There is nothing else on disk to back up.

## Troubleshooting

**The install command fails with `node binary not provisioned`.** The panel has
no binary staged for this architecture. Stage it on the panel host as shown
above, then re-run the command with a fresh token.

**The install command fails with `invalid token` or `token already used`.** Node
install tokens are single-use and time-limited. Generate a new one in the panel
and copy the whole command again.

**The node runs but the panel shows it offline.** Something between the two is
dropping the connection. Check, in order:

```bash
systemctl status nexora-node          # on the node
journalctl -u nexora-node -n 50       # on the node
nc -vz NODE_IP 62050                  # from the panel server
```

A firewall rule that does not include the panel's address is the usual cause; a
cloud provider security group is the second.

**The node was reinstalled and the panel refuses it.** A reinstall generates a
new server certificate, and the panel is still pinning the old one. Delete the
node in the panel and add it again — the pin is taken fresh on the next first
connection.
