---
name: selfhost-deploy
description: Use when deploying, planning, or diagnosing self-hosted projects with Docker/Compose on macOS + Colima or Linux VPS, preserving existing services and offering explicit choices for access, database, proxy, HTTPS, and persistence. Also use to automate deploy-on-git-push via GitHub webhook and Tailscale Funnel, and to guarantee the full stack (Colima, Docker, containers, cloudflared, Tailscale) auto-recovers without manual intervention after a macOS reboot, power loss, update, or crash.
---

# selfhost-deploy

## Goal

Deploy projects repeatably and safely on self-hosted infrastructure, especially:

- Mac Apple Silicon with macOS + Colima + Docker;
- Linux VPS with Docker;
- multiple projects on the same host;
- web frontend + API + database/cache/telemetry;
- local access, private via Tailscale, or public via a domain.

The skill must **investigate before changing**. Never assume variable names, ports, database, framework, repository layout, or migrations strategy.

---

## Mandatory principles

1. **Do not take down existing services.**
   - Before creating containers, proxies, or ports, audit the host.
   - Do not run `docker compose down`, `docker-compose down`, `kill`, `pkill`, `rm`, `down -v`, or remove another project's volumes without explicit authorization.

2. **Do not invent configuration.**
   - Find out how the application actually reads configuration.
   - Look for `DATABASE_URL`, `DB_HOST`, `ConfigService`, Prisma, TypeORM, Sequelize, Redis, Vite envs, etc.

3. **Separate the three network contexts.**
   - Container → container: Docker service name, e.g. `postgres:5432`.
   - Host → container: `localhost:<published-port>`.
   - Other device → host: Tailscale IP/hostname, public IP, or domain.

4. **Databases and caches stay private by default.**
   - Do not publish PostgreSQL, Redis, Timescale, etc. on the host unless there is a real need.

5. **Persistence is mandatory for data.**
   - Databases, persistent Redis, uploads, and other data need proper volumes/bind mounts.

6. **Production does not depend on `synchronize=true`.**
   - Prefer migrations.
   - `synchronize` may only be offered as a temporary bootstrap for a brand-new, empty database, with an explicit warning.

7. **Secrets never go to Git.**
   - Use `.env.production`, a secret manager, or equivalent.
   - Ensure `.gitignore`.

8. **Vite frontend is build-time.**
   - Changing any `VITE_*` requires a rebuild.
   - Do not use `localhost` as the API URL when the frontend will be opened from another device.

9. **HTTPS and the API should preferably share an origin.**
   - Prefer `/api` behind a reverse proxy over different URLs/ports.
   - Avoid mixed content and reduce CORS.

10. **Every deployment ends with validation.**
    - containers;
    - health endpoint;
    - database;
    - frontend;
    - frontend → API;
    - persistence;
    - restart;
    - chosen external access.

11. **"Deploy works right now" is not the same as "deploy is done."**
    - Persistence, auto-start after reboot/power loss, health checks, dependency ordering, external access, and a documented rollback are part of "done," not optional extras.
    - When the host is a persistent server (e.g., a Mac acting as a server), never declare success solely from `up -d --build`; the full stack must also come back on its own after the machine restarts, with no Terminal, `nohup`, `screen`, or `tmux` involved.

---

# Operational flow

## Phase 0 — Determine the target

Identify:

- project/repository;
- destination host;
- operating system and architecture;
- whether other projects are already running;
- whether the user wants to prepare files only or actually run the deploy.

If the user has not chosen the exposure level, present:

**Option A — Local**
- access only on the host.

**Option B — Private via Tailscale HTTP**
- fast for a team/staging.

**Option C — Private via Tailscale HTTPS**
- Tailscale Serve and `*.ts.net` hostname.

**Option D — Public with domain + HTTPS**
- central reverse proxy, DNS, and TLS.

Do not silently pick an option when it changes exposure or security.

---

## Phase 1 — Host audit

Before choosing ports:

```bash
docker ps
docker ps -a
sudo lsof -nP -iTCP -sTCP:LISTEN
```

On macOS + Colima:

```bash
colima status
docker --version
docker-compose --version || docker compose version
```

If needed, identify processes:

