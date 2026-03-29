# Zabbix Monitoring — Distributed Infrastructure over WireGuard Mesh

A practical guide to setting up Zabbix monitoring across a distributed infrastructure using a WireGuard-based mesh network (Tailscale, NetBird, or similar). Covers Zabbix Server + Proxy + Agent deployment, all communication isolated inside the mesh — nothing exposed to the public internet.

---

## Architecture Overview

```
[Zabbix Agent] vps-1        ──┐
[Zabbix Agent] vps-2         ├──► [Zabbix Proxy] relay-node ──► [Zabbix Server] main-vps
[Zabbix Agent] relay-node    │       (mesh hub)                    (PostgreSQL)
[Zabbix Agent] home-server  ──┘
                                                                      │
                                                                [Web UI] :8888
                                                                (mesh only)
                                                                      │
                                                                [Telegram Bot]
                                                                (alerts)
```

**Key design decisions:**
- Zabbix Server on a reliable VPS with enough RAM for PostgreSQL (2+ GB recommended)
- Zabbix Proxy on the mesh hub node — it already has connectivity to all peers, natural aggregation point
- All ports bound to mesh interface IPs only — nothing reachable from the public internet
- Active mode everywhere — agents and proxy initiate outbound connections, no inbound ports needed
- Web UI accessible only over mesh — WireGuard traffic is already encrypted, no need for HTTPS

---

## Components

| Component | Recommended Host | Port | Visibility |
|---|---|---|---|
| Zabbix Server | Main VPS | `10051` | Mesh IP only |
| Zabbix Web UI | Main VPS | `8888` | Mesh IP only |
| Zabbix Proxy | Mesh hub node | `10051` | Mesh IP only |
| PostgreSQL | Main VPS | `5432` | Docker internal only |

---

## Main VPS — Zabbix Server

### File structure

```
/opt/docker/zabbix/
├── docker-compose.yml
└── .env
```

### `docker-compose.yml`

```yaml
services:
  zabbix-postgres:
    image: postgres:16-alpine
    container_name: zabbix-postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: zabbix
      POSTGRES_USER: zabbix
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - zabbix-postgres-data:/var/lib/postgresql/data
    networks:
      - zabbix-internal

  zabbix-server:
    image: zabbix/zabbix-server-pgsql:ubuntu-7.0-latest
    container_name: zabbix-server
    restart: unless-stopped
    environment:
      DB_SERVER_HOST: zabbix-postgres
      POSTGRES_DB: zabbix
      POSTGRES_USER: zabbix
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      ZBX_STARTPOLLERS: 5
      ZBX_CACHESIZE: 128M
      ZBX_HISTORYCACHESIZE: 64M
      ZBX_TRENDCACHESIZE: 32M
      ZBX_VALUECACHESIZE: 64M
    ports:
      - '<MESH_IP>:10051:10051'    # bind to mesh interface only
    depends_on:
      - zabbix-postgres
    networks:
      - zabbix-internal
      - zabbix-external

  zabbix-web:
    image: zabbix/zabbix-web-nginx-pgsql:ubuntu-7.0-latest
    container_name: zabbix-web
    restart: unless-stopped
    environment:
      ZBX_SERVER_HOST: zabbix-server
      DB_SERVER_HOST: zabbix-postgres
      POSTGRES_DB: zabbix
      POSTGRES_USER: zabbix
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      PHP_TZ: Europe/Moscow      # adjust to your timezone
    ports:
      - '<MESH_IP>:8888:8080'    # bind to mesh interface only
    depends_on:
      - zabbix-server
      - zabbix-postgres
    networks:
      - zabbix-internal
      - zabbix-external

volumes:
  zabbix-postgres-data:

networks:
  zabbix-internal:
    internal: true    # postgres and server are fully isolated
  zabbix-external:
    driver: bridge    # allows web and server to bind ports on host interfaces
```

### `.env`

```bash
POSTGRES_PASSWORD=your_strong_password_here
```

**Why two networks:**
`zabbix-internal` with `internal: true` — postgres and server have no outbound internet access. `zabbix-external` — allows containers to bind ports on host interfaces including the mesh interface. Without it, Docker cannot map a container port to a WireGuard/mesh IP.

