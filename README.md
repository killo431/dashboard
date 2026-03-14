# Home Server Dashboard

A [Homepage](https://gethomepage.dev/) configuration for a self-hosted media automation stack using a **Shared-Group Model**: each service runs as its own user for security, but shares a common `jellyfin` group (GID 1001) for cross-service data access on the XFS pool.

---

## Service Map

| Component   | Port | User     | Group    | Config Path                              |
|-------------|------|----------|----------|------------------------------------------|
| Homepage    | 3000 | root     | root     | `~/Desktop/Dashboard/homepage-dashboard` |
| Jellyfin    | 8096 | jellyfin | jellyfin | `/var/lib/jellyfin`                      |
| Sonarr      | 8989 | sonarr   | jellyfin | `/var/lib/sonarr`                        |
| Radarr      | 7878 | radarr   | jellyfin | `/var/lib/radarr`                        |
| SABnzbd     | 8080 | sonarr*  | jellyfin | `/home/jellyfin/.sabnzbd`                |
| Prowlarr    | 9696 | prowlarr | jellyfin | `/var/lib/prowlarr`                      |
| Jellyseerr  | 5055 | jellyfin | jellyfin | `/var/lib/jellyseerr`                    |
| FlareSolverr| 8191 | —        | —        | —                                        |

> **\* SABnzbd** runs as the `sonarr` user so that any file downloaded to `/mnt/xfs_pool/Complete` is immediately writable by the Arr apps, without a `chown` step.

---

## Repository Layout

```
├── docker-compose.yaml          # Starts the full stack
└── config/
    ├── services.yaml            # Dashboard tabs & service cards
    ├── settings.yaml            # Homepage appearance settings
    └── widgets.yaml             # Top-bar info widgets (CPU, disk, clock)
```

---

## Dashboard Logic — `config/services.yaml`

The dashboard is split into three functional tabs to prevent information overload:

| Tab          | Services                                |
|--------------|-----------------------------------------|
| **Media**    | Jellyfin, Jellyseerr                    |
| **Automation** | Sonarr, Radarr                        |
| **Downloads** | SABnzbd, Prowlarr, FlareSolverr       |

Each card is enriched with three data layers:

### Layer A · Live Logs (`enableLogs: true`)
By mapping `/var/run/docker.sock` into the Homepage container, `enableLogs: true` streams the Docker binary log API directly to an **Xterm.js** terminal in the browser.  
Debugging a failed download or a failed library scan no longer requires SSH — it's one click away.

> Homepage **must** run as `user: root` in `docker-compose.yaml` for socket access.

### Layer B · Power & Boot Sliders (`enableAction: true`)
`enableAction: true` exposes a power toggle on each card.  
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

### 3. Start the stack

```bash
docker compose up -d
```

Homepage will be available at **http://localhost:3000**.

---

## Troubleshooting

**Logs not showing?**  
Verify Homepage is running as `user: root` in `docker-compose.yaml` and that `/var/run/docker.sock` is mounted.

**Permission Issues on the XFS pool?**  
Run the "Permission Glue" command:
```bash
sudo chown -R :jellyfin /mnt/xfs_pool/Media && sudo chmod -R 775 /mnt/xfs_pool/Media
```

**API Timeouts?**  
Check that the API keys in `config/services.yaml` match the **Security** tab in each respective app's settings.