```bash
ps -p PID -o pid,ppid,command
lsof -a -p PID -d cwd
```

Record:

- occupied ports;
- existing containers;
- already-used names;
- existing reverse proxies;
- Tailscale;
- relevant host resources.

Do not modify services just because they look conflicting. First pick another port or present the decision.

---

## Phase 2 — Project audit

Inspect before writing Compose.

Priority files:

```text
package.json
package-lock.json / pnpm-lock.yaml / yarn.lock
Dockerfile*
docker-compose*.yml
.env*
src/**/configuration*
src/**/database*
prisma/schema.prisma
vite.config.*
next.config.*
README*
```

Find out:

- frontend and framework;
- backend and framework;
- build command;
- runtime command;
- internal port;
- health endpoint;
- database;
- cache;
- queues;
- storage;
- WebSockets;
- external services;
- migrations;
- required variables.

For NestJS/TypeORM, look for:

```bash
grep -R "TypeOrmModule\|DATABASE_URL\|DB_HOST\|ConfigService\|config.get" src --include="*.ts"
```

For envs:

```bash
grep -R "process.env" src --include="*.ts"
```

For Vite:

```bash
grep -R "VITE_" . --exclude-dir=node_modules --exclude-dir=dist
```

Do not assume `DATABASE_HOST` exists if the project uses `DATABASE_URL`.

---

## Phase 3 — Choose architecture

## Option A — Simple application

```text
frontend or API
└── container
```

## Option B — Frontend + API

```text
frontend
API
```

## Option C — Common stack

```text
frontend
API
PostgreSQL/PostGIS
Redis
```

## Option D — Stack with telemetry

```text
frontend
API
PostgreSQL/PostGIS
Redis
TimescaleDB
```

## Option E — External dependencies

Keep database/cache/storage managed externally when the project already depends on them and migrating was not requested.

---

## Phase 4 — Database strategy

Present the applicable option.

## A. New database with migrations

Preferred.

```text
models/entities
→ migrations
→ database
```

Run only migrations compatible with the project.

## B. Temporary bootstrap

Only for a brand-new, empty database when the framework supports automatic creation.

TypeORM example:

```ts
synchronize: env !== 'production'
```

Procedure:

1. confirm the database is empty;
2. enable sync temporarily;
3. start the API;
4. verify tables;
5. switch back to `NODE_ENV=production` immediately;
6. recreate the API;
7. plan migrations for future changes.

Never keep this option as the production default.

## C. Existing database

- obtain dump/backup;
- restore;
- verify extensions;
- validate schema;
- run pending migrations only afterward.

Do not overwrite an existing database without confirmation.

---

## Phase 5 — Dockerfiles

## Node/NestJS backend base

Adapt to the real project:

```dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine AS runtime
WORKDIR /app
ENV NODE_ENV=production
ENV PORT=3000
COPY package*.json ./
RUN npm ci --omit=dev && npm cache clean --force
COPY --from=build /app/dist ./dist
EXPOSE 3000
CMD ["node", "dist/main.js"]
```

If the lockfile is not trustworthy, do not force `npm ci`; stabilize dependencies first.

NestJS should normally listen on:

```ts
await app.listen(process.env.PORT || 3000, '0.0.0.0');
```

## Vite frontend base

```dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine AS runtime
WORKDIR /app
RUN npm install -g serve
COPY --from=build /app/dist ./dist
EXPOSE 3000
CMD ["sh", "-c", "serve -s dist -l ${PORT:-3000}"]
```

If the project already uses Nginx/Caddy, preserve that architecture when appropriate.

---

## Phase 6 — Compose

Generate unique names per project.

Conceptual example:

```yaml
services:
  api:
    build:
      context: .
    container_name: PROJECT-api
    restart: unless-stopped
    environment:
      NODE_ENV: production
      PORT: 3000
      DATABASE_URL: postgresql://app:${POSTGRES_PASSWORD}@postgres:5432/app
    ports:
      - "HOST_API_PORT:3000"
    depends_on:
      postgres:
        condition: service_healthy

  postgres:
    image: postgis/postgis:16-3.4
    container_name: PROJECT-postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: app
    volumes:
      - PROJECT_pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d app"]
      interval: 5s
      timeout: 5s
      retries: 10

volumes:
  PROJECT_pgdata:
```