### Start

```bash
docker compose up -d
docker compose logs -f zabbix-server
# wait for: server #0 started [main process]
```

Web UI: `http://<MESH_IP>:8888`
Default credentials: `Admin` / `zabbix` — change immediately after first login.

---

## Proxy Node — Zabbix Proxy

### File structure

```
/opt/docker/zabbix-proxy/
└── docker-compose.yml
```

### `docker-compose.yml`

```yaml
services:
  zabbix-proxy:
    image: zabbix/zabbix-proxy-sqlite3:ubuntu-7.0-latest
    container_name: zabbix-proxy
    restart: unless-stopped
    environment:
      ZBX_PROXYMODE: 0                           # 0 = active mode
      ZBX_SERVER_HOST: <SERVER_MESH_IP>          # Zabbix Server mesh IP
      ZBX_SERVER_PORT: 10051
      ZBX_HOSTNAME: my-proxy                     # must match name in Web UI
      ZBX_CONFIGFREQUENCY: 60                    # fetch config every 60s
      ZBX_DATASENDERFREQUENCY: 5                 # send data every 5s
    ports:
      - '<PROXY_MESH_IP>:10051:10051'            # bind to mesh interface only
    volumes:
      - zabbix-proxy-data:/var/lib/zabbix

volumes:
  zabbix-proxy-data:
```

**Why sqlite3 image:**
Proxy only needs to buffer data temporarily before forwarding to Server. SQLite is sufficient — no need for a separate PostgreSQL container, which saves ~300MB RAM on the proxy node.

**ZBX_PROXYMODE: 0** = active mode. Proxy initiates the connection to Server — Server never needs to reach out to the proxy node.

### Register proxy in Web UI

After starting the container, go to **Administration → Proxies → Create proxy**:
- **Proxy name:** must exactly match `ZBX_HOSTNAME`
- **Proxy mode:** Active
- **Address:** leave empty (active mode — proxy connects to server)

Proxy logs should show `received configuration data from server` within 60 seconds.

---

## Agents — All Nodes

### Installation (Ubuntu 24.04)

```bash
wget https://repo.zabbix.com/zabbix/7.0/release/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest+ubuntu24.04_all.deb
dpkg -i zabbix-release_latest+ubuntu24.04_all.deb
apt update && apt install -y zabbix-agent2
```

### Installation (Ubuntu 22.04)

Same as above, replace `ubuntu24.04` with `ubuntu22.04` in the filename.

### Installation (Ubuntu 25.04)

zabbix-agent2 7.0.x is available in the standard Ubuntu 25.04 universe repository:

```bash
apt install -y zabbix-agent2
```

### Configuration `/etc/zabbix/zabbix_agent2.conf`

```bash
ServerActive=<PROXY_MESH_IP>:10051   # agent pushes data to proxy
Server=<PROXY_MESH_IP>               # ACL — only accept connections from this IP
Hostname=my-node-name                # must match hostname in Zabbix Web UI
```

```bash
systemctl enable zabbix-agent2
systemctl restart zabbix-agent2
```

### Register host in Web UI

**Data collection → Hosts → Create host:**
- **Host name:** must exactly match `Hostname` in agent config
- **Templates:** `Linux by Zabbix agent active`
- **Host groups:** `Linux servers`
- **Interfaces:** leave empty (active mode — agent connects to proxy, not the other way)
- **Monitored by proxy:** select your proxy

Agent logs should show `active check configuration update from` within 60 seconds.

**Why active mode:**
Agent initiates the connection to Proxy — no inbound port `10050` needs to be open on any node. Data is pushed, not pulled.

**Full data flow:**
```
agent → Proxy → Server → PostgreSQL → Web UI
```

---

## Web Monitoring

Create a dedicated host `web-monitoring` with `Monitored by proxy: <your-proxy>` — this way the proxy node performs the HTTP checks independently from where the services actually run. If the server hosting the service goes down, the proxy will detect it.

### Create web scenario

**Data collection → Hosts → web-monitoring → Web scenarios → Create web scenario:**

