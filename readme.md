# CIRIS Ethics Engine Enterprise (EEE) — Demo Deployment Log & Runbook

> **Purpose (StakeHolder/CEO’s ask, verbatim summary):**  
> “Stand it up to demonstrate to people that it’s possible to do alignment testing with Hendrix ethics. Host a version behind Google OAuth limited to @ciris.ai (like the manager), so SalesTeam can sell it. I’ll handle Discord policies & mods. You’ll likely need admin creds and a ciris.ai email.”

---

## What we accomplished (high level)

-- **Confirmed server access and environment** on `node0` (hostname **cirisnode0**) at ***** (Ubuntu 24.10).  
- **Enumerated and validated running containers**: `ciris-datum` (agent API), `ciris-gui` (Next.js), `ciris-sage-…`, `ciris-nginx`.  
- **Brought up a working reverse proxy (`mini-proxy`)** that cleanly routes:
  - `GET /health` → the agent **API** health
  - `GET /` → the **GUI** (Next.js)
  - `GET /api/*` → the **API**  
  Verified **200 OK** responses for GUI and **healthy** JSON for API.
- **Verified the engine API is healthy** via `/v1/system/health` with live JSON.  
- **Stood up a clean Python venv** for running the engine API locally when needed; captured the pitfalls & fixes (PEP 668, shell `source` vs `.`, etc.).  
- **Documented OAuth + DNS steps** to finalize the hosted, OAuth-gated demo.  
- **Captured full troubleshooting trail** so anyone can reproduce or recover quickly.

---

## Running containers (baseline)

```bash
docker ps
```

Representative output (multiple sessions):
```
CONTAINER ID   IMAGE                                COMMAND                  STATUS                    PORTS                          NAMES
8971e935335c   ghcr.io/cirisai/ciris-agent:latest   "python main.py --te…"   Up (healthy)              0.0.0.0:8001->8080/tcp        ciris-datum
ceb1e5b329ab   ghcr.io/cirisai/ciris-gui:latest     "docker-entrypoint.s…"   Up (unhealthy)            —                              ciris-gui
2c54a189e525   3bef54e1d40a                         "python main.py --te…"   Up (healthy)              0.0.0.0:8003->8080/tcp        ciris-sage-2wnuc8
9065b1bb7eb9   nginx:alpine                         "/docker-entrypoint.…"   Up                         —                              ciris-nginx
```

- **`ciris-datum`** exposes the agent/engine API on container port **8080**, mapped to **host:8001**.
- **`ciris-gui`** is a Next.js app listening on **3000** but runs in **host network mode** in some snapshots, causing some healthcheck headaches (see below).

---

## GUI container healthcheck (why it showed “unhealthy”)

Healthcheck definition:
```bash
docker inspect --format='{{json .Config.Healthcheck}}' ciris-gui | jq
```
Output:
```json
{
  "Test": ["CMD-SHELL","node -e \"require('http').get('http://localhost:3000', (r) => {if (r.statusCode !== 200) throw new Error()})\" || exit 1"],
  "Interval": 30000000000,
  "Timeout": 3000000000,
  "StartPeriod": 5000000000,
  "Retries": 3
}
```

Notes:
- Next.js **starts quickly** but the healthcheck expects **HTTP 200** at `/`. App sometimes serves non-200 during warmup or requires assets, which can trip the 3s timeout.  
- Inside the container we verified listener:
  ```bash
  docker exec -it ciris-gui sh -lc 'which wget || which curl; ss -lntp | grep 3000; printenv | sort'
  # shows :::3000 listening; NEXT_PUBLIC_* envs present
  ```
- This **does not block** serving traffic; it only marks the container “unhealthy.”

---

## Engine API: endpoints that matter

- `HEAD /health` → may return **404** (not implemented as simple head).  
- `GET  /v1/system/health` → **primary** health endpoint, returns JSON like:

```json
{
  "data": {
    "status": "healthy",
    "version": "1.4.4-beta",
    "services": { "communication": {"available":2,"healthy":2}, ... },
    "cognitive_state": "work",
    "timestamp": "2025-08-21T12:53:42.912560+00:00"
  }
}
```

We also saw during a local (venv) run:
```json
{
  "data": {
    "status": "healthy",
    "version": "3.0.0",
    "uptime_seconds": 71.33,
    "services": { ... }, 
    "cognitive_state": "work"
  }
}
```

---

## Local engine API (venv) — pitfalls & fixes