Add Redis/Timescale only when the project actually uses them.

Between containers:

```text
postgres:5432
redis:6379
timescale:5432
```

Never:

```text
localhost:5432
```

for container → container communication.

---

## Phase 7 — Ports

If there is no central reverse proxy, reserve non-conflicting ports.

Example convention:

```text
Project A  front 8080  api 3000
Project B  front 8081  api 3001
Project C  front 8082  api 3002
```

The convention is an example, not an obligation. Always check the host first.

Databases/caches get no public port by default.

---

## Phase 8 — Frontend → API

## Option A — Browser on the host itself

```env
VITE_API_URL=http://localhost:3000
```

## Option B — Another device via Tailscale

```env
VITE_API_URL=http://TAILSCALE_IP:API_PORT
```

The client device must be able to reach the Tailnet.

## Option C — Reverse proxy / same origin — preferred

```env
VITE_API_URL=/api
```

Topology:

```text
https://host/
├── /      → frontend
└── /api   → backend
```

After changing any `VITE_*`, rebuild the frontend image.

---

## Phase 9 — Exposure

## A. Local

Validate with:

```bash
curl http://localhost:PORT/health
```

## B. Tailscale HTTP

Access:

```text
http://TAILSCALE_IP:PORT
```

or the MagicDNS hostname when applicable.

## C. Tailscale HTTPS

On macOS with the Tailscale app, the CLI may be at:

```bash
/Applications/Tailscale.app/Contents/MacOS/Tailscale
```

Example:

```bash
/Applications/Tailscale.app/Contents/MacOS/Tailscale serve --bg http://localhost:8080
```

Status:

```bash
/Applications/Tailscale.app/Contents/MacOS/Tailscale serve status
```

Disable HTTPS config on port 443:

```bash
/Applications/Tailscale.app/Contents/MacOS/Tailscale serve --https=443 off
```

Inspect `serve status` before replacing an existing Serve configuration.

## D. Public domain + HTTPS

Prefer a central reverse proxy:

```text
Internet
→ 443
→ Caddy/Nginx/Traefik
→ project
```

For multiple projects:

```text
app-a.example.com → A
app-b.example.com → B
app-c.example.com → C
```

Do not expose databases/caches.

---

## Phase 10 — Secrets

Create a separate production file, for example:

```env
POSTGRES_PASSWORD=...
JWT_SECRET=...
```

Ensure:

```gitignore
.env.production
.env*.local
```

Do not print full secrets in reports or logs.

---

## Phase 11 — Deploy

Choose the command compatible with the host.

Standalone:

```bash
docker-compose \
  --env-file .env.production \
  -f docker-compose.prod.yml \
  up -d --build
```

Modern plugin:

```bash
docker compose \
  --env-file .env.production \
  -f docker-compose.prod.yml \
  up -d --build
```

Do not silently switch between both when one has already been validated on the host.

---

## Phase 12 — Mandatory validation

## Containers

```bash
docker ps
```

## Compose

```bash
docker-compose -f docker-compose.prod.yml ps
```

## Logs

```bash
docker-compose -f docker-compose.prod.yml logs --tail=100 api
```

## API

```bash
curl -i http://localhost:API_PORT/health
```

## Database

```bash
docker exec PROJECT-postgres pg_isready -U USER -d DATABASE
```

When needed:

```bash
docker exec -it PROJECT-postgres psql -U USER -d DATABASE -c '\dt'
```

## Frontend

```bash
curl -I http://localhost:FRONT_PORT
```

## Remote

Validate from the device that will actually consume the system.

If the frontend opens but login returns `Failed to fetch`, check:

1. the effective API URL in the bundle;
2. wrong `localhost`;
3. CORS;
4. mixed content;
5. firewall/Tailscale;
6. published port;
7. API health.

---

## Phase 13 — Deploy automation on git push (webhook + Tailscale Funnel)

Latest-latency option: every push to a specific branch redeploys the host automatically.

```text
git push (GitHub)
→ webhook POST https://hostname.ts.net/hooks/project
→ Tailscale Funnel (public HTTPS, *.ts.net)
→ local receiver validates HMAC + branch
→ scripts/deploy.sh (background)
→ docker compose up -d --build + frontend rebuild + healthchecks
```

