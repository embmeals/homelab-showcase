# Homelab Backend

> Project overview for a private repository. The source stays private; this README is synced from it automatically.

[![License](https://img.shields.io/badge/license-MIT-22d3ee)](LICENSE)
[![.NET 10](https://img.shields.io/badge/.NET-10.0-512BD4?logo=dotnet&logoColor=white)](https://dotnet.microsoft.com/)

A .NET 10 application that monitors and manages a multi-machine homelab infrastructure. Built with Hangfire for job scheduling, Razor Pages + HTMX for a live-updating dashboard, and EF Core + SQLite for persistence.

**Live at** [homelab.emills.net](https://homelab.emills.net) (protected by Cloudflare Access)

## What It Does

Monitors a homelab spanning a Mac server, a Synology NAS, a Windows PC and a Raspberry Pi 5, running Plex, Radarr, Sonarr, Prowlarr, qBittorrent, Syncthing, Bazarr and a dozen Docker containers. Replaces a collection of shell scripts and a Python monitor with a single, always-on service.

### Public Services
These are the services this app monitors — some are publicly accessible:

| Service | Public URL                                         | Purpose |
|---------|----------------------------------------------------|---------|
| Seerr | [requests.emills.net](https://requests.emills.net) | Request movies and TV shows |
| Tautulli | [stats.emills.net](https://stats.emills.net)       | Plex viewing statistics |
| Immich | [photos.emills.net](https://photos.emills.net)     | Photo library |
| This Dashboard | [homelab.emills.net](https://homelab.emills.net)   | Monitoring dashboard |
| Portainer | [docker.emills.net](https://docker.emills.net)     | Docker management |

*Protected services require email OTP via Cloudflare Access.*

## Media Pipeline

The core of the homelab is a fully automated media pipeline. Here's how a movie goes from request to screen:

```
 You request a movie         Radarr searches for         Prowlarr feeds
 through Seerr        →    the best quality release  →   indexer results
 (web UI)                   (Movies) / Sonarr (TV)       to Radarr/Sonarr

         ↓                          ↓                          ↓

 qBittorrent downloads      Syncthing transfers         Radarr/Sonarr imports,
 behind Mullvad VPN    →    completed files from   →    renames, and files it
 on the Windows PC          PC to Mac over LAN          on the Synology NAS

         ↓                          ↓                          ↓

 Plex scans the library     Tautulli tracks viewing     Bazarr downloads
 and streams to any    →    stats and history      →    matching subtitles
 device on any network                                  automatically
```

**Why each service exists:**
- **Seerr** — clean request UI so friends/family can request without touching Radarr/Sonarr (the Overseerr successor; same API)
- **Radarr/Sonarr** — automates searching, downloading, and organizing movies/TV
- **Prowlarr** — single place to manage indexers, syncs to Radarr + Sonarr
- **qBittorrent** — download client, bound to Mullvad VPN for privacy
- **Syncthing** — real-time file sync from the PC (downloads) to the Mac (import staging)
- **Synology NAS** — holds the media library on RAID, mounted on the Mac over NFS
- **Plex** — media server with hardware transcoding, serves all devices
- **Tautulli** — Plex analytics and viewing history
- **Bazarr** — automated subtitle downloads for all media, running on the NAS

**Why the PC downloads instead of the Mac:** The VPN protects download traffic. Keeping it on the PC isolates it from the server — if the VPN drops, only the PC is affected. Syncthing handles the transfer seamlessly.

## Dashboard

Dark-mode trading-card aesthetic built on a shared design system. Geist typography, per-destination accent colors, canonical `.card` component with gradient glow, and `prefers-reduced-motion` respected throughout. Live-updates every 30 seconds via HTMX partial swaps — no full-page reloads.

### Navigation

Persistent left sidebar on desktop (≥ 960px), hamburger drawer on mobile. Grouped into **Monitoring** (Dashboard, Architecture, Uptime, Media) and **Tools** (Apps, Hangfire). Each link is a mini trading card with its own accent color — the active destination gets a glow and colored border. Drawer traps focus, `Escape` dismisses, and disables the slide animation under reduced-motion.

### Dashboard tabs

The main Dashboard page splits into four tabs, driven by a `?tab=x` querystring so the URL is shareable and the back button navigates between tabs. `hx-boost` swaps only the tab content when JavaScript is available; without it the links fall back to plain navigation.

- **Overview** — alerts banner, services, disk usage, auto-remediation, CI status, tunnel health
- **Media** — Now Playing, library stats, Seerr requests, torrents, Syncthing, backups, library integrity
- **Infrastructure** — services detail, tunnel health, disks, quick actions, 24-hour response-time chart
- **History** — recent events, auto-remediation log

### Other pages

- **Architecture** — annotated pipeline diagram of how a request flows from Seerr through Radarr/Sonarr to Plex
- **Uptime** — 90-day per-service uptime with daily history bars
- **Media** — the whole Radarr/Sonarr library, browsable and actionable (see below)
- **Apps** — launcher grid of every homelab service with per-app accent, icon, auth hint, and a Local-network shortcut on hover
- **Hangfire** — recurring job list, fire-and-forget queue, failed-job retry

### Media library

`/media` lists everything Radarr and Sonarr know about, split into **Movies** and **Shows** tabs, with search, sortable columns (title, year, size, added), and 50 rows per page. Every filter, sort and pager link is a real URL, so a new tab opens the same view; HTMX swaps just the table when JavaScript is available.

**Filters** — buttons with live counts, because "in the library" means several different things:

| Filter | What it means |
|--------|--------------------------------------------------------------------------------|
| All | Every entry Radarr/Sonarr tracks |
| On disk | The file is there — watchable tonight |
| Wanted | No file, still monitored — the service is actively hunting for it |
| Not downloaded | No file, unmonitored — deliberately kept as a record so it never re-downloads |
| Stale | On disk and flagged by `StaleMediaJob` as unwatched for 180+ days — the cleanup list |
| Watched | On disk and marked watched — candidates to delete once they've been enjoyed |

Lumping wanted and not-downloaded together buried a 42-title backlog inside a 132-title archive. Stale and Watched only count titles still taking up space, so a title leaves those lists the moment its file is deleted.

**Row actions:**
- **Mark watched** — scrobbles the title in Plex *and* writes a permanent local dismissal. Tautulli's watch history doesn't reach back far enough, so `StaleMediaJob` keeps flagging films that were watched years ago; this is the corrective action, and the dismissal is what makes it stick.
- **Monitor / Unmonitor** — flips the monitored flag
- **Delete file** — removes the file from disk and leaves the entry behind unmonitored, so the space is reclaimed without the service re-downloading it. Confirms first, then re-renders the table in the same view, so the title drops out of any filter it no longer matches and the counts update. The outcome banner still shows if Radarr/Sonarr stops answering before the reload.

**Bulk actions** — row checkboxes plus a bulk bar for mark-watched, unmonitor and delete. The selection survives paging, filtering and sorting, so a title ticked on page three is still ticked on page one. Each item is applied on its own, so one unreachable title costs only itself; the result summary reports per outcome (succeeded, skipped, failed) and leaves the failures still ticked for a retry.

Persistent badges mark stale, watched, and per-availability state on every row.

### CI status card

The Overview tab carries a card showing recent GitHub Actions runs for this repo — pass/fail/running per run, with a rollup pill across everything retained. `CiStatusJob` polls the GitHub API every 10 minutes and stores the runs in SQLite, so the API is never in the request path: the card renders from the database and a rate-limited or expired token costs the freshness, not the page.

### Recurring Jobs

| Job | Frequency | What it checks                                                                 |
|-----|-----------|--------------------------------------------------------------------------------|
| Service Health | 5 min | HTTP ping 8 services (incl. Bazarr on the NAS), measure response time          |
| Torrent Monitor | 5 min | SSH to PC, query qBit for stalled/errored downloads                            |
| Syncthing | 5 min | Folder sync status via REST API                                                |
| VPN Watchdog | 5 min | SSH to PC, detect Mullvad drop, auto-reconnect with cooldown                   |
| Remediation | 5 min | Match recent pipeline findings to rules, dispatch known fixes with audit       |
| Alert Notifications | 5 min | Route critical findings through the notification router                        |
| Pipeline Health | 10 min | Drive mounts, qBit save path, arr queues, stale downloads, DNS, Plex libraries |
| Docker Health | 10 min | Colima VM + every container status                                             |
| CI Status | 10 min | Pull recent GitHub Actions runs for this repo into SQLite                       |
| Two Minds | 10 min | Music agents: signed-track age, 24h sessions, live session, player page — critical only once output stops. Also `two_minds_deploy` from deployed.json: unknown (ok) until first seen; missing after first observation or stale = warning; rollback or behind/skipped over 60 min = critical. Stale and lag conditions do not escalate while the PC is unreachable |
| Tunnel Health | 15 min | HTTP check all Cloudflare Tunnel endpoints                                     |
| Request Fulfillment | 15 min | Track Seerr requests end-to-end, flag any stuck > 48h                          |
| Disk Space | 30 min | `df` on every disk in `DiskCheck` — the NAS media mount, the old Saved_Media drive (backup copy) and Backups |
| Media Library Stats | 6 hours | Movie/show/episode counts from Radarr + Sonarr APIs                            |
| Digest Flush | Daily 8 AM | Bundle warning-severity notifications into one email                           |
| Backup Health | Daily 9 AM | SSH to Mac, verify backup age + expected artifacts                             |
| Data Pruning | Daily 3 AM | Delete health check data older than 30 days — never watched dismissals         |
| Docker Update Reporter | Daily 3:15 AM | Dashboard note of Watchtower container updates                                 |
| Subtitle Gap | Daily 4 AM | Find missing subtitles in Bazarr, trigger searches                             |
| Gmail Cleanup | Daily 4:15 AM | Archive old promos/social, apply label rules                                   |
| Immich Maintenance | Daily 4:30 AM | Photo library stats, duplicate detection report                                |
| Disk Usage Trend | Weekly Sun 3:30 AM | 30-day growth projection, alert if filling within 4 weeks                      |
| Library Integrity | Weekly Sun 5 AM | Reconcile Radarr/Sonarr against disk — orphans, ghosts, duplicates             |
| Library Integrity Remediation | Weekly Sun 5:30 AM | Clear ghost findings, behind a volume-health floor check                 |
| Stale Media | Weekly Mon 3:45 AM | Flag unwatched media (>180 days), skipping anything marked watched; optionally auto-unmonitors (off by default) |
| Monthly SLO Report | Monthly 1st 8 AM | Per-service uptime, disk growth, library growth, remediation stats             |

### Quick Actions

| Action | What it does                                  |
|--------|-----------------------------------------------|
| Scan Plex | Trigger library rescan for new files          |
| Restart Plex | Force-kill and restart Plex Media Server      |
| Restart Radarr | Restart Radarr LaunchAgent                    |
| Flush DNS | Clear macOS DNS cache                         |
| Rescan Syncthing | Force folder re-check                         |
| Clean Downloads | Delete stale files (>24h) from PCDownloads    |
| Start Downloads | Connect Mullvad VPN + start qBittorrent on PC |
| Stop VPN | Disconnect Mullvad VPN                        |
| Refresh Media Stats | Re-query Radarr/Sonarr for library counts     |

## Infrastructure

```
┌─────────────────────────────────────────────────────┐
│  Mac (M3 Pro) — Primary Server, always on           │
│                                                     │
│  Native: Plex, Radarr, Sonarr, Prowlarr             │
│  Docker: Seerr, Tautulli, Immich, Portainer,        │
│          Kometa, Watchtower, RustDesk               │
│  Infra:  Cloudflare Tunnel, This Dashboard          │
│  Storage: NAS media over NFS (~/NAS/media),         │
│           Backups 1.8TB (photos + weekly backups)   │
├─────────────────────────────────────────────────────┤
│  Synology DS918+ NAS — Media Library                │
│                                                     │
│  Movies + TV on SHR (RAID 5), Btrfs, ~2.7TB         │
│  Container Manager: Bazarr                          │
│  NFS export to the Mac only                         │
├─────────────────────────────────────────────────────┤
│  Windows PC (RTX 4060 Ti) — Download Relay          │
│                                                     │
│  qBittorrent (VPN-bound), Syncthing, Mullvad VPN    │
│  Downloads here, syncs to Mac, never stores media   │
├─────────────────────────────────────────────────────┤
│  Raspberry Pi 5 — Undertow FM radio box             │
│                                                     │
│  Kiosk album player, OLED + case lights, AI DJ      │
└─────────────────────────────────────────────────────┘
```

### Networking

- All machines on the same LAN subnet; the NAS and the PC have fixed addresses on the gateway
- SSH key auth between machines (PC → Mac, PC → NAS, PC → Pi, NAS → Mac for copies)
- The NAS media share is exported over NFSv3 to the Mac only; a root LaunchDaemon keeps it mounted
- Cloudflare Tunnel exposes services at `*.emills.net` with email OTP protection
- Docker containers on a custom bridge network; Colima routes LAN traffic through vmnet so containers can reach the NAS

## Why Hangfire

Every job in this app used to wrap its `ExecuteAsync` in a catch-all `try-catch` that logged the error and moved on. The job would silently fail, Hangfire would mark it as succeeded, and the bug would go unnoticed.

When we removed the catch-all try-catches and let exceptions propagate, Hangfire immediately surfaced a bug that had been hidden for weeks — `StaleMediaJob` was crashing because Tautulli returns Unix timestamps as JSON numbers, but the code called `GetString()` on them:

Hangfire gives you this for free: failed job visibility, stack traces, automatic retries (2 attempts), and manual requeue. Swallowing exceptions defeats all of it. The rule now: only catch exceptions at system boundaries (SSH, HTTP) for per-item isolation. Never wrap an entire job in a catch-all.

## Notifications

Every job that used to send email directly now routes through an `INotificationRouter` that dispatches by severity:

| Severity | Channels | When to use |
|----------|----------|-------------|
| `Critical` | Email + ntfy push | Paging events — service down, backup missing, VPN won't reconnect |
| `Recovery` | Email + ntfy push | All-clear — a tracked alert resolved and stayed clear through the grace window |
| `Warning` | Daily digest (one email at 8 AM) | Non-urgent — stale media report, disk trends, request stuck > 48h |
| `Info` | Dashboard only | Informational — successful remediations, Docker update summaries |

This keeps the inbox quiet. Critical events page immediately on the phone via ntfy and arrive as email. Down/critical alerts are state-change based: one batched email when alerts appear, a reminder every `Notifications:ReminderMinutes` (default 6h) while they persist, and one `Recovery` notice once they stay clear through `Notifications:RecoveryGraceMinutes` (default 10m). Warnings accumulate in the `PendingDigestNotifications` table and flush once daily via `DigestFlushJob`. Info-level events land in logs and dashboard surfaces without touching email at all.

ntfy push is optional. If `Ntfy.BaseUrl` isn't configured (it currently isn't), `NtfyChannel` silently no-ops and critical alerts arrive by email only.

## Tech Stack

- **.NET 10** — ASP.NET Core Minimal APIs + Razor Pages
- **Hangfire** — recurring and fire-and-forget job scheduling (SQLite storage)
- **EF Core** — SQLite with migrations for health check history, CI runs, and watched dismissals
- **HTMX** — live dashboard updates via partial view polling
- **Chart.js + Luxon** — response time trend visualization
- **Serilog** — structured logging with rolling file sink
- **xUnit + Moq + FluentAssertions** — unit tests
- **GitHub Actions** — CI on Ubuntu, deploy via self-hosted runner on Mac

## Architecture

```
[Browser] ← HTMX 30s polling → [Razor Pages + Minimal APIs]
                                          ↓
                                    [Hangfire Jobs]
                                 ↙     ↓      ↘
                          [HTTP APIs] [SSH]  [SQLite]
                           ↓    ↓      ↓       ↓
                    Plex  Radarr  Mac   PC   homelab.db
                    Sonarr Prowlarr  ↓    ↓   hangfire.db
                    Seerr         df  mullvad
                    Tautulli    docker  qbit
                    Syncthing  colima  torrents
```

The app runs as a LaunchAgent on the Mac. When on macOS, shell commands execute locally. When running on Windows (dev), they SSH to the Mac. qBittorrent is accessed by SSH-ing to the PC and running curl locally (since qBit is bound to the VPN interface and unreachable over LAN).

### Project layout

```
src/homelabBackend/
├── Program.cs                  — composition root, under 30 lines
├── Startup/                    — extension methods called from Program.cs
│   ├── ConfigurationRegistrations.cs   (IOptions<T> bindings)
│   ├── PersistenceRegistrations.cs     (DbContext + Hangfire)
│   ├── InfrastructureRegistrations.cs  (SSH, HTTP, Email, Notifications)
│   ├── JobRegistrations.cs             (Hangfire job DI)
│   ├── DashboardRegistrations.cs       (Razor + Hangfire dashboard)
│   ├── StatusEndpoints.cs              (read-only /api/*)
│   ├── ActionEndpoints.cs              (mutating /api/actions/*)
│   └── RecurringJobSchedule.cs         (cron wiring)
├── Jobs/                       — one file per Hangfire job + OnDemandActions
├── Infrastructure/             — SSH, HTTP, Email, byte formatting
│   └── Notifications/          — router, channels, severity enum
├── Data/                       — EF Core entities + HomelabDbContext
├── Models/
│   ├── Configuration/          — IOptions<T> config types
│   ├── Dtos/                   — external API + internal domain types
│   └── ViewModels/             — Razor page models
├── Migrations/                 — auto-generated EF migrations
├── Pages/                      — Razor pages, each with its own ViewModels/ folder
│   ├── Dashboard/              — tabs + one partial per card
│   ├── Media/                  — library table, row partial, bulk selection
│   ├── Apps/  Architecture/  Uptime/
│   └── Shared/                 — _Layout, _Card, NavigationModel
└── appsettings.json            — tracked placeholders; real keys in Development.json
```

## Configuration

All values are configurable via `appsettings.json`:

| Section | What it controls                                                           |
|---------|----------------------------------------------------------------------------|
| `HomelabServices` | Service URLs, health endpoints, API keys, per-service alert behavior       |
| `QBittorrent` | qBit URL as seen from the PC (commands run there over SSH), optional `DirectBaseUrl` for the app's own HTTP calls (the Mac's loopback relay), credentials |
| `DiskCheck` | Drives to monitor, SSH hosts, warning/critical thresholds                  |
| `Syncthing` | API URL, key, folder ID                                                    |
| `Pipeline` | SSH hosts (Mac + PC), paths, Plex library section IDs                      |
| `Notifications` | Gmail SMTP credentials + reminder and recovery-grace intervals             |
| `Ntfy` | Self-hosted ntfy base URL, topic, optional auth token                      |
| `Tautulli` | Base URL + API key for Plex watch-history lookups                          |
| `Bazarr` | Base URL + API key for subtitle-gap scanning                               |
| `Immich` | Base URL + API key for photo library stats                                 |
| `Overseerr` | Seerr base URL + API key (section keeps the Overseerr name), stuck-threshold hours, page size |
| `StaleMedia` | Unwatched-threshold days, auto-unmonitor toggle                            |
| `TunnelHealth` | List of Cloudflare Tunnel endpoints to probe                               |
| `BackupHealth` | SSH host, backups root path, warning/critical age, expected artifacts list |
| `VpnWatchdog` | Enable toggle, reconnect cooldown minutes                                  |
| `Remediation` | Enable toggle, lookback minutes, per-pattern rules (check → action)        |
| `LibraryIntegrity` | SSH host, Movies/TV root paths, remediation toggle + folder floors          |
| `GmailCleanup` | IMAP credentials, archive-age, label rules                                 |
| `GitHub` | Owner, repo, PAT and page size for the CI status card                            |
| `JobWatchdog` | Enable toggle, check interval, staleness threshold, startup grace          |
| `TwoMinds` | Album folder path, player URL, PC host + always-on probe port, silence warning/critical minutes, failure count + window, deployed.json path + stale hours + lag-critical minutes |
| `AppsLauncher` | Groups and entries for the Apps launcher grid                              |
| `JobSchedules` | Cron expressions for all recurring jobs                                    |

Secrets go in `appsettings.Development.json` (gitignored). The tracked `appsettings.json` uses placeholder values.

## Deployment

Push to `main` triggers the CI/CD pipeline:
1. **CI** (Ubuntu) — restore, build, test
2. **Deploy** (self-hosted Mac runner) — only runs if CI passes, publishes ARM64 binary, deploys via rsync (preserves config + DB), restarts LaunchAgent
3. **Release** (Ubuntu) — also gated on green CI, cuts the tagged release

`CiStatusJob` reads all three back out of the GitHub API, so the dashboard shows whether the last push landed without leaving the app.
