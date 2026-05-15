# Docker Swarm Migration Checklist

## Env File Analysis

**You were right — `config/.env` is the active one and takes precedence.**

Docker Compose loads **both** `.env` files (root and `config/.env`) when using `include:`, but `config/.env` wins on conflicts. The first `include:` entry (`config/docker-compose.yaml`) sets `config/` as the reference directory for env resolution.

Confirmed via `docker compose config`:
- `KLIPPER_IP` resolves to `192.168.0.35` (from `config/.env`), not `192.168.1.142` (from root `.env`)
- `JELLYFIN_NAME`, `JELLYFIN_PORT`, `CLOUDFLARE_DNS_API_TOKEN` resolve (only in `config/.env`)
- `N8N_NAME`, `N8N_PORT`, `FILEBROWSER_NAME`, `FILEBROWSER_PORT` also resolve (only in root `.env`)

Both files contribute variables, but `config/.env` overrides root `.env` on conflicts.

Env drift between the two files:
| Variable | Root `.env` | `config/.env` | Which Wins |
|---|---|---|---|
| `CLOUDFLARE_DNS_API_TOKEN` | missing | present | `config/.env` |
| `JELLYFIN_NAME` / `JELLYFIN_PORT` | missing | present | `config/.env` |
| `N8N_NAME` / `N8N_PORT` | present | missing | root `.env` (unique) |
| `FILEBROWSER_PORT` / `FILEBROWSER_NAME` | present | missing | root `.env` (unique) |
| `KLIPPER_IP` | `192.168.1.142` | `192.168.0.35` | **`config/.env`** (overrides) |
| `CF_API_KEY` | `key` | missing | root `.env` only, but traefik.yml expects `CLOUDFLARE_DNS_API_TOKEN` |

Also: `env` (no dot) at root is an empty file — likely accidental.

---

## Dependency & Downtime Guide

The work is grouped by **dependency order**:

| Stage | Parallelizable? | Downtime? | Description |
|---|---|---|---|
| **A: Prep** | ✅ Fully parallel | **None** | Cleanup, config fixes, documentation |
| **B: Infrastructure** | ✅ Per-node parallel | **None** | Swarm init, node setup, overlay networks |
| **C: Build** | ✅ Fully parallel | **None** | Write new compose files, create secrets/configs |
| **D: Traefik Migration** | ❌ Serial (blocking) | **Brief** (port swap) | Must be done first among deployments |
| **E: Service Migration** | ✅ Per-service parallel | **None to brief** (<30s each) | Can migrate independently after Traefik is up |
| **F: Post-Cutover** | ✅ Fully parallel | **None** | Cleanup, backup, docs |

**Key insight**: Docker supports running `docker compose` containers AND `docker stack` services simultaneously on the same node. Most services have `container_name:` and commented-out ports — the only conflict point is host port bindings. You can stagger migration per-service.

---

## Stage A: Prep Work — Zero Downtime, Fully Parallel

These have no dependencies and zero impact on running services. Do them any time in any order.

### Env File Cleanup

- [ ] Remove empty `env` file at root
- [ ] Reconcile `.env` files into a single authoritative one:
  - Decide: keep `config/.env` (currently active) or merge into root `.env`
  - `config/.env` is the active one — either keep it and delete root `.env`, or merge all vars into root and delete `config/.env`
  - Ensure `N8N_NAME`, `N8N_PORT`, `FILEBROWSER_NAME`, `FILEBROWSER_PORT` are in the authoritative file (only in root currently)
  - Ensure `JELLYFIN_NAME`, `JELLYFIN_PORT`, `CLOUDFLARE_DNS_API_TOKEN` are in the authoritative file (only in `config/.env` currently)
  - Ensure `CF_API_KEY` is dropped (traefik.yml uses `CLOUDFLARE_DNS_API_TOKEN`)
- [ ] Unify env variable naming: `CF_API_KEY` vs `CLOUDFLARE_DNS_API_TOKEN` — keep only `CLOUDFLARE_DNS_API_TOKEN`
- [ ] Resolve `KLIPPER_IP` discrepancy: `192.168.1.142` vs `192.168.0.35` — confirm correct IP