Choose this path only when the user wants push-triggered deploys and accepts a public endpoint.

## 13.1 Components

- **Receiver** — small HTTP server (e.g., Node.js, zero dependencies) listening on 127.0.0.1:PORT. It:
  1. validates `X-Hub-Signature-256` (HMAC-SHA256 of the raw body with the webhook secret) using a timing-safe comparison;
  2. checks `X-GitHub-Event` is `push` and `payload.ref === 'refs/heads/main'` (or the chosen branch);
  3. replies `202` immediately and runs `deploy.sh` in background (detached).
- **deploy.sh** — `git fetch` + `git merge --ff-only` + `docker compose --env-file .env.production -f docker-compose.prod.yml up -d --build api` + frontend rebuild/recreate + healthchecks. Use a lock file (`.deploy/deploy.lock`) so two pushes never run concurrently.
- **LaunchAgent** (`~/Library/LaunchAgents/com.<org>.<project>-webhook.plist`) — keeps the receiver alive with `KeepAlive`, passing the secret via `EnvironmentVariables` (never in the repo).

Example `deploy.sh` skeleton:

```bash
#!/usr/bin/env bash
set -euo pipefail
cd "$(dirname "$0")/.."
mkdir -p .deploy
[ -e .deploy/deploy.lock ] && { echo "deploy in progress"; exit 1; }
trap 'rm -f .deploy/deploy.lock' EXIT
touch .deploy/deploy.lock

git fetch origin main
git merge --ff-only FETCH_HEAD
docker compose --env-file .env.production -f docker-compose.prod.yml up -d --build api
docker build -q -t PROJECT-front web/
docker rm -f PROJECT-front >/dev/null 2>&1 || true
docker run -d --name PROJECT-front --restart unless-stopped -p 8080:3000 PROJECT-front
for i in $(seq 1 20); do
  curl -fsS http://localhost:3000/health && break || sleep 2
done
curl -fsSI http://localhost:8080 >/dev/null
```

## 13.2 Enable Tailscale Funnel (public HTTPS)

Funnel routes public internet traffic to a local port over HTTPS on `*.ts.net`; it is the only way a GitHub webhook reaches a host on the Tailnet. Steps:

1. **Admin console** (login.tailscale.com) → **Settings → Funnel → Enable** for the target node. Enabling writes the ACL rule automatically.
2. **If flow at the ACL level is still blocked**, add the explicit Funnel rule to Access Controls:

```js
// tailscale-funnel
{
  "action": "accept",
  "src": ["*"],
  "dst": ["<NODE_IP>:443", "<NODE_IP>:<SERVICE_PORT>"],
}
```

The rule must appear as a second rule; `:443` is required for the HTTPS endpoint. Port `<SERVICE_PORT>` is now public on the internet — always protect it with the webhook secret.

3. From the CLI:

```bash
/Applications/Tailscale.app/Contents/MacOS/Tailscale funnel --bg 8787
```

`--bg` is required (without it the config is removed when the process exits). Check:

```bash
/Applications/Tailscale.app/Contents/MacOS/Tailscale funnel status
```

## 13.3 Create the GitHub webhook

```bash
SECRET=$(openssl rand -hex 32)   # store out of the repo, chmod 600
gh api repos/OWNER/REPO/hooks -X POST --input - <<'JSON'
{
  "name": "web",
  "active": true,
  "events": ["push", "ping"],
  "config": {
    "url": "https://<hostname>.ts.net/hooks/project",
    "content_type": "json",
    "secret": "PASTE_SECRET"
  }
}
JSON
```

## 13.4 Diagnostics

| Symptom | Cause / fix |
|---------|-------------|
| GitHub delivery `failed to connect to host`, but `curl` works locally | Funnel blocked at the control plane: enable it in Settings → Funnel and/or add the `tailscale-funnel` ACL rule with `:443` |
| Delivery `OK` but nothing deployed | Check receiver logs and `deploy.lock` (a stale lock is removed by the receiver on exit) |
| `TypeError [ERR_INVALID_ARG_VALUE]: argument 'stdio' is invalid` | Passing a lazily-opened `WriteStream` (fd `null`) to `spawn` stdio fails. Use `fs.openSync(LOG, 'a')` and pass the fd |
| Receiver validates branch incorrectly | GitHub sends all branches; filter by `payload.ref` |

