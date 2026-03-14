# Home Server Dashboard

A [Homepage](https://gethomepage.dev/) configuration for a self-hosted media automation stack.

---

## Service Map

### Docker Containers

| Component          | Port  | Config Path                              |
|--------------------|-------|------------------------------------------|
| Homepage           | 3000  | `~/Desktop/Dashboard/homepage-dashboard` |
| Glances            | 61208 | —                                        |
| Speedtest Tracker  | 8765  | `/var/lib/speedtest-tracker`             |
| Jellyseerr         | 5055  | `/var/lib/jellyseerr`                    |
| FlareSolverr       | 8191  | —                                        |

### Native Services (non-Docker)

| Component   | Port | User     | Group    | Config Path               |
|-------------|------|----------|----------|---------------------------|
| Jellyfin    | 8096 | jellyfin | jellyfin | `/var/lib/jellyfin`       |
| Sonarr      | 8989 | sonarr   | jellyfin | `/var/lib/sonarr`         |
| Radarr      | 7878 | radarr   | jellyfin | `/var/lib/radarr`         |
| SABnzbd     | 8080 | sonarr*  | jellyfin | `/home/jellyfin/.sabnzbd` |
| Prowlarr    | 9696 | prowlarr | jellyfin | `/var/lib/prowlarr`       |

> **\* SABnzbd** runs as the `sonarr` user so that any file downloaded to `/mnt/xfs_pool/Complete` is immediately writable by the Arr apps, without a `chown` step.

---

## Repository Layout

```
├── docker-compose.yaml          # Starts the Docker container stack
└── config/
    ├── services.yaml            # Dashboard tabs & service cards
    ├── settings.yaml            # Homepage appearance settings
    └── widgets.yaml             # Top-bar info widgets (CPU, disk, clock)
```

---

## Dashboard Logic — `config/services.yaml`

The dashboard is split into four functional tabs:

| Tab          | Services                                |
|--------------|-----------------------------------------|
| **Media**    | Jellyfin, Jellyseerr                    |
| **Automation** | Sonarr, Radarr                        |
| **Downloads** | SABnzbd, Prowlarr, FlareSolverr       |
| **System**   | Glances, Speedtest Tracker              |

Docker container cards (homepage, glances, speedtest-tracker, jellyseerr, flaresolverr) are enriched with:

### Layer A · Live Logs (`enableLogs: true`)
By mapping `/var/run/docker.sock` into the Homepage container, `enableLogs: true` streams the Docker binary log API directly to an **Xterm.js** terminal in the browser.  
Debugging no longer requires SSH — it's one click away.

> Homepage **must** run as `user: root` in `docker-compose.yaml` for socket access.

### Layer B · Power & Boot Sliders (`enableAction: true`)
`enableAction: true` exposes a power toggle on each Docker container card.  
Toggling it modifies the container's Docker **RestartPolicy**, controlling whether the service auto-starts across reboots.

### Layer C · Custom API Buttons
Direct `POST` requests bypass the standard web UI:

| Service | Endpoint | Effect |
|---------|----------|--------|
| Jellyfin | `POST /Library/Refresh` | Forces immediate XFS pool scan |
| Sonarr | `POST /api/v3/command` → `MissingEpisodeSearch` | Searches for missing episodes |
| Sonarr | `POST /api/v3/command` → `RefreshSeries` | Refreshes all series metadata |
| Radarr | `POST /api/v3/command` → `MissingMoviesSearch` | Searches for missing movies |
| Radarr | `POST /api/v3/command` → `RefreshMovie` | Refreshes all movie metadata |

---

## Data Flow

```
User (Jellyseerr :5055)
  └─► Radarr (:7878)
        └─► Prowlarr (:9696)  ←─ FlareSolverr (:8191) [Cloudflare bypass]
              └─► SABnzbd (:8080)  [NZB download → /mnt/xfs_pool/Complete]
                    └─► Radarr moves file → /mnt/xfs_pool/Media/Movies  (GID 1001)
                          └─► Jellyfin (:8096) detects & indexes new media
```

---

## Quick Start

### 1. Clone and configure

```bash
git clone https://github.com/killo431/dashboard.git ~/Desktop/Dashboard/homepage-dashboard
cd ~/Desktop/Dashboard/homepage-dashboard
```

### 2. Add your API keys

Edit `config/services.yaml` and replace the placeholder values:

| Placeholder | Where to find it |
|---|---|
| `{{JELLYFIN_API_KEY}}` | Jellyfin → Dashboard → API Keys |
| `{{JELLYSEERR_API_KEY}}` | Jellyseerr → Settings → General |
| `{{SONARR_API_KEY}}` | Sonarr → Settings → General → Security |
| `{{RADARR_API_KEY}}` | Radarr → Settings → General → Security |
| `{{SABNZBD_API_KEY}}` | SABnzbd → Config → General → API Key |
| `{{PROWLARR_API_KEY}}` | Prowlarr → Settings → General → Security |

### 3. Start the Docker stack

```bash
docker compose up -d
```

Homepage will be available at **http://localhost:3000**.

---

## Troubleshooting

**Logs not showing for Docker containers?**  
Verify Homepage is running as `user: root` in `docker-compose.yaml` and that `/var/run/docker.sock` is mounted.

**Logs/actions not available for native services?**  
Jellyfin, Sonarr, Radarr, SABnzbd, and Prowlarr run natively (not in Docker), so the `enableLogs` and `enableAction` features are not available for these services. Use their native service management (e.g. `systemctl`) instead.

**Permission Issues on the XFS pool?**  
Run the "Permission Glue" command:
```bash
sudo chown -R :jellyfin /mnt/xfs_pool/Media && sudo chmod -R 775 /mnt/xfs_pool/Media
```

**API Timeouts?**  
Check that the API keys in `config/services.yaml` match the **Security** tab in each respective app's settings.