**Issue 1 — shell `source` not found**  
- Symptom: `-sh: source: not found` when trying `source .venv/bin/activate` under `/bin/sh`.  
- Fix: Use `bash -lc` or run venv binaries directly (`./.venv/bin/python`, `./.venv/bin/pip`).

**Issue 2 — PEP 668 “externally managed environment”**  
- Symptom: `error: externally-managed-environment` on `pip install` system-wide.  
- Fix: **Always use a venv**.

**Working sequence (we used this):**
```bash
cd ~/CIRISAgent
python3 -m venv .venv 2>/dev/null || true
./.venv/bin/pip install --upgrade pip
./.venv/bin/pip install -r requirements.txt
```

**Run the API locally (optional path):**
```bash
# Run detached in tmux so it survives SSH
tmux new -ds ciris-demo 'cd ~/CIRISAgent && ./.venv/bin/python main.py --adapter api --host 0.0.0.0 --port 8084'
sleep 2

# verify it’s listening
ss -lntp | grep 8084 || echo "8084 not listening"

# health (works if process bound)
curl -sS http://127.0.0.1:8084/v1/system/health
```

> We **also** attempted a `systemd` unit for the API, but could not enable it due to unknown `sudo` password:
> ```ini
> [Unit]
> Description=CIRIS demo API (main.py --adapter api --port 8084)
> After=network-online.target
> 
> [Service]
> User=chunky
> WorkingDirectory=/home/chunky/CIRISAgent
> Environment=PYTHONUNBUFFERED=1
> ExecStart=/home/chunky/CIRISAgent/.venv/bin/python main.py --adapter api --host 0.0.0.0 --port 8084
> Restart=always
> 
> [Install]
> WantedBy=multi-user.target
> ```
> ```
> sudo systemctl enable --now ciris-demo-api  # <- needs sudo creds
> ```

---

## Reverse proxy we stood up (the success path)

We created a **lightweight nginx** reverse proxy container named **`mini-proxy`** to unify routing for health, GUI, and API.

**Networking discovery:**
```bash
docker inspect -f '{{json .NetworkSettings.Networks}}' ciris-datum | jq 'keys'
# => ["ciris-datum-network"]

docker inspect -f '{{json .NetworkSettings.Networks}}' ciris-gui | jq 'keys'
# => ["host"]   # GUI uses host network
```

**Implications:**
- From a container on `ciris-datum-network`, you can reach the API via **`http://ciris-datum:8080`**.
- You **cannot** reach `ciris-gui` by service name (it’s on host network). Use **`host.docker.internal:3000`** and add a host-gateway mapping.

**Final working nginx config (`~/mini-proxy/default.conf`):**
```nginx
server {
  listen 8088;
  server_name _;

  # Docker’s embedded DNS in containers
  resolver 127.0.0.11 ipv6=off;

  # Plain health -> API health
  location = /health {
    proxy_pass http://ciris-datum:8080/v1/system/health;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
  }

  # API under /api/
  location /api/ {
    proxy_pass http://ciris-datum:8080/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
  }

  # GUI (Next.js on host network)
  location / {
    proxy_pass http://host.docker.internal:3000/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
  }
}
```

**Run it on the correct network with host-gateway:**
```bash
mkdir -p ~/mini-proxy && cd ~/mini-proxy

docker rm -f mini-proxy 2>/dev/null || true

docker run -d --name mini-proxy \
  --network "ciris-datum-network" \
  --add-host=host.docker.internal:host-gateway \
  -p 8088:8088 \
  -v "$PWD/default.conf:/etc/nginx/conf.d/default.conf:ro" \
  nginx:alpine
```

**Verification:**
```bash
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}' | grep mini-proxy
# mini-proxy   Up X seconds   80/tcp, 0.0.0.0:8088->8088/tcp, :::8088->8088/tcp

# Health → API
curl -sS http://127.0.0.1:8088/health
# {"data":{"status":"healthy","version":"1.4.4-beta", ...}}

# GUI header
curl -IsS http://127.0.0.1:8088/ | head -n1
# HTTP/1.1 200 OK

# API direct via /api/
curl -sS http://127.0.0.1:8088/api/v1/system/health
# same healthy JSON
```

**Earlier proxy attempts (forensics):**
- Pointing nginx to `127.0.0.1:3000` or to `ciris-gui` by name caused `502` or “host not found,” due to the mixed host/container networks.
- Using `host.docker.internal` **without** host-gateway mapping resulted in timeouts. Adding `--add-host=host.docker.internal:host-gateway` fixed it.