### Config & Security Fixes

- [ ] Change traefik log level from `DEBUG` to `INFO` in `config/traefik/traefik.yml`
- [ ] Disable or secure Traefik insecure API: `api.insecure: true`
- [ ] Add `**/.env` to `.gitignore` (currently .env files are tracked)
- [ ] Generate a proper grafana admin password (not `admin`/`admin`)
- [ ] Generate a proper homarr `SECRET_ENCRYPTION_KEY` (not the hardcoded one)

### Documentation

- [ ] Document the final single `.env` file location for future reference
- [ ] Document the path resolution trick (current `include:` behavior) in case you revert

---

## Stage B: Infrastructure Setup — Zero Downtime, Per-Node Parallel

Doesn't affect running compose containers. Docker supports both compose and Swarm workloads on the same node.

### Node Prerequisites

- [ ] All Swarm nodes have `nfs-common` installed:
  ```bash
  sudo apt-get install -y nfs-common
  ```
- [ ] Firewall ports open between Swarm nodes:
  - `2377/tcp` — Swarm cluster management
  - `7946/tcp` + `7946/udp` — Node communication
  - `4789/udp` — Overlay network VXLAN
- [ ] Verify NFS server (`192.168.0.20`) is reachable from ALL nodes:
  ```bash
  showmount -e 192.168.0.20
  ```
- [ ] All nodes can resolve `couch-tomatoes.me` (DNS or /etc/hosts)
- [ ] Same Docker version installed on all nodes

### Swarm Initialization

- [ ] Initialize swarm: `docker swarm init` (on manager node)
- [ ] Join worker nodes: `docker swarm join --token <token> <manager-ip>:2377`
- [ ] Label nodes for constrained placement:
  - `docker node update --label-add role=manager <manager-node>`
  - `docker node update --label-add role=worker <worker-node>`
  - `docker node update --label-add storage=nfs <nfs-node>`
  - `docker node update --label-add media=true <media-node>` (for Emby/Jellyfin)

### Overlay Network

- [ ] Create overlay network:
  - `docker network create --driver overlay --attachable media_network`
- [ ] Create encrypted overlay for sensitive traffic (optional):
  - `docker network create --driver overlay --opt encrypted media_secure`

---

## Stage C: Build — Zero Downtime, Fully Parallel

All file/secret/config work. Nothing is deployed yet.

### Secrets & Configs Management

### Critical: `env_file:` not supported in Swarm

`docker stack deploy` **silently ignores** `env_file:` directives. The traefik service currently uses:
```yaml
env_file:
  - .env
```
This must be replaced. Options:
- **Recommended**: Move all env vars to `environment:` block (explicit)
- **Alternative**: Use Docker secrets for sensitive values + `environment:` for non-sensitive
- Use `--env-file` flag with `docker stack deploy` (passed at deploy time, not in compose file)

### Secrets Migration

- [ ] Create Docker secrets for sensitive values:
  ```bash
  echo "your-token" | docker secret create cf_dns_api_token -
  echo "your-key" | docker secret create beszel_agent_key -
  echo "your-token" | docker secret create beszel_agent_token -
  echo "your-secret" | docker secret create homarr_encryption_key -
  ```
  Candidates:
  - `CF_API_KEY` / `CLOUDFLARE_DNS_API_TOKEN`
  - `BESZEL_AGENT_KEY`, `BESZEL_AGENT_TOKEN`
  - `PLEX_TOKEN`, `SONARR_API_KEY`, `RADARR_API_KEY`, `OVERSEERR_API_KEY`
  - `SECRET_ENCRYPTION_KEY` (homarr)
- [ ] Update compose files to reference secrets:
  ```yaml
  secrets:
    - cf_dns_api_token
  environment:
    - CLOUDFLARE_DNS_API_TOKEN=/run/secrets/cf_dns_api_token
  ```
- [ ] Plan secret rotation strategy for Swarm

### Docker Secrets Backup Strategy

