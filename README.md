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

- `/opt/containers/plex`  
  Plex Media Server config.

- `/opt/containers/mariadb`  
  MariaDB database files for WordPress and other services.

- `/data/media`  
  Shared media library area.

- `/data/downloads`  
  Shared downloads area.

- `/home/pi/projects/sites`  
  Website content root.

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

## 8. Plex Media Server

### Role

Plex is part of the main Compose stack and uses **host network mode** for better discovery and DLNA support.

### Paths

- Config: `/opt/containers/plex/config`
- Media: `/data/media`

### Networking

- Network Mode: `host`
- Default Port: `32400`

## 9. WordPress Stack (MariaDB + WordPress)

### Role

A multi-site WordPress setup using a single MariaDB instance and separate WordPress containers.

### Paths

- MariaDB Data: `/opt/containers/mariadb/data`
- Site 1 (jphardwoodflooring.com): `/home/pi/projects/sites/jphardwoodflooring.com/public`
- Site 2 (wp.rocket.int.yt): `/home/pi/projects/sites/wp.rocket.int.yt/public`

### Networking

- MariaDB Port: `3306` (Internal to Compose network)
- WordPress Ports: `80` (Internal to Compose network)
- NPM Backend Targets: `wp_jph:80` and `wp_rocket:80`

### Environment

Sensitive credentials (DB passwords, etc.) are stored in a host-side `.env` file at `/opt/containers/.env`.

### WP-CLI

WP-CLI is installed in both WordPress containers to allow for command-line management of the sites.

## 10. Guardrails

- Keep `pi` as the main admin user.
- Keep important app data host-visible.
- Avoid broad mounts unless they are truly needed.
- Prefer one clear path for each category of data.
- Do not expose NPM admin publicly.
- Do not expose backend web ports publicly when NPM is meant to front them.
- For inter-container proxying, prefer service names on the Compose network instead of host IP + published port.
- **Security:** Always enable exploit blocking and HSTS in NPM for public-facing sites.

## 11. Current Public Hostname Plan

- `torrent.rocket.int.yt` -> qBittorrent Web UI through NPM
- `books.rocket.int.yt` -> Audiobookshelf through NPM
- `wp.rocket.int.yt` -> WordPress site through NPM
- `jphardwoodflooring.com` -> WordPress site through NPM
- `npm.rocket.int.yt` -> Nginx Proxy Manager through NPM (Internal use)
