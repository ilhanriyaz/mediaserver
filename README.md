# Media Server

A self-hosted media server stack built with Docker Compose. It combines the *arr suite for automated media management with Jellyfin for streaming, Pi-hole for DNS-level ad blocking, and a VPN-routed torrent client.

## Architecture

```
                     ┌─────────────┐
  You (browser/app)  │ Jellyseerr  │  Request movies & TV shows
         │           └──────┬──────┘
         │                  │ notifies
         ▼                  ▼
  ┌─────────────┐   ┌──────────────┐   ┌──────────────┐
  │   Jellyfin  │   │    Radarr    │   │    Sonarr    │
  │  (streams)  │   │   (movies)   │   │  (TV shows)  │
  └──────┬──────┘   └──────┬───────┘   └──────┬───────┘
         │                 │ searches          │ searches
         │           ┌─────▼───────────────────▼──────┐
         │           │           Prowlarr              │
         │           │      (indexer aggregator)       │
         │           └─────────────┬──────────────────┘
         │                         │ sends to
         │           ┌─────────────▼──────────────────┐
         │           │     qBittorrent (via Gluetun)   │
         │           │       VPN-routed downloads      │
         │           └─────────────┬──────────────────┘
         │                         │ downloads to
         │           ┌─────────────▼──────────────────┐
         └───────────►     /mnt/mediadrive/media       │
                     │   (shared media storage)        │
                     └────────────────────────────────┘

  DNS:  Pi-hole blocks ads network-wide (port 53)
  SSL:  Nginx Proxy Manager terminates HTTPS (ports 80/443)
  Subs: Bazarr fetches subtitles for Radarr/Sonarr libraries
```

## Services

| Service | Image | Port(s) | Purpose |
|---|---|---|---|
| Jellyfin | `jellyfin/jellyfin` | 8096 | Media streaming server |
| Jellyseerr | `fallenbagel/jellyseerr` | 5055 | User-facing request interface |
| Radarr | `linuxserver/radarr` | 7878 | Movie collection manager |
| Sonarr | `linuxserver/sonarr` | 8989 | TV show collection manager |
| Bazarr | `linuxserver/bazarr` | 6767 | Subtitle downloader |
| Prowlarr | `linuxserver/prowlarr` | 9696 | Indexer / tracker aggregator |
| qBittorrent | `linuxserver/qbittorrent` | 8080 | Torrent client (via VPN) |
| Gluetun | `qmcgaw/gluetun` | — | WireGuard VPN container |
| Nginx Proxy Manager | `jc21/nginx-proxy-manager` | 80, 81, 443 | Reverse proxy + SSL |
| Pi-hole | `pihole/pihole` | 53, 8053 | DNS server + ad blocker |

## Prerequisites

- Docker and Docker Compose v2
- An external drive (or directory) mounted at `/mnt/mediadrive` with the structure below:
  ```
  /mnt/mediadrive/
  ├── media/
  │   ├── movies/
  │   └── tv/
  └── torrents/
  ```