**Important**: You cannot read back Docker secret values after creation — `docker secret inspect` shows metadata only. The only ways to recover lost secrets are:

| Method | Restore Without Original? | Notes |
|---|---|---|
| Swarm Raft backup (`/var/lib/docker/swarm`) | ✅ Yes | Full cluster recovery, not per-secret |
| Extract from running containers | ✅ Yes | `docker exec <container> cat /run/secrets/<name>` |
| Keep source `.env` file (encrypted) | ✅ Yes | Simplest — this is what you have now |

**Recommended**: Use AWS SSM Parameter Store — free (up to 10,000 parameters), KMS encryption built-in, no extra tooling.

#### AWS CLI Setup

**Install AWS CLI v2** (on the Swarm manager node):

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

**Create an IAM user** with minimal SSM-only permissions (AWS Console → IAM → Users → Create user):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ssm:PutParameter",
        "ssm:GetParameter",
        "ssm:DeleteParameter"
      ],
      "Resource": "arn:aws:ssm:*:*:parameter/couchpotatoes/*"
    }
  ]
}
```

Generate an **Access Key ID + Secret Access Key** and configure:

```bash
aws configure
# AWS Access Key ID [None]: AKIA...
# AWS Secret Access Key [None]: ...
# Default region [None]: us-east-1
# Default output format [None]: json
```

**Verify connectivity**:

```bash
aws sts get-caller-identity
aws ssm describe-parameters --query "length(Parameters)"  # 0 = working
```

- [ ] Install AWS CLI v2 on manager node
- [ ] Create IAM user with SSM permissions
- [ ] Configure `aws configure` with access keys

#### Back up secrets to SSM Parameter Store

Use `SecureString` for automatic KMS encryption:

```bash
# Store each secret individually (easier to rotate per-secret)
aws ssm put-parameter \
  --name "/couchpotatoes/cloudflare_dns_api_token" \
  --type "SecureString" \
  --value "e5EaAb00STHRkrRivmE2-NtK_m4E-DaOWjDyRt_k"

# Restore to Docker secret:
aws ssm get-parameter \
  --name "/couchpotatoes/cloudflare_dns_api_token" \
  --with-decryption \
  --query "Parameter.Value" --output text | \
  docker secret create cf_dns_api_token -
