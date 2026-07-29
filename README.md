# meowlab-media-stack

A self-hosted media server stack for Jellyfin, automated media management (Sonarr/Radarr/Bazarr),
torrent downloading via a VPN-routed container (Gluetun + qBittorrent), indexer management (Prowlarr),
request management (Jellyseerr), and secure remote access (Tailscale + Cloudflare Tunnel + Traefik).

## Stack Overview

| Service      | Purpose                                      | Access                              |
|--------------|-----------------------------------------------|--------------------------------------|
| Jellyfin     | Media server                                  | `${JELLYFIN_DOMAIN}`                |
| Jellyseerr   | Media request management                      | `${JELLYSEERR_DOMAIN}`, `:5055`     |
| Sonarr       | TV show management                            | `:8989`                             |
| Radarr       | Movie management                              | `:7878`                             |
| Bazarr       | Subtitle management                           | `:6767`                             |
| Prowlarr     | Indexer management                            | `:9696` (via Gluetun)               |
| qBittorrent  | Torrent client (VPN-routed)                   | `:8080` (via Gluetun)               |
| Byparr       | Cloudflare/anti-bot bypass for indexers       | `:8191`                             |
| Gluetun      | VPN gateway (NordVPN) for qBittorrent/Prowlarr | -                                    |
| Traefik      | Reverse proxy / routing                       | `:8000` → `:80`                     |
| Cloudflared  | Cloudflare Tunnel for external access         | -                                    |
| Tailscale    | Private mesh VPN access                       | -                                    |

Domains are not hardcoded in the compose file — they're set via `.env` (see below),
so you can safely use your own domain, a placeholder, or a local-only hostname without
editing `docker-compose.yaml` or exposing it in this public repo.

## Prerequisites

- Docker & Docker Compose installed
- An external Docker network named `traefik`:
  ```bash
  docker network create traefik
  ```
- A NordVPN account (OpenVPN service credentials)
- A Cloudflare account with a Tunnel created
- A Tailscale account with an auth key
- Media and downloads directories on the host (see Volumes below)

## Setup

1. Clone this repo and `cd` into it.

2. Create a `.env` file in the project root (this file is **git-ignored** and must never be committed):

   ```bash
   cp .env.example .env
   ```

   Then fill in `.env` with your real values:

   ```env
   # Domains used by Traefik routing (use your own domain, or a local/private
   # hostname if you don't want anything public-facing)
   JELLYFIN_DOMAIN=jellyfin.yourdomain.com
   JELLYSEERR_DOMAIN=jellyseerr.yourdomain.com

   # NordVPN service credentials (not your regular login — generate from the
   # NordVPN dashboard under Manual Setup)
   NORD_VPN_USERNAME=your_nordvpn_service_username
   NORD_VPN_PASSWORD=your_nordvpn_service_password

   # Tailscale auth key
   TS_TOKEN=your_tailscale_auth_key

   # Cloudflare Tunnel token
   CLOUDFLARE_TUNNEL_TOKEN=your_cloudflare_tunnel_token
   ```

3. Create the required host directories (adjust paths to your setup):
   ```bash
   mkdir -p /media/tv /media/movies /qbittorrent-downloads
   ```

4. Start the stack:
   ```bash
   docker compose up -d
   ```

5. Check logs if anything fails to start:
   ```bash
   docker compose logs -f <service_name>
   ```

## Volumes / Host Paths

This compose file expects the following host paths to exist:

- `/media` – Jellyfin media root
- `/media/tv`, `/media/movies` – used by Sonarr/Radarr/Bazarr
- `/qbittorrent-downloads` – shared download directory for qBittorrent, Sonarr, Radarr

Update these paths in `docker-compose.yaml` to match your host if different.

## Networking Notes

- `qbittorrent` and `prowlarr` share Gluetun's network namespace (`network_mode: service:gluetun`), so **all their traffic is routed through the VPN**. They have no independent network of their own — ports are published on the `gluetun` container.
- `traefik`, `jellyfin`, `sonarr`, `radarr`, `bazarr`, `jellyseerr`, `byparr`, and `cloudflared` sit on the shared external `traefik` network for reverse-proxy routing.
- `tailscale` uses `network_mode: host` for full mesh connectivity.

## Security Notes

- **Never commit `.env`** — it contains your domain names, VPN credentials, and Cloudflare tunnel token.
- Domains are injected via `${JELLYFIN_DOMAIN}` / `${JELLYSEERR_DOMAIN}` so your actual hostnames never appear in the public compose file or git history.
- Traefik is currently configured with only an unencrypted `web` (port 80) entrypoint — no TLS entrypoint is defined. If exposing beyond Cloudflare Tunnel/Tailscale, consider adding an HTTPS entrypoint.
- Several services (Sonarr, Radarr, Bazarr, Prowlarr, qBittorrent, Byparr) expose ports directly on the host in addition to being on the Traefik network. If you don't need direct LAN access, consider removing the `ports:` mappings and relying solely on Traefik/Tailscale.
- Double-check the repo for any real credentials, keys, or domains left in comments before pushing.

## Disabled Services

- `flaresolverr` is commented out in favor of `byparr`, which serves the same anti-bot-bypass purpose for indexers.