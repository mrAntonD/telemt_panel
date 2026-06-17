# Telemt Panel

**English** | [Русский](README.ru.md)

[![CI](https://github.com/amirotin/telemt_panel/actions/workflows/ci.yml/badge.svg)](https://github.com/amirotin/telemt_panel/actions/workflows/ci.yml)
[![Release](https://img.shields.io/github/v/release/amirotin/telemt_panel?include_prereleases)](https://github.com/amirotin/telemt_panel/releases)
[![Go](https://img.shields.io/badge/Go-1.24-00ADD8?logo=go)](https://go.dev/)
[![React](https://img.shields.io/badge/React-18-61DAFB?logo=react)](https://react.dev/)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Stars](https://img.shields.io/github/stars/amirotin/telemt_panel?style=social)](https://github.com/amirotin/telemt_panel/stargazers)

Web management panel for [Telemt](https://github.com/telemt/telemt) MTProxy. Monitor server status, manage users, track security, and update binaries — all from the browser.

The panel UI supports **English and Russian** languages. Switch via the globe button in the sidebar — the preference is saved in the browser.

## Table of Contents

- [Screenshots](#screenshots)
- [Features](#features)
- [Requirements](#requirements)
- [Quick Start](#quick-start)
- [Docker](#docker)
- [Build](#build)
- [Configuration](#configuration)
- [Telegram Bot](#telegram-bot)
- [Systemd](#systemd)
- [CLI](#cli)
- [Tech Stack](#tech-stack)
- [License](#license)

## Screenshots

| Dashboard | Users | Runtime |
|:---------:|:-----:|:-------:|
| ![Dashboard](docs/screenshots/dashboard.png) | ![Users](docs/screenshots/users.png) | ![Runtime](docs/screenshots/runtime.png) |

## Features

- **Dashboard** — server health, uptime, connections, total traffic, DC status
- **Users** — CRUD via Telemt API, column sorting, active connection highlighting
- **Runtime** — connections, ME pool state, ME quality, upstream quality, NAT/STUN, self-test, events
- **Security** — posture (read-only, whitelist, auth header), limits, whitelist
- **Upstreams** — upstream server and pool status
- **Updates** — check for new releases on GitHub, one-click binary update with rollback on error (Telemt and panel)
- **TLS** — custom certificates and automatic Let's Encrypt (ACME)
- **GeoIP** — IP geolocation via MaxMind GeoLite2
- **WebSocket** — real-time data updates without page reload
- **Base Path** — reverse proxy subpath support
- **Telegram Bot** — built-in Go bot: auto-start on panel launch, Start/Stop button with live status, token and admin IDs configured via UI and saved to config.toml
- **i18n** — English / Russian UI switcher (saved to localStorage)

## Requirements

- Linux (x86_64 or aarch64), any distribution (Debian, Ubuntu, Alpine, CentOS, etc.)
- A running Telemt server with an accessible API

To build from source:
- Go 1.24+
- Node.js 20+
- Docker (optional, for cross-compilation)

## Quick Start

### Install via script

```bash
curl -fsSL https://raw.githubusercontent.com/amirotin/telemt_panel/main/install.sh | bash
```

The script downloads the binary, creates a config, sets up a systemd service and starts the panel.

### Manual installation

1. Download the binary from [Releases](https://github.com/amirotin/telemt_panel/releases) (or build it yourself — see below).

2. Create config:

```bash
sudo mkdir -p /etc/telemt-panel
sudo cp config.example.toml /etc/telemt-panel/config.toml
sudo chmod 600 /etc/telemt-panel/config.toml
```

3. Generate a password hash and JWT secret:

```bash
# Password hash
./telemt-panel hash-password

# JWT secret
openssl rand -hex 32
```

4. Edit `/etc/telemt-panel/config.toml`:

```toml
listen = "0.0.0.0:8080"

[telemt]
url = "http://127.0.0.1:9091"
auth_header = ""

[auth]
username = "admin"
password_hash = "$2a$10$..."   # output of hash-password
jwt_secret = "your_secret"     # output of openssl rand
session_ttl = "24h"
```

5. Start:

```bash
./telemt-panel --config /etc/telemt-panel/config.toml
```

The panel will be available at `http://your_server:8080`.

## Docker

The panel ships with a ready-to-use `docker-compose.yml` that runs telemt and telemt-panel together. The Telegram bot is built into the panel binary — no Python or external dependencies needed.

### File structure

```
/opt/telemt/
├── docker-compose.yml
├── panel-config/
│   └── config.toml          # panel config (must be writable)
└── telemt-config/
    └── telemt.toml          # Telemt config (read by bot for auto domain detection)
```

### docker-compose.yml

```yaml
services:
  telemt:
    build: .   # or image: whn0thacked/telemt-docker:latest
    container_name: telemt
    restart: unless-stopped
    volumes:
      - ./telemt-config:/etc/telemt:rw
    network_mode: host

  telemt-panel:
    image: ghcr.io/mrantond/telemt_panel:latest   # pre-built image — no build needed
    container_name: telemt-panel
    restart: unless-stopped
    volumes:
      - ./panel-config/config.toml:/etc/telemt-panel/config.toml   # no :ro — panel writes bot settings
      - ./telemt-config/telemt.toml:/etc/telemt/config.toml        # no :ro — panel can edit Telemt config via Advanced Editor
      - telemt-panel-data:/var/lib/telemt-panel                     # data survives container recreation
    depends_on:
      - telemt
    networks:
      - traefik

volumes:
  telemt-panel-data:

networks:
  traefik:
    external: true
```

> **Important:**
> - Both config files are mounted **without `:ro`** — the panel writes bot settings to its own config and can edit the Telemt config via the Configuration → Advanced Editor. Mounting a single file as read-only (`:ro`) would break saving: Docker's bind-mount prevents atomic rename, and the read-only flag blocks the fallback write.
> - The bot automatically extracts the proxy domain (`general.links.public_host`), port and TLS domain from the Telemt config — no extra configuration needed.
> - The `telemt-panel-data` volume preserves the bot database and extracted files across `docker compose down` / container recreation.

### Panel config (panel-config/config.toml)

```toml
listen = "0.0.0.0:8080"

[telemt]
url = "http://172.18.0.1:9091"          # Docker host IP (host.docker.internal or gateway)
config_path = "/etc/telemt/config.toml" # path to telemt.toml inside the container

[auth]
username = "admin"
password_hash = "$2a$10$..."
jwt_secret = "your_secret"
```

> `telemt.config_path` is the key parameter for Docker: without it the bot cannot read the proxy domain from `telemt.toml`.

### Start

```bash
docker compose up -d
```

### Update

```bash
docker compose pull telemt-panel && docker compose up -d telemt-panel
```

The image is published automatically to `ghcr.io/mrantond/telemt_panel:latest` on every push to `main` via GitHub Actions — no local build needed.

### Logs

```bash
docker logs telemt-panel -f
docker logs telemt -f
```

## Build

### Simple (local)

```bash
make            # build frontend + backend
make release    # build binaries for x86_64 and aarch64
```

### Via Docker (cross-compilation)

```bash
# Linux/macOS
./build.sh

# Windows
build.bat
```

Binaries are placed in `./release/`:
- `telemt-panel-x86_64-linux`
- `telemt-panel-aarch64-linux`
- `SHA256SUMS`

Binaries are static (`CGO_ENABLED=0`) — they run on any Linux without dependencies.

### Override version

```bash
make backend VERSION=1.2.3
# or
go build -ldflags="-s -w -X main.version=1.2.3" -o telemt-panel .
```

## Configuration

Full configuration example: [`config.example.toml`](config.example.toml).

| Section | Parameter | Description | Default |
|---------|-----------|-------------|---------|
| — | `listen` | Bind address and port | `0.0.0.0:8080` |
| — | `base_path` | Subpath for reverse proxy (e.g. `/panel123`) | — |
| `[telemt]` | `url` | Telemt API URL | **required** |
| `[telemt]` | `auth_header` | Authorization header for Telemt API | — |
| `[telemt]` | `binary_path` | Path to telemt binary (for updates) | `/bin/telemt` |
| `[telemt]` | `service_name` | systemd service name | `telemt` |
| `[telemt]` | `github_repo` | GitHub repository for update checks | `telemt/telemt` |
| `[telemt]` | `config_path` | Path to Telemt config (for Docker / non-standard paths) | auto from API |
| `[telemt]` | `config_edit_mode` | Telemt config edit mode: `api` (via API, Docker-safe, default) or `file` (file edit, requires sudoers tee on systemd) | `api` |
| `[panel]` | `binary_path` | Path to panel binary (for self-update) | `/usr/local/bin/telemt-panel` |
| `[panel]` | `service_name` | systemd service name for the panel | `telemt-panel` |
| `[panel]` | `github_repo` | Panel GitHub repository | `amirotin/telemt_panel` |
| `[panel]` | `github_token` | Personal Access Token for GitHub API. Without token: 60 req/h per IP (shared); with token: 5000/h. Needed when update checks hit rate limits. A fine-grained PAT without scopes is sufficient: [create token](https://github.com/settings/personal-access-tokens/new) | — |
| `[panel]` | `max_newer_releases` | Max newer releases shown in update list | `10` |
| `[panel]` | `max_older_releases` | Max older releases shown in update list | `10` |
| `[auth]` | `username` | Admin login | **required** |
| `[auth]` | `password_hash` | Bcrypt password hash | **required** |
| `[auth]` | `jwt_secret` | JWT signing secret | **required** |
| `[auth]` | `session_ttl` | Session lifetime | `24h` |
| `[tls]` | `cert_file` / `key_file` | Custom TLS certificates | — |
| `[tls]` | `acme_domain` | Domain for Let's Encrypt | — |
| `[tls]` | `acme_cache_dir` | Certificate cache directory | `/var/lib/telemt-panel/certs` |
| `[geoip]` | `db_path` | Path to MaxMind GeoLite2 City (.mmdb) | — |
| `[geoip]` | `asn_db_path` | Path to MaxMind GeoLite2 ASN (.mmdb) | — |
| `[telegram]` | `enabled` | Auto-start bot on panel launch | `false` |
| `[telegram]` | `bot_token` | Telegram bot token (get from `@botfather`) | — |
| `[telegram]` | `admin_ids` | Array of Telegram User IDs for bot admins | — |
| `[telegram]` | `default_max_tcp_conns` | Default TCP session limit per new client | `50` |
| `[telegram]` | `default_max_unique_ips` | Default unique IP limit per client (alerts admin when exceeded) | `5` |

## Telegram Bot

The bot is built into the panel binary as a native goroutine — no Python or external dependencies needed.

### Bot capabilities

- User registration via requests (admin approves / rejects)
- Personal MTProxy link with QR code
- Traffic and active IP statistics per user
- Admin menu: all clients statistics, broadcast, blocklist, DB backup
- User message forwarding to admins with Reply support
- Telemt API availability monitoring with Telegram notifications
- Restart notification to admins when the panel restarts

### Initial setup

Open the **Telegram Bot** section in the panel sidebar:

- **Bot Token** — paste the token obtained from `@botfather`
- **Admin User IDs** — Telegram User IDs of admins (one per line); get your ID via `@userinfobot` or the `/id` command in the bot
- **Max sessions per client** — TCP connection limit applied when creating a new user (default: 50)
- **Max unique IPs per client** — the bot notifies admins when a user exceeds this threshold (default: 5)
- Click **Save**, then **Start**

Settings are saved to the `[telegram]` section of `config.toml`. The bot starts automatically on next panel launch.

> **Note:** The bot is built into the panel binary — no Python or external dependencies required.

### Proxy domain auto-detection

The bot automatically reads the proxy domain, port and TLS domain from the Telemt config. Set the Telemt config path in the panel's `config.toml` via `telemt.config_path`.

**Docker:** ensure `telemt.toml` is mounted into the panel container and `config_path` points to it:

```toml
[telemt]
config_path = "/etc/telemt/config.toml"
```

**Bare metal:** if Telemt and the panel are on the same machine:

```toml
[telemt]
config_path = "/etc/telemt/telemt.toml"
```

The bot reads from the Telemt config:
- `general.links.public_host` → proxy domain for MTProxy links
- `general.links.public_port` → port (default `4448`)
- `censorship.tls_domain` → TLS domain for Fake-TLS links (`ee` format)

### Panel controls

| Action | Description |
|--------|-------------|
| **Start** | Starts the bot goroutines, sets `enabled = true` in config.toml |
| **Stop** | Stops the bot, sets `enabled = false` |
| **Save** | Updates token and admin IDs; restarts the bot if running |
| Status badge | Updates every 3 s; shows "Running", "Stopped" or "Error" |

## Systemd

### Install via script (recommended)

The script automatically creates a `telemt-panel` system user, installs the panel binary to `/usr/local/bin`, config to `/etc/telemt-panel`, data to `/var/lib/telemt-panel`, configures a narrow `sudoers` drop-in for updates, and generates a hardened systemd unit:

```bash
curl -fsSL https://raw.githubusercontent.com/amirotin/telemt_panel/main/install.sh | bash
```

| Component | Path |
|-----------|------|
| Panel binary | `/usr/local/bin/telemt-panel` |
| Panel config | `/etc/telemt-panel/config.toml` |
| Data (cert cache etc.) | `/var/lib/telemt-panel/` |
| systemd unit | `/etc/systemd/system/telemt-panel.service` |
| sudoers drop-in | `/etc/sudoers.d/telemt-panel` |

The generated unit includes:

```ini
[Service]
User=telemt-panel
ProtectHome=true
PrivateTmp=true
ReadWritePaths=/etc/telemt-panel /var/lib/telemt-panel
```

`NoNewPrivileges` is not set because the service uses `sudo` for strictly limited update operations.

The `sudoers` drop-in allows the `telemt-panel` user to run only the commands needed for updates: replacing binaries, cleaning staging files, and restarting `telemt.service` and `telemt-panel.service`.

### Manual installation

```bash
sudo useradd --system --shell /usr/sbin/nologin --home /nonexistent telemt-panel
sudo cp telemt-panel.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now telemt-panel
```

> **Note:** this unit file runs the panel as the `telemt-panel` user.
> If installing manually, create the user and configure permissions equivalent to the installer-managed `sudoers` drop-in, or edit the unit for your deployment model.

### For Telegram bot (bare metal)

If Telemt is on the same machine, set the path to its config in the panel's `config.toml`:

```toml
[telemt]
config_path = "/etc/telemt/telemt.toml"
```

The bot will read the domain, port and TLS domain from this file automatically.

### Uninstall

```bash
# Service and binary only (config and data preserved)
./install.sh uninstall

# Full removal (including telemt-panel user)
./install.sh purge
```

Logs:

```bash
sudo journalctl -u telemt-panel -f
```

## CLI

```bash
telemt-panel --config config.toml    # start the server
telemt-panel hash-password           # generate bcrypt hash
telemt-panel version                 # show version
```

## Tech Stack

- **Backend:** Go 1.24, standard library + gorilla/websocket, golang-jwt, BurntSushi/toml
- **Frontend:** React 18, TypeScript, Tailwind CSS, Vite, i18next
- **Build:** Multi-stage Docker, static linking

## License

[MIT](LICENSE)