Validation: trigger `POST /hooks/project` with a correctly signed body for `refs/heads/main` and expect `202`; then follow `.deploy/deploy.log` and `docker ps`.

---

## Phase 14 — Boot & power-loss recovery (macOS server)

Use this phase whenever the host is a persistent macOS machine (e.g., a Mac mini/M1 acting as a server) that must recover the **entire** stack automatically after a reboot, power failure, macOS update, or unexpected shutdown — with nobody opening a Terminal or typing a command.

Expected chain:

```text
macOS boots
  ↓
Colima starts automatically
  ↓
Docker Engine becomes available
  ↓
containers with a restart policy come back
  ↓
project containers (API/DB/cache/frontend) return healthy
  ↓
cloudflared comes back
  ↓
Cloudflare Tunnel connects
  ↓
public hostnames (*.yourdomain) are online

(in parallel)

macOS boots → Tailscale comes back → SSH / private admin access
```

"Deploy" and "auto-recovering deploy" are different deliverables. Do not treat this phase as optional polish.

### 14.1 Audit the current state first

Never install a new LaunchAgent/LaunchDaemon or reinstall a service before checking what already exists. Run:

```bash
brew services list
colima status
docker info
docker ps
launchctl list | grep -Ei "colima|cloudflared|tailscale|docker"
ps aux | grep '[c]loudflared'
```

Also inspect existing plists:

```bash
ls -la ~/Library/LaunchAgents ~/Library/LaunchDaemons 2>/dev/null
ls -la /Library/LaunchDaemons 2>/dev/null
grep -l -Ei "colima|cloudflared|tailscale|docker" ~/Library/LaunchAgents/*.plist /Library/LaunchDaemons/*.plist 2>/dev/null
```

Do not create a duplicate service for something that is already registered and working. If something is already correctly configured, only validate it — do not touch it.

### 14.2 Colima auto-start

Investigate the actually installed version and its supported mechanism before creating anything:

```bash
colima version
brew info colima   # Homebrew formulas sometimes print autostart caveats
colima --help | grep -i -A3 'start\|daemon\|service'
```

Prefer, in this order:

1. A native/official mechanism exposed by the installed Colima version (check its own docs/`--help`/release notes for the version in use — do not assume flags that may not exist).
2. A Homebrew-provided service definition, if `brew services` lists one for `colima`.
3. Only as a fallback, a **LaunchAgent** (not a LaunchDaemon) under `~/Library/LaunchAgents/`, because Colima's VM runs in the user's session. Use `RunAtLoad` (and `KeepAlive` only if idempotent) to run `colima start` with the correct profile/runtime flags used in production.

Critical nuance: a LaunchAgent only starts once the user logs in — not at raw boot. If nobody is expected to log in physically after a reboot, this requires **Automatic Login** to be enabled for that user (System Settings → Users & Groups → Login Options). Note that **FileVault disables automatic login**; if FileVault is enabled, decide explicitly with the user whether to trade that security control for unattended recovery — do not disable FileVault silently.

Validate:

```bash
colima status
docker info
```

### 14.3 Docker container restart policies

Audit every project's containers, not just the one being deployed:

```bash
docker inspect -f '{{.Name}} -> {{.HostConfig.RestartPolicy.Name}}' $(docker ps -aq)
```

Every production container must use `restart: unless-stopped` (see Phase 6). Apply it in the Compose file that is the actual source of truth for that project — not with an ad-hoc `docker update --restart` on a running container that will be lost on the next `up --build`.

This restart policy is inert until Docker itself is reachable, so 14.2 (Colima) and 14.3 (restart policies) only deliver recovery **together**.

### 14.4 cloudflared auto-start

If a tunnel (e.g., `cloudflared tunnel run <name>`) was installed as a service (`cloudflared service install`), verify — do not reinstall blindly:

```bash
launchctl list | grep -i cloudflared
ps aux | grep '[c]loudflared'
sudo launchctl print system/com.cloudflare.cloudflared 2>/dev/null
```