| Field | Value |
|---|---|
| Name | service name (e.g. `Vaultwarden`) |
| Update interval | `1m` |
| Steps → URL | `https://your-domain.com` |
| Steps → Required status codes | `200` |
| Follow redirects | ✓ |

### Create trigger for each scenario

Web scenarios in Zabbix 7.0 do not auto-create triggers. Add manually:

**Data collection → Hosts → web-monitoring → Triggers → Create trigger:**

- **Name:** `web-monitoring: Vaultwarden is unavailable`
- **Expression:** `last(/web-monitoring/web.test.fail[Vaultwarden])<>0`
- **Severity:** High

Repeat for each scenario — use **Clone** to speed it up.

---

## Telegram Alerts

### 1. Configure media type

**Administration → Media types → Telegram:**
- Parameter `api_token`: your bot token from @BotFather

Make sure the bot token is set and the media type is Enabled.

> Before setting up, send `/start` to your bot in Telegram — bots cannot initiate conversations with users who haven't started a chat first.

### 2. Add media to user

**Users → Users → Admin → Media → Add:**
- **Type:** Telegram
- **Send to:** your chat ID (get it by sending `/getid` to @myidbot, or via `https://api.telegram.org/botTOKEN/getUpdates`)
- **Use if severity:** select all desired severity levels

### 3. Create action

**Alerts → Actions → Trigger actions → Create action:**
- **Name:** `Telegram alerts`
- **Status:** Enabled
- **Operations → Add:**
  - Operation type: Send message
  - Send to users: Admin
  - Send only to: Telegram

> Existing problems (open before the action was created) will not generate notifications — only new PROBLEM events trigger actions. To test, temporarily raise a trigger threshold to resolve an existing problem, then lower it back to fire a new event.

---

## Housekeeping

**Administration → Housekeeping** — controls how long Zabbix retains data in PostgreSQL.

| Parameter | Recommended value | Notes |
|---|---|---|
| Trigger data storage | `90d` | Event history for 3 months |
| History — Override item history | ✓ enabled | Forces all items to use this value |
| History — Data storage period | `30d` | Metric history for 1 month |
| Trends — Override item trend period | ✓ enabled | Forces all items to use this value |
| Trends — Data storage period | `365d` | Trends are lightweight, keep for a year |

Enabling Override is important — without it each item uses the retention period defined in the template (often 90 days for history), which can cause PostgreSQL to grow unexpectedly.

---

## How It Works Under the Hood

### Active vs Passive mode

Two agent modes — the key difference is who initiates the connection.

**Passive mode:**
```
Server/Proxy ──► connect to agent :10050 ──► pull metrics
```
Server connects to the agent and pulls data. Requires port `10050` open and reachable on every monitored node.

**Active mode:**
```
Agent ──► connect to Proxy :10051 ──► push metrics
```
Agent connects to Proxy, fetches the list of checks, executes them, and pushes results. No inbound port needed on the node.

Active mode is the only practical choice in a distributed setup: nodes may be behind NAT, firewalls, or in different networks. An agent can always make an outbound TCP connection — but reaching the agent from outside is not always possible.

### Data flow and the role of Proxy

```
agent (any node)
    │
    └──► Proxy (mesh hub)         ← all agents push here
              │
              │  buffers data locally (SQLite)
              │  if Server is unreachable — stores data, flushes on reconnect
              │
              └──► Server (main VPS)
                        │
                        └──► PostgreSQL
                                  │
                                  └──► Web UI
```

**Why route all agents through Proxy even if some nodes can reach Server directly:**
- Uniformity — all agents have identical config regardless of location
- Server has no knowledge of agent topology — adding a new node requires no changes on Server
- Buffering works for all nodes: if Server restarts or is briefly unreachable, Proxy holds the data

### Why Docker won't bind ports to a mesh interface

`internal: true` on a Docker network means the container has no route to the host's external interfaces. Docker physically cannot expose a port from such a container on a host IP — there is no network path.

```
Problem (didn't work):
container ──► internal network (internal: true) ──► ✗ no path to mesh interface

Solution (works):
container ──► zabbix-internal (internal: true)   ← postgres stays isolated
         └──► zabbix-external (bridge)            ← path to host interfaces exists
                    │
                    └──► bind port on mesh IP ✓
```