---

## OAuth & DNS (what we need from StakeHolder/CEO to finish)

**DNS (Cloudflare):**
- Create **A record**: `eee.ciris.ai` → `***`  
  Proxied ✅ (orange cloud)

**Google Cloud Console (OAuth):**
- Authorized domain: **ciris.ai**
- OAuth client (Web application):
  - **Redirect URI**:  
    `http://eee.ciris.ai:8088/api/auth/callback/google`  *(if serving via mini-proxy)*  
    or `http://eee.ciris.ai:8080/api/auth/callback/google` *(if GUI speaks to API on 8080 directly)*
  - Restrict to **@ciris.ai** users only (domain restriction)

**Secrets to drop into the environment (provide to me):**
```dotenv
# for GUI/auth layer (example)
GOOGLE_CLIENT_ID=...
GOOGLE_CLIENT_SECRET=...
GOOGLE_ALLOWED_DOMAIN=ciris.ai

# for API if it validates tokens (example, adjust to codebase)
OAUTH_AUDIENCE=...
OAUTH_ISSUER=https://accounts.google.com
```

**Admin bits:**
- A `@ciris.ai` email for me (Dev) so I can access manager/agents UIs.
- Which accounts should be **initial admins**?

**Legal banner / disclaimer (Discord pilot):**
- Text for login page:  
  “You are participating in a pilot for autonomous AI. You agree to allow CIRIS L3C to use data from your actions on this server to test and train ethical AI systems. You agree to hold CIRIS L3C harmless for any harms, perceived or real, that may come to you through participating in this pilot.”
  - I can add this to the login / landing as needed.

---

## Discord pilot context (status from StakeHolder/CEO)

- **1.4.5** almost ready (telemetry for public dashboard of the Discord pilot).  
- I’m **standing up hosted EEE** for SalesTeam and the sales workflow; ethicsengine rebrands as **CIRISConsult** (entity name).  
- Big hurdles: more **Discord mods** (Kelsy), and **echos with full telemetry** (StakeHolder/CEO).  
- Target: **Friday** for ready-to-open; aim to **soft open** over the weekend.

---

## Troubleshooting log (chronological-ish)

**SSH & environment checks**
```bash
sshpass -p 'yourmomBezosBitch!' ssh chunky@***
whoami && hostname -f && pwd
```
Outputs confirmed: user `chunky`, host `cirisnode0`, workdir `/home/chunky`.

**Repo inspection**
```bash
cd ~/CIRISAgent
ls -lah
# (Large file list; confirmed main.py, requirements.txt, docker/, etc.)
```

**Container health & logs**
```bash
docker ps
docker inspect ciris-gui --format='{{json .State.Health}}' | jq
docker logs --tail=80 ciris-gui
docker logs --tail=80 ciris-datum
docker logs --tail=80 ciris-nginx
```
- `ciris-gui` repeatedly: **“Health check exceeded timeout (3s)”**.
- Next.js startup lines: **Next.js 15.3.5**, local 3000, network advert.
- API logs show frequent `GET /v1/system/health` & `/v1/agent/status` returning 200.

**Inside GUI container**
```bash
docker exec -it ciris-gui sh -lc '
  which curl || which wget || (apk add --no-cache curl || apt-get update && apt-get install -y curl || true);
  wget -S -O- http://127.0.0.1:3000/ 2>&1 | head -n1
  ss -lntp || netstat -lntp
  printenv | sort
'
```
- Showed `:::3000` listening and env: `NEXT_PUBLIC_*`, `NODE_ENV=production`, etc.

**Local API run attempts**
- First attempt failed due to shell/PEP 668. We corrected to:
```bash
cd ~/CIRISAgent
python3 -m venv .venv 2>/dev/null || true
./.venv/bin/pip install --upgrade pip
./.venv/bin/pip install -r requirements.txt
tmux new -ds ciris-demo 'cd ~/CIRISAgent && ./.venv/bin/python main.py --adapter api --host 0.0.0.0 --port 8084'
sleep 2
curl -sS http://127.0.0.1:8084/v1/system/health
```
- In one run, 8084 wasn’t listening (process exited); primary path we kept was using **existing `ciris-datum`** for API and proxying to it.