`cloudflared service install` normally registers a system LaunchDaemon, which starts at boot without requiring a user login (unlike the Colima LaunchAgent case above). If it is already registered and running, only confirm in the Cloudflare dashboard that the tunnel is `Healthy`/`Connected`. Reinstalling an already-correct service risks duplicate/conflicting configs.

### 14.5 Tailscale auto-start

Verify Tailscale reconnects without touching the Tailnet configuration:

```bash
launchctl list | grep -i tailscale
/Applications/Tailscale.app/Contents/MacOS/Tailscale status
```

The Tailscale macOS app and `tailscaled` are normally managed by the app/Homebrew installer's own launchd integration. Confirm it survives reboot; do not change ACLs, keys, or node settings as part of this phase. Tailscale remains the private/admin/SSH path; Cloudflare Tunnel remains the public path for applications — do not blur that boundary.

### 14.6 Dependency ordering and health

Do not rely on container start order alone. Combine:

- `healthcheck` in Compose (see Phase 6 example);
- `depends_on: condition: service_healthy` for direct dependents;
- application-level retry/backoff when connecting to the database/cache;
- readiness/liveness endpoints exposed by the API.

An API must not crash-loop permanently just because Postgres took a few extra seconds to become ready after a cold boot.

### 14.7 Nothing production-critical may depend on a Terminal session

After this phase, none of Colima, Docker Engine, project containers (frontend/API/DB/cache/telemetry), cloudflared, or Tailscale may depend on an open Terminal window, `nohup`, `screen`, or `tmux`. Acceptable mechanisms are: launchd (LaunchAgent/LaunchDaemon), Homebrew services, and Docker's own `restart` policy.

### 14.8 Reboot test protocol

Before physically rebooting, walk through and show the user:

1. volumes present and correct (`docker volume ls`);
2. containers and their restart policy (14.3);
3. Colima auto-start mechanism verified (14.2);
4. cloudflared auto-start mechanism verified (14.4);
5. Tailscale auto-start verified (14.5);
6. no database migration/backup/long-running job currently in progress.

**Ask for explicit confirmation before running `sudo reboot` or power-cycling the machine.**

After the machine comes back, do **not** manually run `colima start`, `docker compose up`, `cloudflared tunnel run`, or equivalents. First check whether everything recovered on its own; manually starting something masks whether auto-recovery actually works.

### 14.9 Post-reboot validation

```bash
colima status
docker info
docker ps
ps aux | grep '[c]loudflared'
```

Check every project's containers are `Up` and healthy, then validate externally:

- each public hostname over HTTPS (the actual domains configured for the tunnel);
- login and at least one authenticated API call;
- a database round-trip (not just "container is Up");
- the Cloudflare Tunnel status (`Healthy`/`Connected` in the dashboard);
- Tailscale/SSH access for private administration.

### 14.10 Power-loss recovery

Audit current power settings before changing anything:

```bash
pmset -g custom
```

Show the current state to the user first. If this Mac is dedicated to acting as a server, propose (do not silently apply):

```bash
sudo pmset -a autorestart 1
```

Also check that sleep is disabled for a headless server (`Sleep`/`disksleep`/`displaysleep` in `pmset -g custom`), since a sleeping Mac cannot serve traffic even though it is technically "on." Treat this like any other host-level change: show the before state, propose the after state, and apply only with explicit confirmation.

### 14.11 Success criteria for this phase

This phase is only complete when the full chain below works with zero manual intervention:

```text
power returns → Mac powers on → macOS boots
  → Colima → Docker
      → project containers (DB/API/frontend) per project
          → cloudflared → Cloudflare → public hostnames online
  (in parallel) → Tailscale → SSH/private admin
```

"Deploy works" and "deploy is done" are different states:

```text
Deploy
+ persistence
+ auto-start
+ health checks
+ reboot recovery
+ power-loss recovery
+ external access
+ documented rollback   = done
```

---

# Quick troubleshooting

## `ECONNREFUSED` on the database

Check whether the application uses `localhost`.

Container → Postgres should normally use:

```text
postgres:5432
```

## `relation "usuarios" does not exist`

The connection works; the schema does not exist.

Resolve with a migration, restore, or an explicitly chosen bootstrap.

