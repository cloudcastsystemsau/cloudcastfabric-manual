# CloudCast Fabric — Installation Guide

CloudCast Fabric is two programs:

- the **controller** — the control plane. Agents connect to it on **:8600**;
  operators use the dashboard on **:8601** (http) / **:8443** (https). One per
  fabric. It carries no audio — it only coordinates the network.
- the **agent** (`ccf-agent`) — runs at every site: studios, transmitter sites,
  cloud instances. Agents build encrypted WireGuard tunnels to each other and
  carry audio **directly, edge to edge**.

This guide takes you from nothing to a running fabric with one controller and
one or more agents. Everything you need is a single download per machine.

> **Downloads:** all installers live at
> **https://cloudcast-fabric-dist.s3.ap-southeast-2.amazonaws.com/fabric/latest/**
> with a checksum file (`SHA256SUMS`) beside them. Pinned versions are under
> `…/fabric/<version>/`. The download page lists every file:
> **http://cloudcast-fabric-dist.s3-website-ap-southeast-2.amazonaws.com/**

---

## 1. Ports and hosts

| Port | Who connects | Purpose |
|---|---|---|
| **8600** / tcp | agents → controller | control channel (routes, discovery, peering) |
| **8601** / tcp | operators → controller | dashboard (http) |
| **8443** / tcp | operators / enrolling agents → controller | dashboard + `/enrol` (https) |
| **51820** / udp | agents → controller | WireGuard (the controller is the fabric's reachable endpoint) |
| **7777** / udp | agent ↔ agent | fabric media (direct, over WireGuard) |

The controller needs a **public, stable address** for :51820 and :8443 so agents
anywhere can reach it. Agents need only **outbound** UDP — they dial the
controller and each other; nothing inbound has to be opened for a NAT'd agent.

---

## 2. Install the controller

The controller ships as a **self-contained** build — the .NET runtime is bundled,
so the host needs nothing preinstalled. Pick the tarball for the host's CPU:
`linux-arm64` (AWS Graviton and other ARM) or `linux-x64` (Intel/AMD).

```bash
# on the controller host (example: arm64 / Graviton)
BASE=https://cloudcast-fabric-dist.s3.ap-southeast-2.amazonaws.com/fabric/latest
curl -fsSLO $BASE/ccf-controller-latest-linux-arm64.tar.gz
curl -fsSLO $BASE/SHA256SUMS && sha256sum -c --ignore-missing SHA256SUMS

tar xzf ccf-controller-latest-linux-arm64.tar.gz
cd ccf-controller-*-linux-arm64
sudo ./install-controller.sh --admin-password 'pick-a-strong-one'
```

That installs to `/opt/ccf-controller`, writes `/etc/ccf/controller.env`,
enables a `ccf-controller` systemd service, and starts it. When it comes up:

```
ccf-controller is up.
  dashboard : http://<this-host>:8601   (https :8443)
  agents    : point --controller at <this-host>:8600
```

Open the firewall for **8600, 8601, 8443** (tcp) and **51820** (udp), then browse
`http://<host>:8601` and log in with the admin password.

### Controller options

`install-controller.sh` keeps any value you set on a later run unless you pass
the flag again. The ones that matter for a real deployment:

| Flag | What it sets |
|---|---|
| `--admin-password <pw>` | dashboard login (written 0600, never on the command line) |
| `--wg-endpoint <ip:port>` | the **public** WireGuard endpoint agents dial (e.g. `203.0.113.9:51820`) |
| `--hub <fabric-ip>` | a forwarding node that carries the `10.99.0.0/24` catch-all so agents reach every site; leave unset to keep agents control-plane + direct-peer only |
| `--relay off` | disable the automatic hub relay fallback (see §6) |

For enrolment to work the controller also needs its **public HTTPS base** set, so
tokens embed a reachable `/enrol` URL. If you installed behind a name like
`fabric.example.com:8443`, add to `/etc/ccf/controller.env`:

```ini
CCF_PUBLIC_HTTPS=https://fabric.example.com:8443
CCF_WG_ENDPOINT=<public-ip>:51820
```

then `sudo systemctl restart ccf-controller`. The `/enrol` endpoint must present
a **trusted** TLS certificate — an enrolling agent rejects a self-signed cert.
Front it with a real certificate (e.g. Caddy or a load balancer) for the public
name.

---

## 3. Install an agent and join it to the fabric

Joining is two steps: **mint a one-time token** on the controller, then **enrol**
the agent with it. The token is self-describing — it carries the controller's URL,
so you never type an address twice.

### 3a. Mint a join token (on the controller)

From the dashboard, or over the loopback (which counts as admin on the controller
host):

```bash
curl -s -X POST http://127.0.0.1:8601/api/join-tokens \
     -H 'Content-Type: application/json' \
     -d '{"name":"studio-1","ttlSeconds":7200}'
# -> {"token":"ccf_join_…","note":"copy this now — it is never shown again"}
```

### 3b. Install and enrol (on the agent host)

The Linux agent is one static binary plus an installer. Download, enrol, then
install the service for the mode this site runs.

```bash
BASE=https://cloudcast-fabric-dist.s3.ap-southeast-2.amazonaws.com/fabric/latest
curl -fsSLO $BASE/ccf-latest-linux-x86_64.tar.gz
curl -fsSLO $BASE/SHA256SUMS && sha256sum -c --ignore-missing SHA256SUMS
tar xzf ccf-latest-linux-x86_64.tar.gz
cd ccf-*-linux-x86_64

# put the binary in place, then enrol (writes /etc/wireguard/wg0.conf, brings up the tunnel)
sudo install -m755 ccf-agent /usr/local/bin/ccf-agent
sudo ccf-agent enrol --token 'ccf_join_…' --name studio-1
```

A good enrol prints `admitted as 10.99.0.N` and `brought up wg0`. Confirm the
tunnel and reach the controller over the fabric:

```bash
sudo wg show wg0            # a recent handshake to the controller
ping -c2 10.99.0.4          # the controller on the fabric
```

Then install the agent service for this site's **mode** (§4). For a site that
bridges a real AoIP LAN:

```bash
sudo ./install.sh --mode bridge --lan-iface eth0 \
                  --controller 10.99.0.4:8600 --lan-jitter-ms 40 --apply-peers
```

The installer pulls its own dependencies (`wireguard-tools`, `libasound2`), writes
`/etc/ccf/agent.env`, and starts a `ccf-agent` service. The agent now appears on
the controller's **Fabric** page.

> `--apply-peers` lets the controller wire **direct** peer-to-peer tunnels between
> this agent and the sites it exchanges audio with (with automatic NAT-traversal
> recovery — see §6). Without it the agent stays hub-and-spoke. Recommended on.

---

## 4. Agent modes

Pass one `--mode` to `install.sh`:

| Mode | Use it when | Key flags |
|---|---|---|
| **bridge** | the site has a real AoIP LAN (Axia/Livewire, AES67 hardware) to attach to | `--lan-iface`, `--lan-ip`, `--lan-jitter-ms` |
| **mesh** | the instance itself sends/receives multicast on a virtual NIC (`ccf0`) | `--addr <cidr>` |
| **router** | a fan-out relay for large subscriber counts | `--members a:7777,b:7777` |

A bridge learns what its LAN wants either from a downstream switch's IGMP
snooping table (`--switch-snoop-cmd <cmd>`) or a static `--lan-subs`. It
discovers local sources automatically over **SAP** and **Livewire**, and reports
them to the controller so an operator can publish them to other sites.

### Generating test sources

Any agent host can run standalone AES67 source generators — useful for a soak
test or a new region before real hardware arrives:

```bash
ccf-agent --mode aes67-source --iface <ip> --group 239.120.1.10 --port 5004 \
          --format l24 --channels 2 --ptime 1 --sap \
          --name "Test Tone 440" --pace-clock auto --tone 440 --gain-db -20
```

Run the generators on a **different IP** from the bridge's `--lan-ip` (a second
address on the NIC is fine) — a bridge ignores SAP that originates from its own
address, exactly as it ignores its own re-announcements. On a real plant the
sources are separate devices, so this is automatic.

