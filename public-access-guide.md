# Making Jellyfin Publicly Accessible

Current setup:
- Cloudflare domain (`couch-tomatoes.me`) pointed to a local IP
- Traefik reverse proxy with Let's Encrypt (DNS-01 via Cloudflare)
- Tailscale for external access to admin services
- ISP: Digi (with free DDNS)

---

## Digi DDNS — How It Works

Digi offers **free DDNS** to all residential customers. Activating it also removes you from CGNAT (Carrier-Grade NAT) and assigns a public IP to your connection.

### Activate

1. Log in at [https://www.digi.ro](https://www.digi.ro) → **Contul Meu** → **Serviciile Mele**.
2. Select your internet subscription → **DNS Dinamic**.
3. Choose a subdomain (e.g., `jellyfin`) — your DDNS hostname will be `jellyfin.go.ro`.
4. Click **Salvează** and wait ~10 minutes.
5. **Restart the router** (power cycle).

### Verify

```bash
nslookup jellyfin.go.ro
curl -s https://api.ipify.org  # should match the IP above
```

### Keep IP Updated

**Simplest — CNAME on Cloudflare.** Don't bother with update scripts. Create:

| Type | Name | Target | Proxy |
|------|------|--------|-------|
| CNAME | `jellyfin` | `jellyfin.go.ro` | Proxied (orange cloud) |

Cloudflare resolves `jellyfin.go.ro` at each request. If your IP changes, Cloudflare picks up the new resolution automatically.

---

## Recommended Setup: Dedicated Entrypoint

Only Jellyfin is public. Everything else (Sonarr, Radarr, Portainer, Homarr, etc.) stays local-only, accessible via LAN or Tailscale.

### How it works

```
User → https://jellyfin.couch-tomatoes.me (Cloudflare proxy)
  → Digi router: port 443 forwarded to server:8443
    → Traefik entrypoint "websecure-public" (:8443)
      → Jellyfin container (:8096)
```

Router never forwards to port 443 on the server — only 8443. All other Traefik routers listen on `websecure` (:443) which is unreachable from outside.

### 1. Add entrypoint to Traefik

In `config/traefik/traefik.yml`:

```yaml
entryPoints:
  web:
    address: :80
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https
  websecure:
    address: :443
  websecure-public:
    address: ":8443"
```

### 2. Move Jellyfin router to the new entrypoint

In `config/traefik/dynamic/media.traefik.yml`:

```yaml
jellyfin:
  rule: 'Host(`jellyfin.couch-tomatoes.me`)'
  entryPoints:
    - "websecure-public"        # was "websecure"
  service: jellyfin
  tls:
    certResolver: letsencrypt
```

### 3. Port forwarding on Digi router

Forward a **single** port on your router:

| External Port | Internal IP | Internal Port |
|---------------|-------------|---------------|
| 443 | `192.168.1.100` | 8443 |

Only port 443 is open on your router, and it goes straight to Traefik's `websecure-public` entrypoint. Nothing else is reachable.

**Digi router steps:**
1. Log into `192.168.1.1`.
2. Navigate to **Advanced** → **NAT** / **Port Forwarding**.
3. Add rule: External `443` → server IP → Internal `8443`.

### 4. Cloudflare DNS

| Type | Name | Target | Proxy |
|------|------|--------|-------|
| CNAME | `jellyfin` | `jellyfin.go.ro` | Proxied (orange cloud) |

Only this record exists publicly. No DNS records for sonarr, radarr, portainer, etc. — they're local/Tailscale only.

### 5. Jellyfin config

In Jellyfin admin **Dashboard → Networking**:

- **Public HTTP Port**: `8096`
- **Public HTTPS Port**: `443`
- **External domain**: `jellyfin.couch-tomatoes.me`

---

## Alternative: Cloudflare Tunnel (No Ports)

If you'd rather open zero ports, use a Cloudflare Tunnel instead.

### Run cloudflared

```yaml
services:
  cloudflared:
    image: cloudflare/cloudflared:latest
    container_name: cloudflared
    restart: unless-stopped
    networks:
      - media_network
    command: tunnel --no-autoupdate run --token YOUR_TUNNEL_TOKEN
```

### Create tunnel

1. **Cloudflare Dashboard** → **Zero Trust** → **Networks** → **Tunnels**.
2. Create a tunnel, copy the token, deploy cloudflared.
3. Add public hostname: `jellyfin.couch-tomatoes.me` → `http://jellyfin:8096`.

No ports forwarded. Only an outbound connection from cloudflared to Cloudflare.

---

## Comparison

| Approach | Ports Open | Extra Config | Latency | Isolation |
|----------|-----------|-------------|---------|-----------|
| Dedicated entrypoint | 443 → 8443 | Traefik entrypoint + router port forward | Low | Full |
| Cloudflare Tunnel | None | cloudflared container + tunnel setup | Medium | Full |

**Dedicated entrypoint** is recommended for your setup — lower latency, no dependency on Cloudflare's tunnel infrastructure, and complete isolation of admin services.

---

## Jellyfin Streaming Tweaks

In **Dashboard → Playback → Streaming**:
- Enable "Allow playback of media that is not fully downloaded"
- Set bandwidth limits based on your upload speed

### Check upload speed

```bash
curl -s https://packagecloud.io/install/repositories/ookla/speedtest-cli/script.deb.sh | sudo bash
sudo apt-get install speedtest-cli && speedtest-cli
```

1080p transcode ≈ 10–15 Mbps. 4K ≈ 25–50 Mbps.

---

## Testing

```bash
# From outside your network:
curl -s -o /dev/null -w "%{http_code}" https://jellyfin.couch-tomatoes.me
# Expect 200 or 302

nslookup jellyfin.couch-tomatoes.me
```
