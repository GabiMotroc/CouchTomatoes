# 🍅 CouchTomatoes

CouchTomatoes is a comprehensive, self-hosted media streaming and automation suite. A playful blend of "Couch Potato" and "Rotten Tomatoes," this project provides a modular, containerized environment for media management, download automation, system monitoring, and home server administration.

## 🏗️ Architecture

The project uses a modular `docker-compose` architecture, splitting services into logical categories for better maintainability and organization. All services are routed through **Traefik** as a reverse proxy with automatic SSL certificate management via Let's Encrypt (DNS-01 challenge).

### Project Structure
- `apps/`: Contains category-specific Docker Compose files.
  - `automation/`: Download and library management (Sonarr, Radarr, etc.).
  - `management/`: Server administration and dashboards (Portainer, Homarr).
  - `media/`: Streaming servers and request management (Plex, Emby).
  - `monitoring/`: System metrics and visualization (Grafana, Prometheus).
  - `networking/`: Core infrastructure (Traefik).
- `config/`: Centralized persistent storage for service configurations.
- `.env`: Environment-specific variables (ports, paths, API tokens).

---

## 🚀 Services

### 🎬 Media & Streaming
- **Plex / Emby**: High-performance media streaming servers.
- **Jellyseerr**: A user-friendly request management system for your media library.

### 🤖 Automation
- **qBittorrent**: Robust torrent client for automated downloads.
- **Radarr**: Automatic movie downloader and library manager.
- **Sonarr**: Automatic TV show downloader and library manager.
- **Prowlarr**: Indexer manager for torrent and Usenet trackers.
- **Bazarr**: Automated subtitle downloader for your media library.

### 🛠️ Management & Tools
- **Homarr**: A sleek, customizable dashboard for all your services.
- **FileBrowser**: Web-based file manager for easy access to your media and configs.
- **Portainer**: Graphical interface for managing Docker containers.
- **n8n**: Workflow automation tool to connect your services.

### 📊 Monitoring
- **Grafana**: Beautiful dashboards for visualizing system and service metrics.
- **Prometheus**: Time-series database for collecting metrics.
- **Dashdot**: Simple, modern server dashboard for real-time hardware stats.
- **Otel Collector**: OpenTelemetry collector for system-wide observability.

---

## 🛠️ Setup & Installation

### 1. Prerequisites
- **Docker** and **Docker Compose** installed.
- A registered domain (e.g., `couch-tomatoes.me`).
- Cloudflare account (if using the default DNS-01 challenge for SSL).

---

### ⚠️ One-Time Node Setup (NFS)

Media volumes (`movies`, `shows`, `downloads`) are mounted from an NFS share. Each new node added to the cluster **must** have the NFS client utilities installed before running `docker compose up`, otherwise Docker will fail to mount the volumes with a `no such file or directory` error.

Run the following **once per node**:

```bash
sudo apt-get install -y nfs-common
```

Also ensure `NFS_SERVER` is set in `.env`:

```bash
NFS_SERVER=192.168.0.20
```

The NFS server exports the media library from `192.168.0.20:/mnt/main/media` (verified via `showmount -e 192.168.0.20`).

#### NFS Folder Permissions

All media folders on the NFS share **must be owned by UID 1000 / GID 1000** — the user the containers run as (`PUID=1000`, `PGID=1000`). If ownership is wrong, containers will get `Permission denied` errors even though the NFS mount itself works.

**When creating a new folder on TrueNAS Scale:**

1. Create the dataset/folder in TrueNAS as usual.
2. Fix ownership — choose one of the two methods below:

   **Option A — TrueNAS Scale UI (no shell required):**
   - Go to **Datasets** in the left sidebar.
   - Find your dataset (e.g., `main/media/<new-folder>`).
   - Click the three-dot menu → **Edit Permissions**.
   - Under **Owner**, set **User** to `1000` and **Group** to `1000` (type the UID directly if the user doesn't exist by name).
   - Check **Apply permissions recursively**.
   - Click **Save Access Control List**.

   **Option B — TrueNAS Shell / SSH:**
   ```bash
   chown -R 1000:1000 /mnt/main/media/<new-folder>
   chmod -R 775 /mnt/main/media/<new-folder>
   ```
3. Add the new volume to `docker-compose.yaml` (root file) following the same NFS pattern as `movies`, `shows`, or `downloads`.
4. Mount it in the relevant service's `volumes:` block.
5. Run `docker compose up -d` — Docker will mount the new NFS volume automatically.

**Symptoms of a permissions problem:**
- Container logs show `UnauthorizedAccessException` or `Permission denied` on a path inside the volume.
- Write test: `docker exec -u abc <container_name> touch /mountpoint/test && echo "OK"` — if this fails, it's a permissions issue on the NAS side.

**Quick fix — choose one method:**

- **TrueNAS Scale UI**: Go to **Datasets** → find the affected dataset → three-dot menu → **Edit Permissions** → set Owner User/Group to `1000` → check **Apply permissions recursively** → **Save Access Control List**.
- **TrueNAS Shell / SSH**:
  ```bash
  chown -R 1000:1000 /mnt/main/media/<affected-folder>
  ```

Then restart the affected container:
```bash
docker restart <container_name>
```

---

### 2. Configuration
Copy the `.env` file (or create one) and update the following key variables:

```bash
# Media & Download paths
MOVIES_DIR=/path/to/your/movies
SHOWS_DIR=/path/to/your/shows
DOWNLOADS_DIR=/path/to/your/downloads

# User/Group ID (usually 1000:1000)
USER_ID=1000
GROUP_ID=1000
TIMEZONE=Etc/UTC

# Networking
DOMAIN=yourdomain.com
CLOUDFLARE_DNS_API_TOKEN=your_cloudflare_token
```

### 3. Deployment
Deploy the entire stack using the main `docker-compose.yaml` file at the root:

```bash
docker compose up -d
```

---

### 📊 Monitoring & Observability

#### Service Configuration Permissions (Grafana & OpenTelemetry)

Monitoring services often require specific permissions to read their configuration and provisioning files. If a container is stuck in a restart loop with `permission denied` errors in the logs, it's likely because the container user cannot read the files mounted from the host.

Run the following on the host to allow read access to other users (since these services run as their own non-root users):

**For Grafana (UID 472):**
```bash
# Set configuration files to 644 (read for all)
chmod -R 644 config/grafana/provisioning/datasources/*.yml
chmod -R 644 config/grafana/provisioning/dashboards/*.json

# Set directories to 755 (execute for all)
find config/grafana -type d -exec chmod 755 {} +
```

**For OpenTelemetry Collector:**
```bash
# Set the main config file to 644
chmod 644 config/otel/otel-collector-config.yaml
```

---

## 🌐 Networking & Access

All services are accessible via subdomains on your configured `DOMAIN`. Traefik handles the routing and SSL termination automatically.

- **Dashboard**: `homarr.yourdomain.com`
- **Plex**: `plex.yourdomain.com`
- **Sonarr/Radarr**: `sonarr.yourdomain.com`, `radarr.yourdomain.com`
- **...and so on.**

### SSL Certificates
The system is configured to use **Let's Encrypt** with the **Cloudflare DNS-01 challenge**. This allows for wildcard certificates and internal-only services to have valid HTTPS without exposing ports 80/443 to the entire internet (if configured accordingly).

---

## 📁 Volume Structure
- **Config Data**: All persistent app data is stored in the `config/` directory, mapped to `./config/<app_name>`.
- **Media Assets**: Movies, TV Shows, and Downloads are mapped via global volumes defined in the root `docker-compose.yaml`.

---

*Enjoy your personal media empire!* 🍿