## `Failed to fetch` outside the server

If the bundle contains:

```env
VITE_API_URL=http://localhost:3000
```

the browser looks for the API on the client machine.

Use a reachable address or `/api`.

## HTTPS + HTTP API

Possible mixed content.

Prefer same-origin HTTPS with `/api`.

## Vite keeps using the old URL

Rebuild:

```bash
docker build --no-cache -t PROJECT-front .
```

## Port seemingly occupied by `ssh` on macOS/Colima

Do not assume a wrongful process. Inspect first; Colima may use SSH forwarding for ports published by containers.

## Colima doesn't come back after a reboot

Check whether Colima's auto-start relies on a LaunchAgent (user-level). LaunchAgents only run after login — if the Mac does not auto-login, Colima stays stopped until someone logs in. Check `pmset`/login options and whether FileVault is blocking automatic login (14.2).

## cloudflared shows `Down`/disconnected after a reboot but the process is running

Check DNS/network readiness order — cloudflared may start before the network interface is fully up. Check `sudo launchctl print system/com.cloudflare.cloudflared` for exit/restart history rather than reinstalling the service.

## Everything comes back except the app data looks reset

The container returned, but its volume may not be the one previously containing data (e.g., an anonymous volume was recreated). Re-check `docker inspect` mounts and the Compose file's volume names (Phase 6), never assume a running container implies the same volume.

---

# Destructive operations

## Allowed only with explicit intent

```bash
docker-compose down -v
docker volume rm ...
docker system prune --volumes
rm -rf ...
```

Before any destructive operation:

- identify the project;
- identify the volume;
- explain the impact;
- confirm a backup exists when important data is involved.

Stopping containers without deleting volumes is different from deleting persistence.

---

# Deployment profiles

## Profile 1 — Internal development

```text
Docker/Compose
individual ports
optional Tailscale HTTP
volumes
```

## Profile 2 — Private staging

```text
Docker/Compose
Tailscale
private HTTPS
migrations
backup
frontend/API same origin when possible
```

## Profile 3 — Public production

```text
Docker/Compose
central reverse proxy
domain
HTTPS
migrations
automated backup
monitoring
health checks
secrets
rollback
minimal exposure
```

---

# Final checklist

Before declaring success:

```text
[ ] Real stack identified
[ ] Real dependencies identified
[ ] Real variables identified
[ ] Free ports verified
[ ] Unique container names
[ ] Unique volumes
[ ] Secrets outside Git
[ ] Database/cache not unnecessarily exposed
[ ] Migrations strategy defined
[ ] API health OK
[ ] Frontend OK
[ ] Frontend calls the correct API
[ ] Chosen remote access works
[ ] HTTPS validated when applicable
[ ] Restart policy set
[ ] Data survives restart
[ ] Existing services remained intact
[ ] Operation/rollback commands documented
[ ] Push automation (if chosen): webhook delivery OK, validate branch filter, deploy lock works
[ ] For a persistent macOS server: Colima auto-start verified (with login/FileVault implications checked)
[ ] For a persistent macOS server: cloudflared auto-start verified and tunnel Healthy/Connected
[ ] For a persistent macOS server: Tailscale auto-start verified, SSH/admin access confirmed
[ ] For a persistent macOS server: full reboot recovery tested end-to-end with no manual commands
[ ] For a persistent macOS server: power-loss recovery (`pmset autorestart`) reviewed with the user
```

---

# Skill response format

When implementing, answer step by step and present relevant decisions as options.

Example:

```text
Diagnosis
- stack:
- services:
- occupied ports:
- risks:

Exposure choice
A. Local
B. Tailscale HTTP
C. Tailscale HTTPS
D. Public domain

Selected plan
- ...

Files to create/change
- ...

Commands
- ...

Validation
- ...

Rollback
- ...
```

Do not dump dozens of commands before confirming decisions that change public exposure, an existing database, or persistent data.

---

# Golden rule

Before any change, answer these questions internally:

1. What is already running on this host?
2. How does this project really read its configuration?
3. Which data must survive?
4. Who needs to access the project, and from where?
5. Which address is valid in each context: container, host, and remote client?
6. How do I revert this change without losing data?

If any essential answer is unknown, investigate before executing.