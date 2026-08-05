# Raspberry Pi Home Server Stack

This repository manages the Docker Compose stack for the Raspberry Pi home server.

## 1. Host System
- **Hostname:** `rocket.int.yt`
- **Internal IP:** `192.168.1.31`
- **OS:** Debian Trixie (Testing)
- **User:** `pi`

## 2. Directory Structure
All container data and configurations are stored in persistent host paths:
- `/opt/containers/`  
  Main configuration root for all services.
- `/opt/containers/audiobookshelf`  
  Audiobookshelf config and metadata.
- `/opt/containers/plex`  
  Plex Media Server config.
- `/opt/containers/mariadb`  
  MariaDB database files for WordPress and other services.
- `/opt/containers/homeassistant`
  Home Assistant configuration files.
- `/data/media`  
  Shared media library area.
- `/data/downloads`  
  Shared downloads area.
- `/home/pi/projects/sites`  
  Website content root.

## 3. Samba Access
The `/data` directory is shared via Samba for easy network access.
- **Share Name:** `data`
- **User:** `pi`
- **Force User/Group:** `pi` (to prevent permission issues)
- **Create Mask:** `0664` / **Directory Mask:** `0775`

## 4. Compose Project Model
The current direction is **one main Compose project**, not one Compose file per service.
- Compose file location: `/opt/containers/docker-compose.yml`
- Bring services up from `/opt/containers`
- Containers in this file share the default Compose network
- NPM should proxy to backend services by **service name** when possible

Examples:
- qBittorrent backend: `qbittorrent:8080`
- Audiobookshelf backend: `audiobookshelf:80`
- WordPress backend: `wp_rocket:80` or `wp_jph:80`
- Home Assistant backend: `192.168.1.31:8123` (uses host network)

## 5. Reverse Proxy and Public Access
### DNS and router
- `rocket.int.yt` and selected subdomains point to the home public IP.
- Router forwards these web ports to `192.168.1.31`:
  - `80/tcp`
  - `443/tcp`

### Nginx Proxy Manager
- NPM is the single public HTTP/HTTPS entry point.
- NPM admin UI remains LAN-only on port `81`.
- Do **not** expose `81` publicly.
- For normal public web access, expose only `80` and `443`.
- **Standard Proxy Configuration:**
  - Enable **Block Common Exploits**.
  - Enable **Force SSL**, **HTTP/2 Support**, **HSTS Enabled**, and **HSTS Subdomains**.
  - For Home Assistant and some modern apps, enable **Websockets Support**.

## 6. qBittorrent
### Role
qBittorrent is part of the main Compose stack and is reachable publicly only through NPM for its Web UI.
### Paths
- Config: `/opt/containers/qbittorrent/config`
- Downloads root on host: `/data/downloads/qbittorrent`
### VueTorrent
VueTorrent is enabled via Docker mod:
- `DOCKER_MODS=ghcr.io/vuetorrent/vuetorrent-lsio-mod:latest`
- Alternative Web UI path: `/vuetorrent`

## 7. Audiobookshelf & Library Manager
### Role
Audiobookshelf handles your audiobooks and podcasts. The Library Manager is accessible as a subpath.
### Paths
- Audiobooks: `/data/media/audiobooks`
- Podcasts: `/data/media/podcasts`
- Config: `/opt/containers/audiobookshelf/config`
- Metadata: `/opt/containers/audiobookshelf/metadata`
### Networking
- Main App: `books.rocket.int.yt` -> `audiobookshelf:80`
- **Library Manager:** `books.rocket.int.yt/manage` -> `library-manager:5000` (NPM Custom Location)
### Maintenance
- **File Watcher:** Enabled via `CONFIG_PATH` and `METADATA_PATH` environment variables.

## 8. Plex Media Server
### Role
Plex uses **host network mode** for discovery and DLNA.
### Paths
- Config: `/opt/containers/plex/config`
- Media: `/data/media`
### Networking
- Network Mode: `host`
- Default Port: `32400`

## 9. Home Assistant
### Role
Home Automation platform.
### Paths
- Config: `/opt/containers/homeassistant/config`
### Networking
- Network Mode: `host`
- Default Port: `8123`
- Suggested public hostname: `ha.rocket.int.yt`
### Reverse Proxy Configuration
To fix "400 Bad Request" when using NPM, the following must be in `configuration.yaml`:
```yaml
http:
  use_x_forwarded_for: true
  trusted_proxies:
    - 172.16.0.0/12
    - 192.168.1.31
```

## 10. WordPress Stack (MariaDB + WordPress)
### Role
A multi-site WordPress setup using a single MariaDB instance.
### Paths
- MariaDB Data: `/opt/containers/mariadb/data`
- Site 1 (jphardwoodflooring.com): `/home/pi/projects/sites/jphardwoodflooring.com/public`
- Site 2 (wp.rocket.int.yt): `/home/pi/projects/sites/wp.rocket.int.yt/public`
### WP-CLI
WP-CLI is installed in both containers for management.

## 11. Maintenance & Automation (Watchtower)
Watchtower is used to automate container updates with an opt-in strategy.
- **Schedule:** Every Sunday at 3:00 AM local time (`0 0 3 * * 0`).
- **Image Cleanup:** Enabled to remove old images after updates.
- **Opt-in Mode:** Only containers with the label `com.centurylinklabs.watchtower.enable=true` are updated.
- **Enabled Services:** `plex`, `audiobookshelf`.

## 12. Guardrails
- Keep `pi` as the main admin user.
- Keep important app data host-visible.
- Avoid broad mounts unless truly needed.
- **Security:** Always enable exploit blocking and HSTS in NPM for public-facing sites.
- **Inter-container proxying:** Prefer service names on the Compose network.

## 13. Current Public Hostname Plan
- `torrent.rocket.int.yt` -> qBittorrent
- `books.rocket.int.yt` -> Audiobookshelf
- `books.rocket.int.yt/manage` -> Library Manager
- `ha.rocket.int.yt` -> Home Assistant
- `wp.rocket.int.yt` -> WordPress site
- `jphardwoodflooring.com` -> WordPress site
- `npm.rocket.int.yt` -> Nginx Proxy Manager (Internal)