```

You can also store the entire `.env` file as a single `SecureString` parameter.

- [ ] Store all secrets in SSM Parameter Store (`SecureString`)
- [ ] Test restore by destroying a Docker secret and recreating from SSM
- [ ] Document recovery procedure

### Configs Migration (Static Files)

- [ ] Create Docker configs for static configuration files (optional — bind mounts also work):
  ```bash
  docker config create grafana-datasources config/grafana/provisioning/datasources/
  docker config create prometheus-config config/prometheus/prometheus.yml
  ```
  Candidates:
  - Traefik dynamic YAML files
  - Prometheus config
  - Grafana dashboards & datasources
  - OTel collector config
- [ ] Or keep using bind mounts (simpler for stateful configs that apps modify)

---

## Stage D: Deploy Traefik in Swarm — Brief Downtime (Blocking)

**Why first**: Unless Traefik is serving traffic in Swarm, all other Swarm services have no ingress. Deploy this first, then everything behind it can route through it. Brief downtime only for the port 80/443 swap on the Traefik host.

### Compose File Restructuring

#### Hard Blocker: `include:` NOT supported by `docker stack deploy`

This is the single biggest change. `docker stack deploy` does not process `include:`. You have two options:

#### Option A: Single Mega-Stack File
- [ ] Merge all 6 compose files into one monolithic `docker-compose.yaml`
- [ ] All paths become relative to root (NOT `config/` anymore)
- [ ] Update every volume source path: `./jellyfin` → `./config/jellyfin`, `./traefik` → `./config/traefik`, etc.
- [ ] Deploy with: `docker stack deploy -c docker-compose.yaml couchpotatoes`

#### Option B: Multiple Independent Stacks (Recommended for gradual migration)
- [ ] Split into separate stack files (e.g., `networking-stack.yaml`, `media-stack.yaml`, etc.)
- [ ] **Start with the networking stack first** (contains Traefik — must be deployed before other stacks)
- [ ] Each stack file must be self-contained — declare its own networks, volumes, secrets
- [ ] **Network sharing**: Create the overlay network externally (already in Stage B), each stack references `external: true`
- [ ] **Volume sharing**: NFS volumes must be created externally or declared in each stack
- [ ] Update all paths in each stack file to be correct relative to that file's location (NOT relative to `config/`)
- [ ] Deploy order:
  ```bash
  # 1. Networking first (Traefik)
  docker stack deploy -c networking-stack.yaml networking
  # 2. Then everything else (order doesn't matter)
  docker stack deploy -c media-stack.yaml media
  docker stack deploy -c automation-stack.yaml automation
  docker stack deploy -c management-stack.yaml management
  docker stack deploy -c monitoring-stack.yaml monitoring
  ```

### Path Resolution Overhaul

Currently, all sub-compose file paths resolve relative to `config/` due to the first-include behavior. **Every** relative path must be checked and updated for the new structure:

| Compose File | Current Path | Resolves To | New Path (root-based) |
|---|---|---|---|
| `apps/media/*` | `./jellyfin` | `config/jellyfin` | `./config/jellyfin` |
| `apps/media/*` | `./emby` | `config/emby` | `./config/emby` |
| `apps/media/*` | `./seerr` | `config/seerr` | `./config/seerr` |
| `apps/automation/*` | `./qbittorrent` | `config/qbittorrent` | `./config/qbittorrent` |
| `apps/automation/*` | `./radarr` | `config/radarr` | `./config/radarr` |
| `apps/automation/*` | `./sonarr` | `config/sonarr` | `./config/sonarr` |
| `apps/automation/*` | `./prowlarr` | `config/prowlarr` | `./config/prowlarr` |
| `apps/automation/*` | `./bazarr` | `config/bazarr` | `./config/bazarr` |
| `apps/automation/*` | `./profilarr` | `config/profilarr` | `./config/profilarr` |
| `apps/management/*` | `./homarr` | `config/homarr` | `./config/homarr` |
| `apps/management/*` | `./filebrowser` | `config/filebrowser` | `./config/filebrowser` |
| `apps/management/*` | `./portainer` | `config/portainer` | `./config/portainer` |
| `apps/management/*` | `./n8n` | `config/n8n` | `./config/n8n` |
| `apps/management/*` | `./n8n_files` | `config/n8n_files` | `./config/n8n_files` |
| `apps/monitoring/*` | `./prometheus` | `config/prometheus` | `./config/prometheus` |
| `apps/monitoring/*` | `./grafana` | `config/grafana` | `./config/grafana` |
| `apps/monitoring/*` | `./otel` | `config/otel` | `./config/otel` |
| `apps/monitoring/*` | `./beszel` | `config/beszel` | `./config/beszel` |
| `apps/networking/*` | `./traefik` | `config/traefik` | `./config/traefik` |
| `apps/networking/*` | `./acme` | `config/acme` | `./config/acme` |

### Convert Root Volumes & Network

- [ ] NFS volumes: convert from `driver: local` with NFS opts. In Swarm, you need:
  ```yaml
  volumes:
    movies:
      driver: nfs
      driver_opts:
        share: "192.168.0.20:/mnt/main/media/movies"
        nfsvers: 4
        rw: "true"
        soft: "true"
  ```
  Or create volumes externally per node:
  ```bash
  docker volume create --driver local --opt type=nfs --opt o=addr=192.168.0.20,rw,nfsvers=4,soft \
    --opt device=:/mnt/main/media/movies movies
  ```
- [ ] Add `driver: overlay` to `media_network` (or reference externally created overlay network):
  ```yaml
  networks:
    media_network:
      external: true
  ```

### Per-Service Conversions

For each service, replace Compose-only features with Swarm equivalents:

- [ ] **Remove `container_name:`** — Swarm auto-generates service names. Update any config that references old container names (e.g., Grafana datasources, Prometheus targets, healthchecks)
- [ ] **Remove `restart:`** — replace with Swarm restart policy:
  ```yaml
  deploy:
    restart_policy:
      condition: any
      delay: 5s
      max_attempts: 3
  ```
- [ ] **Remove `network_mode: host`** — incompatible with Swarm. For beszel-agent:
  - Cannot join overlay network when using host mode
  - Consider running beszel-agent as a standalone container outside Swarm on each node
  - Or use `ports:` with `mode: host` and pin to a single node
- [ ] **Remove `env_file:`** — not supported in Swarm. Move all env to `environment:` block
- [ ] **Fix healthchecks** — current healthchecks use `${VAR_NAME}.${DOMAIN}:${PORT}`. In Swarm:
  - Service DNS names are just `<service-name>`, not env-var-based
  - Update healthcheck URLs to use actual Swarm service names
  - Example: `test: ["CMD", "curl", "--fail", "http://sonarr:8989/ping"]`
- [ ] **Remove `--fail` flag** from curl healthchecks — `curl --fail` returns non-zero on HTTP 4xx/5xx, which marks the container unhealthy. Use proper healthcheck endpoints instead
- [ ] Add resource limits under `deploy.resources.limits`:
  ```yaml
  deploy:
    resources:
      limits:
        cpus: "1.0"
        memory: "512M"
  ```
- [ ] Add placement constraints:
  ```yaml
  deploy:
    placement:
      constraints:
        - node.labels.role == worker
  ```
- [ ] Add `update_config` for rolling updates:
  ```yaml
  deploy:
    update_config:
      parallelism: 1
      delay: 30s
      order: start-first
  ```
- [ ] Add `rollback_config` for safety:
  ```yaml
  deploy:
    rollback_config:
      parallelism: 1
      delay: 30s
      failure_action: rollback
  ```

### Traefik — File Provider to Docker Provider

- [ ] Update traefik config (`config/traefik/traefik.yml`) to enable Docker Swarm provider:
  ```yaml
  providers:
    docker:
      exposedByDefault: false
      swarmMode: true
      network: media_network
  ```
  Keep the file provider as well for non-Docker routes (klipper).
- [ ] Remove dynamic config files that correspond to Docker services (or convert to labels):
  - `media.traefik.yml` → inline labels on media services
  - `automation.traefik.yml` → inline labels on automation services
  - `management.traefik.yml` → inline labels on management services
  - `monitoring.traefik.yml` → inline labels on monitoring services
  - Keep `klipper.traefik.yml` (not a Docker service) and `networking.traefik.yml` (empty)
- [ ] Add Traefik labels to each service:
  ```yaml
  deploy:
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.radarr.rule=Host(`radarr.${DOMAIN}`)"
      - "traefik.http.routers.radarr.entrypoints=websecure"
      - "traefik.http.routers.radarr.tls.certresolver=letsencrypt"
      - "traefik.http.services.radarr.loadbalancer.server.port=${RADARR_PORT}"
  ```
- [ ] OR keep file provider entirely, but serve configs via Docker configs or bind mounts
- [ ] Ensure traefik is pinned to a manager node (needs Docker socket for Swarm provider)
- [ ] Ensure traefik has access to Docker socket: mount `/var/run/docker.sock`

### Networking & Ports Audit

- [ ] Audit all `ports:` mappings for Swarm compatibility:
  - Ports with `published:` / `target:` syntax are fine
  - For Swarm routing mesh, decide which ports need `mode: host` (e.g. qBittorrent `6881:6881/udp` for DHT)
  - Services exposed only through Traefik should NOT publish host ports (most are already commented out — good)
  - Traefik `80:80` and `443:443` work fine on routing mesh
- [ ] Verify overlay network `media_network` replaces all per-service bridge networks
- [ ] Service DNS in Swarm: services resolve by **service name**, not `container_name`. Examples:
  - `radarr` → resolves to `radarr` service VIP, not the old container name
  - `sonarr` → same
  - Update any hardcoded container names in configs (Prometheus scrape targets, Grafana datasources, etc.)
- [ ] Traefik `8010:8080` (dashboard) — consider restricting to internal network only
- [ ] OTel collector ports (`4317`, `4318`, `8888`, `9464`) — only needed if scraping from outside Swarm

### Traefik Deployment Sequence (brief downtime)

- [ ] **Option A (clean swap)**: Stop old compose Traefik → `docker stack deploy networking stack` → brief downtime while ports 80/443 transfer
- [ ] **Option B (side-by-side)**: Deploy Swarm Traefik on alternate ports (`8080`/`8443`) for testing, then swap to `80`/`443` in a second deploy
- [ ] Verify SSL cert generation via Let's Encrypt after deployment
- [ ] Verify Traefik dashboard is accessible

---

## Stage E: Per-Service Migration — Brief Downtime Per Service, Parallelizable

**Once Traefik is running in Swarm**, migrate individual services. Most services don't expose host ports, so they can be swapped independently with <30s downtime. Services that DO expose host ports (qBittorrent, monitoring) need the new container to wait for the port to be released.

### Migration Pattern (for each service without host ports)

```
1. docker stack deploy -c <stack>.yaml <name>   # Swarm service starts on overlay
2. docker stop <old-container-name>              # Stop compose container
3. docker compose -f <file> rm <service>         # Remove compose container (optional)
4. Verify service is healthy via Traefik route
```

### Services Without Host Port Conflicts (can be migrated individually, any order)

- [ ] **Radarr** — no host ports → stop compose `radarr`, Swarm `radarr` takes over via Traefik
- [ ] **Sonarr** — no host ports
- [ ] **Prowlarr** — no host ports
- [ ] **Bazarr** — no host ports
- [ ] **Profilarr** — no host ports
- [ ] **Emby** — no host ports
- [ ] **Jellyfin** — no host ports
- [ ] **Jellyseerr** — no host ports
- [ ] **Homarr** — no host ports
- [ ] **FileBrowser** — no host ports
- [ ] **Portainer** — no host ports (but needs Docker socket → pin to manager)
- [ ] **n8n** — no host ports
- [ ] **Beszel** — no host ports (but shared socket with agent issue)
- [ ] **OTel Collector** — has ports `4317`, `4318`, `8888`, `9464` → brief conflict if compose version still runs

### Services With Host Port Conflicts (brief interruption for port swap)

- [ ] **qBittorrent** — ports `8080`, `6881/tcp+udp`:
  1. `docker compose stop qbittorrent` (releases ports)
  2. `docker stack deploy -c automation-stack.yaml automation` (binds ports)
  3. Verify web UI accessible
- [ ] **Prometheus** — port `9090`:
  1. Brief interruption for port swap
- [ ] **Grafana** — port `3000`:
  1. Brief interruption for port swap

### Special Cases

- [ ] **Beszel Agent** — `network_mode: host` is **incompatible with Swarm**. Run as a standalone container outside Swarm on each node:
  ```bash
  docker run -d --network host --name beszel-agent \
    -v /var/run/docker.sock:/var/run/docker.sock:ro \
    henrygd/beszel-agent:latest
  ```
  Update Beszel hub to connect via TCP instead of Unix socket.
- [ ] **Klipper** — not a Docker service, just a static Traefik route. Update `klipper.traefik.yml` if needed but no Docker migration required.

### Stateful Service & Volume Considerations

This applies to each service during migration:

- [ ] **NFS volumes** (`movies`, `shows`, `downloads`) — already shared, Swarm service mounts them the same way as compose. No data migration needed.
- [ ] **Local config volumes** — Swarm service mounts the same `./config/<app>` path. The new container reads existing config. **Stop the old container before starting the new one** if the app uses a database (Emby, Jellyfin, qBittorrent, etc. — writes to config while running → corruption if two instances write simultaneously).
- [ ] **Certificates/acme.json** — Traefik's `acme.json` stays at `config/acme/acme.json`. If Traefik moves to a different node, this file must be accessible from that node (NFS or manual copy).

### Monitoring & Observability Updates

- [ ] Update Prometheus scrape targets to use Swarm service names (not container names)
- [ ] Update Grafana datasources to use Swarm-resolvable endpoints
- [ ] Verify metrics are flowing after migration
- [ ] Add Promtail/Loki for centralized logging across nodes (optional)
- [ ] Update dashboards for multi-node metrics

### Testing Checklist (per service)

- [ ] Service accessible via its Traefik subdomain (e.g., `radarr.couch-tomatoes.me`)
- [ ] Service responds to healthchecks
- [ ] Config persisted (restart service, verify changes survive)
- [ ] NFS volumes mounted correctly (`movies`, `shows`, `downloads`)
- [ ] Logs stream without errors: `docker service logs <service>`
- [ ] Rolling update works: `docker service update --update-parallelism 1 <service>`
- [ ] Failure recovery works: `docker service scale <service>=0 && docker service scale <service>=1`

---

## Stage F: Post-Cutover Cleanup

### Production Verification

- [ ] Media streaming works (Emby/Jellyfin direct play + transcoding)
- [ ] Downloads/automation pipeline functional (Radarr/Sonarr → qBittorrent → Jellyfin)
- [ ] All Traefik routes healthy with valid SSL certs
- [ ] Monitoring shows all services (Prometheus targets, Grafana dashboards)
- [ ] Backup scripts updated to reference Swarm services

### Cleanup

- [ ] Stop all remaining compose containers: `docker compose down`
- [ ] Remove old compose volumes: `docker volume prune`
- [ ] Remove unused bridge network: `docker network prune`
- [ ] Remove old sub-compose files (`apps/*/docker-compose.yaml`) if migrating to single file
- [ ] Remove old config paths if they changed during restructure

### Long-Term

- [ ] Automate secrets backup (as chosen in Stage C):
  - If S3+GPG: set up cron job to re-encrypt and upload `.env` weekly
  - If SSM: script to dump all `SecureString` params to S3 as a secondary backup
- [ ] Document full disaster recovery procedure:
  - How to restore secrets from backup
  - How to restore Swarm from Raft backup (`/var/lib/docker/swarm`)
  - How to rebuild a manager node from scratch
- [ ] Document how to add new services to the Swarm
- [ ] Set up regular backup of `docker service` definitions and Swarm raft logs
- [ ] Consider `docker swarm ca --rotate` for certificate rotation
- [ ] Set up manager node HA (3+ managers for production — currently you have 1)

---

## Appendix: Services & Swarm Compatibility Matrix

| Service | Stateful | Host Network | Ports Exposed | Swarm Strategy |
|---|---|---|---|---|
| Traefik | acme.json | no | 80, 443, 8010 | Pin to manager, NFS for acme |
| qBittorrent | config (local) | no | 8080, 6881 | Pin to worker, NFS downloads |
| Radarr | config (local) | no | no (traefik) | Pin to worker |
| Sonarr | config (local) | no | no (traefik) | Pin to worker |
| Prowlarr | config (local) | no | no (traefik) | Pin to worker |
| Bazarr | config (local) | no | no | Pin to worker |
| Profilarr | config (local) | no | no | Pin to worker |
| Emby | config (local) | no | no | Pin to media node |
| Jellyfin | config (local) | no | no | Pin to media node |
| Jellyseerr | config (local) | no | no | Pin to worker |
| Homarr | config (local) | no | no | Pin to worker |
| FileBrowser | no | no | no | Replicated |
| Portainer | config (local) | no | no | Pin to manager or replicated |
| n8n | config (local) | no | no | Pin to worker |
| Grafana | config (local) | no | no | Pin to monitoring node |
| Prometheus | config (local) | no | no | Pin to monitoring node |
| OTel Collector | no | no | 4317, 4318, etc | Replicated or global |
| Beszel | data (local) | yes | no | **Needs rework** — host network incompatible |
| Beszel Agent | data (local) | yes | no | **Needs rework** — host network incompatible |
| Klipper | no | N/A | no | Static route via Traefik (no Docker needed) |