- A [ProtonVPN](https://protonvpn.com) account with a WireGuard config (free tier works)

## Quick Start

```bash
# 1. Clone the repo
git clone https://github.com/ilhanriyaz/mediaserver.git
cd mediaserver

# 2. Create your local secrets file
cp .env.example .env
# Edit .env and fill in WIREGUARD_PRIVATE_KEY and PIHOLE_PASSWORD

# 3. Start the stack
docker compose up -d

# 4. Check everything is running
docker compose ps
```

> **Note:** On first run, Docker will create the `config/`, `data/`, `etc-pihole/`, and `letsencrypt/` directories automatically as each service initialises.

## Configuration

### VPN (Gluetun)

qBittorrent runs inside Gluetun's network namespace — all torrent traffic is routed through the VPN. If the VPN drops, torrent traffic stops (kill-switch by design). To get your WireGuard key:

1. Log in to [protonvpn.com](https://protonvpn.com)
2. Go to **Downloads → WireGuard configuration**
3. Generate a config and copy the `PrivateKey` value into your `.env`

### Nginx Proxy Manager

Access the admin UI at `http://<host-ip>:81` on first run. Default credentials: `admin@example.com` / `changeme`. Set up proxy hosts to expose Jellyfin, Jellyseerr, and other services over HTTPS.

### Pi-hole

Pi-hole runs as a DNS server on port 53. Point your router (or individual devices) to `<host-ip>` as the DNS server to get network-wide ad blocking. The admin UI is at `http://<host-ip>:8053/admin`.

### Domain Routing with Tailscale + Pi-hole + Nginx Proxy Manager

Use this pattern when you want friendly domain links like `jellyfin.yourdomain.com` and `jellyseerr.yourdomain.com` to work from devices connected through Tailscale.

How resolution works:

1. **Tailscale DNS** sends your domain lookups to your Pi-hole nameserver.
2. **Pi-hole Local DNS** returns your media host IP (usually the host running this stack).
3. **Nginx Proxy Manager** receives the HTTPS request and routes it to the correct container (`jellyfin` or `jellyseerr`).

#### Step-by-step setup

1. **Get the IP to use for DNS answers**
   - Use the Tailscale IP of the host running this stack (preferred for remote Tailscale clients).
   - Example: `100.x.y.z`

2. **Create local DNS records in Pi-hole**
   - Open Pi-hole admin: `http://<host-ip>:8053/admin`
   - Go to **Local DNS → DNS Records**
   - Add:
     - `jellyfin.yourdomain.com` → `100.x.y.z`
     - `jellyseerr.yourdomain.com` → `100.x.y.z`

3. **Configure proxy hosts in Nginx Proxy Manager**
   - Open NPM admin: `http://<host-ip>:81`
   - Add proxy host for `jellyfin.yourdomain.com` forwarding to `jellyfin:8096`
   - Add proxy host for `jellyseerr.yourdomain.com` forwarding to `jellyseerr:5055`
   - Request/enable SSL certificates and force HTTPS

4. **Set Pi-hole as nameserver in Tailscale**
   - Open Tailscale admin console → **DNS**
   - Add your Pi-hole host as a nameserver (use the host Tailscale IP)
   - Under split DNS / restricted nameserver domains, add `yourdomain.com`
   - Enable DNS settings for your tailnet clients

5. **(Optional) Make Pi-hole your global Tailscale resolver**
   - If you want all DNS queries from Tailscale devices to use Pi-hole (not only `yourdomain.com`), set Pi-hole as the global nameserver in the same DNS page.

After this, any Tailscale-connected client resolving `*.yourdomain.com` will use Tailscale DNS → Pi-hole → Nginx Proxy Manager, and be routed to Jellyfin/Jellyseerr correctly.

### Connecting the *arr Services

After the stack is up, configure each service in this order:

1. **Prowlarr** → add your indexers
2. **Radarr / Sonarr** → add Prowlarr as an indexer, add qBittorrent as a download client (host: `localhost`, port: `8080`)
3. **Bazarr** → connect to Radarr and Sonarr via their API keys
4. **Jellyseerr** → connect to Jellyfin, then to Radarr and Sonarr

## Directory Structure

```
mediaserver/
├── docker-compose.yaml   # Service definitions
├── .env.example          # Secret template — copy to .env
├── .gitignore
└── README.md

# Created at runtime (gitignored):
├── config/               # Per-service config and databases
├── data/                 # Nginx Proxy Manager data
├── etc-pihole/           # Pi-hole config and databases
└── letsencrypt/          # Let's Encrypt certificates
```

## Port Reference

| Port | Protocol | Service |
|---|---|---|
| 53 | TCP/UDP | Pi-hole DNS |
| 80 | TCP | Nginx (HTTP) |
| 81 | TCP | Nginx Proxy Manager admin |
| 443 | TCP | Nginx (HTTPS) |
| 5055 | TCP | Jellyseerr |
| 6767 | TCP | Bazarr |
| 6881 | TCP/UDP | qBittorrent (via Gluetun) |
| 7878 | TCP | Radarr |
| 8053 | TCP | Pi-hole web UI |
| 8080 | TCP | qBittorrent web UI (via Gluetun) |
| 8096 | TCP | Jellyfin |
| 8989 | TCP | Sonarr |
| 9696 | TCP | Prowlarr |
