# Raspberry Pi Home Server Spec (Condensed, Current)

## 1. Purpose

This is the current short-form spec for the Raspberry Pi home-server rebuild. It replaces the earlier longer handoff as the working reference for the setup that was actually shaped in this chat.

The goal is a clean, host-visible, Docker-based setup with one main Compose project, one reverse proxy front end, and clearly separated media, downloads, and site content.

## 2. Baseline

- Host: Raspberry Pi 5
- Main admin user: `pi`
- Hostname: `pi`
- Fixed LAN IP: `192.168.1.31`
- Main domain: `rocket.int.yt`
- OS/root storage: local SSD on the Pi
- Important note: `/data` is not a default system folder. It is a manually created host directory on the Pi's filesystem unless a separate drive is mounted there later.

## 3. Host Layout

Use these host paths:

- `/opt/containers`  
  Main Docker project root and service config area.

- `/opt/containers/docker-compose.yml`  
  The single main Compose file for the current stack.

- `/opt/containers/npm`  
  Nginx Proxy Manager persistent state.

- `/opt/containers/qbittorrent`  
  qBittorrent config.

- `/opt/containers/audiobookshelf`  
  Audiobookshelf config and metadata.

- `/opt/containers/apachephp`  
  Future custom PHP/Apache build context and vhost config.

- `/data/media`  
  Shared media library area.

- `/data/downloads`  
  Shared downloads area.

- `/home/pi/projects/sites`  
  Future website content root.

## 4. Compose Project Model

The current direction is **one main Compose project**, not one Compose file per service.

- Compose file location: `/opt/containers/docker-compose.yml`
- Bring services up from `/opt/containers`
- Containers in this file share the default Compose network
- NPM should proxy to backend services by **service name** when possible

Examples:

- qBittorrent backend: `qbittorrent:8080`
- Audiobookshelf backend: `audiobookshelf:80`
- Future PHP/Apache backend: `apachephp:80`

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

### Important NPM adjustment

Do **not** set `PUID` / `PGID` on the NPM container. During this rebuild, that caused write/permission failures in NPM's mounted `/data` path. Let NPM manage its own internal ownership model.

## 6. qBittorrent

### Role

qBittorrent is part of the main Compose stack and is reachable publicly only through NPM for its Web UI.

### Paths

- Config: `/opt/containers/qbittorrent/config`
- Downloads root on host: `/data/downloads/qbittorrent`

### Networking

- Internal Web UI port: `8080`
- Public Web UI URL: `https://torrent.rocket.int.yt`
- NPM backend target: `qbittorrent:8080`

### Torrent port

The current torrent listening port is:

- `49494/tcp`
- `49494/udp`

Forward both from the router to `192.168.1.31` for peer connectivity.

### Mount rule

Do **not** bind all of `/data` into qBittorrent. Restrict it to the downloads area. The intended mount is the qBittorrent downloads tree, not the entire shared data tree.

### VueTorrent

VueTorrent is enabled via Docker mod on the LinuxServer qBittorrent container:

- `DOCKER_MODS=ghcr.io/gabe565/linuxserver-mod-vuetorrent`

After container recreation, enable qBittorrent's alternative Web UI and set the files location to:

- `/vuetorrent`

### Note on incomplete downloads

qBittorrent is intended to move completed downloads out of the incomplete folder, but that behavior is not perfectly reliable in practice. A finished torrent may still remain in the incomplete folder in some cases.

## 7. Audiobookshelf

### Role

Audiobookshelf is part of the main Compose stack and is intended to be reverse-proxied by NPM.

### Paths

- Audiobooks: `/data/media/audiobooks`
- Podcasts: `/data/media/podcasts`
- Config: `/opt/containers/audiobookshelf/config`
- Metadata: `/opt/containers/audiobookshelf/metadata`

### Networking

- Internal app port: `80`
- NPM backend target: `audiobookshelf:80`
- Suggested public hostname: `books.rocket.int.yt`

## 8. Planned PHP/Apache Web Hosting Model

This design was agreed conceptually and should be used when Apache/PHP is added to the stack.

### Container model

- Use a custom `php:8.2-apache` image in the same main Compose file.
- Do not publish a host port for Apache if it is only meant to sit behind NPM.
- NPM should proxy to `apachephp:80`.

### Site layout

All sites live under one root:

- `/home/pi/projects/sites`

Each site gets its own folder named by its full base hostname, for example:

- `/home/pi/projects/sites/example.com`
- `/home/pi/projects/sites/rocket.int.yt`

Recommended subfolders per site:

- `public` for the web root
- `bin` for scripts
- `var` for generated files, uploads, and working data

### Virtual hosts

Use Apache name-based virtual hosts. Each site gets its own vhost file. The document root points to that site's `public` folder.

### PHP running shell commands

PHP is allowed to execute shell commands intentionally in this design, including running Python scripts.

Guardrails:

- Keep executable scripts inside the mounted site tree, preferably in each site's `bin` folder.
- Install Python inside the Apache/PHP container.
- Prefer calling scripts by full path.
- Treat shell execution as deliberate application behavior, not a general host-control mechanism.

## 9. Guardrails

- Keep `pi` as the main admin user.
- Keep important app data host-visible.
- Avoid broad mounts unless they are truly needed.
- Prefer one clear path for each category of data.
- Do not expose NPM admin publicly.
- Do not expose backend web ports publicly when NPM is meant to front them.
- For inter-container proxying, prefer service names on the Compose network instead of host IP + published port.
- If a temporary debug workaround is needed, publishing a backend port on the host is acceptable, but it is not the preferred steady-state design.

## 10. Current Public Hostname Plan

- `torrent.rocket.int.yt` -> qBittorrent Web UI through NPM
- `books.rocket.int.yt` -> Audiobookshelf through NPM
- `rocket.int.yt` and future domains/subdomains -> Apache/PHP sites through NPM when that service is added