**Mini-proxy attempts and fixes**
- **Attempt A (host-only)**: map `/`→3000 and `/api/`→8084; got **502** / timeouts.
- **Attempt B (container service names)**: tried `ciris-gui` upstream; failed **“host not found”** because GUI is on host network.
- **Final (working)**: attach proxy to `ciris-datum-network`, use DNS `ciris-datum:8080` for API; use `host.docker.internal:3000` for GUI with `--add-host=...:host-gateway`.

**Final verification (working responses)**
```bash
curl -sS http://127.0.0.1:8088/health
# {"data":{"status":"healthy","version":"1.4.4-beta", ...}}

curl -IsS http://127.0.0.1:8088/ | head -n1
# HTTP/1.1 200 OK

curl -sS http://127.0.0.1:8088/api/v1/system/health
# healthy JSON from API
```

---

## Operational snippets

**Follow logs**
```bash
# engine API (container)
docker logs -f --tail=200 ciris-datum

# GUI
docker logs -f --tail=200 ciris-gui

# proxy
docker logs -f --tail=200 mini-proxy
```

**Restart proxy**
```bash
docker rm -f mini-proxy
docker run -d --name mini-proxy \
  --network "ciris-datum-network" \
  --add-host=host.docker.internal:host-gateway \
  -p 8088:8088 \
  -v "$PWD/default.conf:/etc/nginx/conf.d/default.conf:ro" \
  nginx:alpine
```

**Check health quickly**
```bash
curl -sS http://127.0.0.1:8088/health | jq
curl -IsS http://127.0.0.1:8088/ | head -n1
```

---

## What’s left (to finish Stakholder/CEO ask cleanly)

1. **Cloudflare DNS:** `eee.ciris.ai` → `***` (proxied).  
2. **Google OAuth:** Authorized domain `ciris.ai`; OAuth client for web with proper **redirect URI**.  
3. **Secrets:** Share `CLIENT_ID`/`CLIENT_SECRET` and the **initial admin list**.  
4. **Lockdown:** Restrict login to **@ciris.ai**; add disclaimer text.  
5. **Optional**: fold mini-proxy routes into `ciris-nginx` main config and pick final external port (8080/443 via Cloudflare tunnel or LB).

---

## FAQ / gotchas we hit

- **Why is GUI “unhealthy”?** Healthcheck is strict on 200 within 3s; actual app is up.  
- **Why can’t nginx see `ciris-gui` by name?** It’s on **host network**; use `host.docker.internal:3000` with host-gateway mapping.  
- **Why did `/health` return 404?** Use `/v1/system/health` for JSON health.  
- **PEP 668 pip error?** Always install into a **venv**.  
- **Sudo/systemd?** Need sudo password to install services. Without it, use **tmux** or Docker.

---

## Current state (as of the latest session)

- `mini-proxy` up on **8088**, routing:  
  - `/health` → API health (healthy)  
  - `/api/*` → agent API (healthy)  
  - `/` → GUI (200 OK)  
- Engine API is healthy (via `ciris-datum`).  
- We have a clear path to **wire OAuth + DNS** and flip the public demo on for **SalesTeams**.

---

## Appendix: one-liners you can paste

```bash
# Start proxy from scratch
mkdir -p ~/mini-proxy && cd ~/mini-proxy
cat > default.conf <<'CONF'
server {
  listen 8088;
  server_name _;
  resolver 127.0.0.11 ipv6=off;

  location = /health {
    proxy_pass http://ciris-datum:8080/v1/system/health;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
  }
  location /api/ {
    proxy_pass http://ciris-datum:8080/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
  }
  location / {
    proxy_pass http://host.docker.internal:3000/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
  }
}
CONF

docker rm -f mini-proxy 2>/dev/null || true
docker run -d --name mini-proxy \
  --network "ciris-datum-network" \
  --add-host=host.docker.internal:host-gateway \
  -p 8088:8088 \
  -v "$PWD/default.conf:/etc/nginx/conf.d/default.conf:ro" \
  nginx:alpine

# Verify
curl -sS http://127.0.0.1:8088/health
curl -IsS http://127.0.0.1:8088/ | head -n1
curl -sS http://127.0.0.1:8088/api/v1/system/health
```

---

## Contact & ownership

- **Primary:** Dev (chunkywizard) — bringing up hosted EEE for Sales Team sales motion.  
- **Partner:** Stakeholder/CEO — OAuth/DNS, Discord pilot coordination, telemetry targets, policies.  
- **Notes:** This doc intentionally includes sensitive commands/creds used during setup. Store privately and rotate credentials post-launch.
