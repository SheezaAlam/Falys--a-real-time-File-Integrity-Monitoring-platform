# Dynamic IP Discovery (Server ⇄ Agent)

## The problem

Both the central server and the agent endpoints are on DHCP, so their IPs
change over time. Previously, the agent's `config.json` had the server's
IP hardcoded — once that IP changed, the agent had no way to find the
server again, and connectivity was lost completely until someone manually
updated `config.json` with the new address.

## The solution

Since the server and all agents share the same enterprise LAN (same
broadcast domain/subnet), this uses a small UDP broadcast discovery
protocol — no extra infrastructure (no DNS server, no dependency on
internet access) required.

### What happens on `server.py` startup
1. Resolves its own current LAN IP (`discovery.get_local_ip()`).
2. Immediately broadcasts a `falys_announce` UDP packet on the LAN so
   any agent already running picks up the current IP right away.
3. Starts a background listener that answers any agent's `falys_discover`
   broadcast directly with its current IP.
4. Starts a background watchdog that re-checks its own IP every 30s; if
   it ever changes while running, it re-broadcasts immediately.

### What happens on `agent.py` startup
1. Resolves its own current LAN IP the same way (used in every event's
   `ip` field, instead of the old `socket.gethostbyname(socket.gethostname())`
   call, which is unreliable — commonly returns `127.0.1.1` or the wrong
   interface on multi-homed machines).
2. Checks whether the server address currently in `config.json` is still
   reachable **and genuinely the FALYS server** (via a `/health` check —
   see "Authentication" below). If yes, nothing changes.
3. If not, broadcasts a `falys_discover` request on the LAN and waits for
   an authenticated reply. If found, updates `config.json` immediately so
   the next restart also starts from the right address.
4. If nothing answers, the agent keeps queuing events locally (existing
   offline-queue behavior) and retries discovery every 20s in the
   background until the server is found.
5. For the rest of the session, a background listener watches for the
   server announcing itself again (step 2 of the server's startup, or its
   periodic re-announce if its IP changes later) and updates immediately
   — **so a live agent recovers from a server IP change without needing
   to be restarted.**

## Authentication ("is this really the server?")

Broadcast discovery on its own just tells you *something* answered on a
port — not that it's genuinely the FALYS server and not some other
device. Both sides are configured with the same shared secret
(`DISCOVERY_TOKEN`, in `server/config.py` and every agent's
`config.json`). Every discovery message carries a one-way hash of that
secret (`sha256("FALYS_DISCOVERY_V1:" + token)`) — never the secret
itself — so:
- A device that doesn't know the token gets ignored, even if it responds
  on the same port.
- Packet-sniffing the LAN doesn't reveal the secret.

**You must change `DISCOVERY_TOKEN` to your own value before deploying,
and keep it identical across `server/config.py` and every agent's
`config.json`** — this is what's shipped as a placeholder default.

## Verified behavior (tested in this environment)

- Agent configured with a deliberately wrong/unreachable server IP →
  detects the failure → broadcasts → finds the real server → updates and
  persists `config.json` → all within the same run, no restart.
- A running agent, already correctly configured, receives a live
  `falys_announce` for a genuinely different IP → updates immediately,
  no restart.
- A discovery request carrying the wrong token is correctly ignored.

## Known limitations

- **Same broadcast domain only.** This will not cross routed
  subnets/VLANs without a broadcast relay (e.g. a UDP relay agent, or
  switching to multicast DNS with proper router support). If your
  "enterprise network" spans multiple subnets, each subnet needs its own
  relay, or agents in other subnets need the server's hostname/IP
  configured through another means (e.g. internal DNS).
- **Firewalls**: UDP port `50999` must be allowed (inbound on the server
  for discovery requests, and on agents for the live-announce listener)
  through host firewalls (Windows Firewall / iptables / firewalld).
- **Discovery timing**: the initial discovery attempt on agent startup
  can take a few seconds (multiple broadcast attempts with timeouts) —
  this is intentional, to tolerate a slow/busy network, but means the
  agent won't send its very first events instantly if the server was
  unreachable at that exact moment. They're queued locally and flushed
  once the server is found.