postgres stays in `zabbix-internal` only — it never touches the outside world.

### Zabbix reacts to events, not states

This is critical to understand when setting up alerts.

**Event** = the moment a trigger transitions between states:
- OK → PROBLEM = new event, action fires, notification sent
- PROBLEM persists without change = no new events, no notifications

If a trigger was already in PROBLEM state when you created an action — no notification will be sent. Actions only subscribe to events from the moment of creation, not to current state.

**How to test an alert after setting up actions:**
Change a threshold macro so the trigger resolves (PROBLEM → OK), then change it back (OK → PROBLEM) — two new events, both generate notifications.

### Why web-monitoring runs on a separate host via Proxy

If a web scenario lives on the same host as the service, they go down together:

```
home-server goes down:
├── agent goes silent   → alert "host unavailable"
└── web scenario stops  → no HTTP check, no service alert
```

An independent proxy node as the observation point separates two different failure scenarios:

```
home-server alive, service crashed:
├── agent works         → host Available
└── Proxy makes HTTP    → gets 502 → alert "service is unavailable"

home-server goes down:
├── agent silent        → alert "host unavailable"
└── Proxy makes HTTP    → timeout → alert "service is unavailable"
```

Two different alerts with different meanings — one says "look at the server", the other says "look at the container".

### Override in Housekeeping

Without Override, each item retains data according to its template definition — different items have different retention periods, making PostgreSQL growth unpredictable.

```
Without Override:
item A (Linux template)  → 90 days
item B (Web template)    → 30 days
item C (custom)          → 365 days
→ PostgreSQL grows unpredictably

With Override (30d History):
all items                → 30 days, no exceptions
→ PostgreSQL grows predictably
```

Trends use a separate, longer retention (e.g. 365d) because they are much lighter than raw history — aggregated min/max/avg per hour instead of every data point. A year of trends gives you seasonality patterns useful for capacity planning.

---

## Common Issues

**Container port not binding to mesh IP:**
Docker cannot bind a port to a mesh/WireGuard interface (`wt0`, `tailscale0`, etc.) if the container only has `internal: true` networks. Add a regular bridge network (`zabbix-external`) and connect the container to both networks. The `internal: true` network handles isolation; the bridge network allows Docker to expose ports on any host interface.

**`proxy "my-proxy" not found` in proxy logs:**
The proxy is running but not registered in the Web UI. Go to Administration → Proxies → Create proxy and add it with the exact same name as `ZBX_HOSTNAME`.

**`host [my-node] not found` in agent logs:**
The agent is running and can reach the proxy, but the host isn't registered in the Web UI. Go to Data collection → Hosts → Create host.

**Graphs empty after adding hosts:**
Normal behavior — graphs need data to render. Wait 10-15 minutes for the first data points to arrive. The default graph time range is 1 hour; if agents were just connected, switch to "Last 15 minutes" to see data sooner.

**`vfs.file.contents["/sys/class/net/wt0/speed"]` errors in agent log:**
Virtual network interfaces (WireGuard, Tailscale, etc.) don't expose a `speed` value in `/sys/class/net/*/speed`. This is a kernel limitation, not a bug. Safe to ignore — it only produces log noise, no monitoring impact. Disable the specific item in Web UI if you want to suppress it.

**Telegram test returns `Incorrect "event_source" parameter`:**
This is expected — the Media type Test button sends raw macro strings that don't get resolved in test mode. Ignore this error; real alerts will work correctly.

---

## Notes

- Zabbix 7.0 LTS — supported until 2029
- Agent version on a node can be slightly behind Server version — Zabbix maintains backward compatibility within major versions
- `ZBX_CONFIGFREQUENCY: 60` — proxy fetches updated host configuration from Server every 60 seconds. After adding a new host in Web UI, wait up to 60s for the agent to start receiving checks
- Web scenarios do not auto-create triggers in Zabbix 7.0 — add them manually per scenario
- HTTPS is unnecessary for Web UI if accessed exclusively over a WireGuard mesh — traffic is already encrypted at the transport layer