---

## 5. Verify

- **Dashboard → Fabric** lists every agent, its fabric IP, discovered source
  count, and a peer matrix showing which pairs can exchange audio directly
  (green), are relaying while a punch recovers (amber **R**), or have no path
  (red).
- On any agent, `curl -s localhost:9464/metrics | grep ccf_` shows fabric and
  per-peer counters, including `ccf_wg_peer_handshake_age_seconds`.

---

## 6. Direct peering and NAT traversal

Agents carry audio **directly** to each other. The controller coordinates the
connection (it observes each agent's public address and tells the pair to punch
simultaneously), but WireGuard itself has no NAT traversal, so occasionally a
punch fails. With `--apply-peers` the agent recovers on its own:

1. audio **falls back through the controller** immediately, so it never drops;
2. in the background the agent rotates its port to get a fresh path and keeps
   retrying the direct tunnel;
3. the moment the direct tunnel is up, audio moves back off the controller.

The relay uses controller bandwidth, so it is a fallback, not the norm — set
`--relay off` on the controller to disable it, or leave it on (default) for
resilience. The **Fabric** page shows which pairs are direct vs relaying.

---

## 7. Upgrading and removing

**Upgrade** — download the new tarball and re-run the same installer; settings in
`/etc/ccf` are kept:

```bash
# agent
sudo install -m755 ccf-agent /usr/local/bin/ccf-agent && sudo systemctl restart ccf-agent
# controller
cd ccf-controller-*-linux-arm64 && sudo ./install-controller.sh
```

**Remove** — units and program files go, `/etc/ccf` is kept so you can reinstall:

```bash
sudo ./install.sh --uninstall              # agent
sudo ./install-controller.sh --uninstall   # controller
```

---

## 8. Platforms and troubleshooting

**Platforms.** Controller: `linux-arm64` and `linux-x64` (self-contained). Agent:
`linux-x86_64`. A Windows agent MSI is available for plant PCs; ask your
CloudCast contact. The controller runs happily on a small instance (2 vCPU / 2 GB);
an agent runs on anything from a NUC to a cloud VM.

**Time / PTP.** For sample-accurate AES67 media, an agent should run on a clock
disciplined to the plant's PTP grandmaster (a bridge follows the LAN's PHC
automatically). On AWS, enable the Nitro hardware clock and point chrony at it —
this needs a **reboot** to take effect. Without it you get NTP-grade time, which
is fine for a data overlay and for testing, not for a production media clock.

**Common issues.**

| Symptom | Cause / fix |
|---|---|
| `libasound.so.2: cannot open shared object file` | missing ALSA runtime; `install.sh` installs it, or `apt install libasound2t64` (24.04) / `libasound2` |
| enrol: *request to …/enrol failed* / cert rejected | the controller's `:8443` must present a **trusted** certificate; front it with a real cert |
| agent shows up but carries no audio | check the site actually **subscribes** (a device IGMP-joins the group) and the source is **published** to it on the dashboard |
| a pair is stuck **relaying** (amber R) | the direct punch is still recovering; it re-converges on its own. Confirm both ends run `--apply-peers` |
| `no ptp dev` on a cloud host | the Nitro PHC is not enabled — see *Time / PTP* above |

---

*CloudCast Fabric is a Cloudcast Systems product.*
