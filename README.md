# Jarvis — Self-Hosted Home Server Stack

A complete guide to building a production-grade home server from an old laptop running Ubuntu Server, Docker Swarm, and Dokploy — exposed securely to the internet via Cloudflare Tunnel with zero open ports.

> **Note:** All domain names, usernames, and service URLs in this guide use `johndoe.dev` and `johndoe` as placeholders. Replace them with your own domain and username throughout.

**Services running on this stack:**

- 🌐 `johndoe.dev` — Personal portfolio
- 📸 `photos.johndoe.dev` — Immich (private Google Photos alternative)
- 🎬 `stream.johndoe.dev` — Jellyfin (private media server)
- ⚙️ `panel.johndoe.dev` — Dokploy dashboard
- 🧪 `app.johndoe.dev` — Full-stack web app (Next.js + FastAPI + MongoDB)
- 📡 `status.johndoe.dev` — Live service status API (feeds Android home screen widget)

---

## 📐 Architecture Overview

```mermaid
flowchart TD
    Internet(["🌐 Internet"])

    subgraph CF["Cloudflare Edge"]
        DNS["DNS + WAF + HTTPS"]
        Tunnel["Zero Trust Tunnel\n(cloudflared)"]
    end

    subgraph Jarvis["Ubuntu Server — jarvis (Laptop)"]
        Traefik["Traefik\nReverse Proxy :80"]

        subgraph Swarm["Docker Swarm"]
            Portfolio["Portfolio\n:3000"]
            Dokploy["Dokploy\n:3000"]
            Immich["Immich\n:2283"]
            Jellyfin["Jellyfin\n:8096"]
            app["app\n:3000"]
            Status["Status Server\n:3001"]
        end

        HDD["/mnt/external-media\n(HDD, read-only)"]
    end

    Internet --> CF
    DNS --> Tunnel --> Traefik
    Traefik --> Portfolio
    Traefik --> Immich
    Traefik --> Jellyfin
    Traefik --> app
    Traefik --> Status
    Tunnel -->|":3000 direct"| Dokploy
    Immich --> HDD
    Jellyfin --> HDD
```

**Key design principle:** The only thing touching the public internet is the Cloudflare Tunnel agent (`cloudflared`). No ports are forwarded on the router. All internal services communicate over Docker's virtual bridge network.

---

## 🗂️ Table of Contents

1. [Prerequisites](#prerequisites)
2. [Phase 1 — Ubuntu Server Setup](#phase-1--ubuntu-server-setup)
3. [Phase 2 — Static IP & Network Config](#phase-2--static-ip--network-config)
4. [Phase 3 — Docker & Dokploy](#phase-3--docker--dokploy)
5. [Phase 4 — Cloudflare Domain & Tunnel](#phase-4--cloudflare-domain--tunnel)
6. [Phase 5 — Deploying Apps via Dokploy](#phase-5--deploying-apps-via-dokploy)
7. [Phase 6 — External HDD Setup](#phase-6--external-hdd-setup)
8. [Phase 7 — Immich (Self-Hosted Photos)](#phase-7--immich-self-hosted-photos)
9. [Phase 8 — Jellyfin (Media Server)](#phase-8--jellyfin-media-server)
10. [Phase 9 — app (Full-Stack App)](#phase-9--app-full-stack-app)
11. [Reliability & Boot Configuration](#reliability--boot-configuration)
12. [Phase 10 — Secure Remote SSH via Cloudflare Tunnel](#phase-10--secure-remote-ssh-via-cloudflare-tunnel)
13. [Phase 11 — Android Status Widget](#phase-11--android-status-widget)
14. [Phase 12 — Resilient HDD Mount (Emergency Mode Fix)](#phase-12--resilient-hdd-mount-emergency-mode-fix)
15. [Phase 13 — Docker Boot Race Condition Fix](#phase-13--docker-boot-race-condition-fix)
16. [Phase 14 — Remote Commands Screen (Phone App)](#phase-14--remote-commands-screen-phone-app)
17. [Troubleshooting](#troubleshooting)

---

## Prerequisites

| Item                    | Notes                                                  |
| ----------------------- | ------------------------------------------------------ |
| Old laptop (server)     | Any x86-64 machine works. Mine has 8GB RAM, 256GB SSD. |
| Personal PC             | For flashing the USB and SSH access                    |
| USB drive (≥4 GB)       | For the Ubuntu Server installer                        |
| Domain name             | Registered anywhere; we'll delegate DNS to Cloudflare  |
| Cloudflare account      | Free tier is sufficient for everything in this guide   |
| External HDD (optional) | For media libraries (Immich / Jellyfin)                |

---

## Phase 1 — Ubuntu Server Setup

### 1.1 Flash the Installer USB

On your personal PC, download:

- [Ubuntu Server 24.04 LTS ISO](https://ubuntu.com/download/server)
- [Balena Etcher](https://etcher.balena.io/)

Open Balena Etcher → **Flash from file** → select the ISO → select your USB drive → **Flash**.

### 1.2 Install Ubuntu Server

Boot the server laptop from USB (enter BIOS/boot menu, usually `F12` or `Esc`):

| Stage             | Choice                                                                                        |
| ----------------- | --------------------------------------------------------------------------------------------- |
| Language          | English (or your preference)                                                                  |
| Keyboard layout   | Default detected layout                                                                       |
| Installation type | Ubuntu Server (not minimized)                                                                 |
| Network           | Connect to Wi-Fi now; configure static IP later                                               |
| Storage           | **Uncheck "Set up this disk as an LVM group"** — plain partition is simpler for a home server |
| Profile           | Full name, server name (`jarvis`), username (`johndoe`), password                             |
| SSH               | **Install OpenSSH server** — critical for remote management                                   |
| Featured snaps    | Skip everything                                                                               |

When prompted, remove the USB and press `Enter` to reboot. Log in with your credentials.

---

## Phase 2 — Static IP & Network Config

### 2.1 Keep the Server Awake with the Lid Closed

```bash
sudo nano /etc/systemd/logind.conf
```

Find and set these four lines (uncomment them if needed):

```ini
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
LidSwitchIgnoreInhibited=no
```

```bash
sudo reboot
```

### 2.2 Fix Wi-Fi SSID with Spaces (Common Issue)

If your Wi-Fi name contains a space, Ubuntu's default netplan config may fail silently. Check and fix:

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

```yaml
network:
  version: 2
  wifis:
    wlp1s0:
      access-points:
        "My WiFi Name": # Quotes handle spaces correctly
          password: "yourpassword"
      dhcp4: true
```

Apply and verify:

```bash
sudo netplan apply
hostname -I    # Should now show an IP address
```

### 2.3 Assign a Static IP

Find your router's gateway IP:

```bash
ip route show
# Output example: default via 192.168.1.1 dev wlp1s0
```

Edit your netplan config to pin the server to a fixed address:

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

```yaml
network:
  version: 2
  renderer: networkd
  wifis:
    wlp1s0:
      dhcp4: no
      addresses:
        - 192.168.1.100/24 # Your chosen static IP
      routes:
        - to: default
          via: 192.168.1.1 # Your router's IP
      nameservers:
        addresses: [1.1.1.1, 8.8.8.8]
      access-points:
        "My WiFi Name":
          password: "yourpassword"
```

```bash
sudo netplan try    # Preview (auto-reverts after 120s if broken)
sudo netplan apply  # Commit the config
```

---

## Phase 3 — Docker & Dokploy

### 3.1 Install Docker

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
newgrp docker

# Verify
docker version
docker compose version
docker ps
```

### 3.2 Install Dokploy

[Dokploy](https://dokploy.com) is an open-source PaaS that runs on top of Docker Swarm. It gives you a Heroku-like dashboard for deploying apps from GitHub, managing environment variables, routing domains, and viewing logs — all self-hosted.

```bash
mkdir -p ~/services && cd ~/services
curl -sSL https://dokploy.com/install.sh | sudo sh

# Verify Swarm services are running
docker service ls
```

Access the dashboard at `http://192.168.1.100:3000` and create your admin account.

---

## Phase 4 — Cloudflare Domain & Tunnel

### 4.1 Register a Domain & Add to Cloudflare

1. Buy a domain from any registrar (Namecheap, Porkbun, etc.)
2. Create a free [Cloudflare account](https://cloudflare.com)
3. In Cloudflare, click **Add a site** and enter your domain
4. Cloudflare will provide two nameservers (e.g., `ada.ns.cloudflare.com`)
5. Go to your registrar's DNS settings and **replace** the existing nameservers with Cloudflare's two

Propagation takes 5–30 minutes.

### 4.2 Create a Zero Trust Tunnel

This is the core of the secure architecture — no router port forwarding required.

1. Go to **Cloudflare Zero Trust** → **Networks** → **Tunnels**
2. Click **Create a tunnel** → Name it `homeserver`
3. Select **Cloudflared** as the connector type
4. Copy the install command with your token

### 4.3 Install the Tunnel Agent on the Server

```bash
curl -L --output cloudflared.deb \
  https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb

sudo dpkg -i cloudflared.deb

# Paste your full command from the Cloudflare dashboard:
sudo cloudflared service install YOUR_SECRET_TOKEN_HERE

# Verify it's running
sudo systemctl status cloudflared
```

### 4.4 Add Public Hostnames to the Tunnel

In **Zero Trust → Networks → Tunnels** → click your tunnel → **Public Hostnames** → **Add**:

| Subdomain | Domain        | Type | URL              |
| --------- | ------------- | ---- | ---------------- |
| `panel`   | `johndoe.dev` | HTTP | `localhost:3000` |
| `photos`  | `johndoe.dev` | HTTP | `127.0.0.1:80`   |
| `stream`  | `johndoe.dev` | HTTP | `127.0.0.1:80`   |
| `app`     | `johndoe.dev` | HTTP | `127.0.0.1:80`   |
| _(blank)_ | `johndoe.dev` | HTTP | `127.0.0.1:80`   |

> **Why `127.0.0.1:80`?** Dokploy runs Traefik as a global reverse proxy listening on port 80. When you assign a domain to an app inside Dokploy, Traefik reads the `Host:` header and routes the request to the correct container internally. So every subdomain except the Dokploy panel itself points to Traefik.

---

## Phase 5 — Deploying Apps via Dokploy

### 5.1 Connect GitHub

In the Dokploy dashboard, go to **Settings → Providers** and connect your GitHub account.

### 5.2 Deploy an App (e.g., Portfolio)

1. **Create Project** → name it `portfolio`
2. Inside the project, **Create Service** → **Application**
3. Select **GitHub** as provider, choose your repository and branch
4. Go to **Domains** tab → **Add Domain**:

| Field          | Value                                               |
| -------------- | --------------------------------------------------- |
| Host           | `johndoe.dev`                                       |
| Container Port | `3000` (or your app's internal port)                |
| HTTPS / SSL    | **Disabled** _(Cloudflare handles TLS at the edge)_ |

5. Go to **General** → **Deploy**

### 5.3 WWW Redirect via Cloudflare Rules

Rather than deploying a second container to handle `www`, use a Cloudflare Redirect Rule (free):

**Cloudflare Dashboard → Rules → Redirect Rules → Create rule:**

```
Rule name:  WWW to Root Redirect
Field:      Hostname
Operator:   equals
Value:      www.johndoe.dev

Then... URL redirect
Type:       Dynamic
Expression: concat("https://johndoe.dev", http.request.uri.path)
Status:     301 (Permanent)
```

---

## Phase 6 — External HDD Setup

### 6.1 Identify the Drive

```bash
lsblk -f
```

Example output:

```
NAME   FSTYPE   LABEL      SIZE  MOUNTPOINT
sdb                          1T
└─sdb1 ntfs     MediaDrive 931G
```

Note the partition name (`sdb1`) and filesystem type (`ntfs` or `exfat`).

### 6.2 Install Filesystem Drivers

```bash
# For NTFS (Windows-formatted drives)
sudo apt update && sudo apt install -y ntfs-3g

# For exFAT (common on external USB drives)
sudo apt install -y exfat-fuse exfat-utils
```

### 6.3 Create Mount Point

```bash
sudo mkdir -p /mnt/external-media
```

### 6.4 Configure Auto-Mount on Boot (fstab)

Get the partition's UUID:

```bash
sudo blkid /dev/sdb1
# UUID="E1A2-B3C4" (exFAT) or UUID="E1A2B3C4D5E6F7A8" (NTFS)
```

Open the filesystem table:

```bash
sudo nano /etc/fstab
```

Add one of the following lines at the bottom:

**NTFS:**

```
UUID=YOUR-UUID-HERE  /mnt/external-media  ntfs-3g  defaults,nls=utf8,uid=1000,gid=1000,dmask=022,fmask=133  0  0
```

**exFAT:**

```
UUID=YOUR-UUID-HERE  /mnt/external-media  exfat  defaults,uid=1000,gid=1000,dmask=022,fmask=133  0  0
```

> `uid=1000,gid=1000` maps the drive's ownership to your primary user account. This is critical — without it, Docker containers won't be able to read the files.

Apply immediately:

```bash
sudo mount -a
ls /mnt/external-media    # Should show your files
```

---

## Phase 7 — Immich (Self-Hosted Photos)

Immich is a self-hosted alternative to Google Photos with AI-powered facial recognition, object detection, and automatic mobile backup.

We deploy it as a **Docker Swarm Stack** via Dokploy and mount the external HDD in **read-only mode** so Immich can index existing files without moving or modifying them.

### 7.1 Create Stack in Dokploy

**Dokploy → Create Project** (`immich`) → **Create Service → Compose** → **Name:** `immich-stack` → **Type: Stack**

### 7.2 Compose Configuration

Paste into the **Compose** tab:

```yaml
version: "3.8"

services:
  immich-server:
    image: ghcr.io/immich-app/immich-server:release
    volumes:
      - immich-upload:/usr/src/app/upload
      - /mnt/external-media:/external-media:ro
    env_file:
      - stack.env
    ports:
      - "2283:2283"
    depends_on:
      - redis
      - database
    restart: always

  immich-machine-learning:
    image: ghcr.io/immich-app/immich-machine-learning:release
    volumes:
      - immich-model-cache:/cache
    env_file:
      - stack.env
    restart: always

  redis:
    image: docker.io/redis:6.2-alpine
    restart: always

  database:
    image: docker.io/tensorchord/pgvecto-rs:pg16-v0.2.0
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_USER: ${DB_USERNAME}
      POSTGRES_DB: ${DB_DATABASE_NAME}
      POSTGRES_INITDB_ARGS: "--data-checksums"
    volumes:
      - immich-postgres:/var/lib/postgresql/data
    restart: always

volumes:
  immich-upload:
  immich-model-cache:
  immich-postgres:
```

### 7.3 Environment Variables

In the **Environment Variables** tab:

```env
DB_PASSWORD=changeme_secure_password
DB_USERNAME=immich
DB_DATABASE_NAME=immich
DB_HOSTNAME=database
REDIS_HOSTNAME=redis
IMMICH_VERSION=release
```

### 7.4 Domain Routing

**Domains tab** inside `immich-stack`:

| Field          | Value                |
| -------------- | -------------------- |
| Service        | `immich-server`      |
| Host           | `photos.johndoe.dev` |
| Container Port | `2283`               |
| HTTPS          | Disabled             |

Deploy → then in Cloudflare Tunnel ensure `photos.johndoe.dev → http://127.0.0.1:80` is set.

### 7.5 Add External Library

After the setup wizard completes at `https://photos.johndoe.dev`:

1. **Avatar → Administration → External Libraries → Create Library**
2. Set owner to your admin account
3. **Add folder path:** `/external-media`
4. Three-dot menu → **Scan New Library Files**

---

## Phase 8 — Jellyfin (Media Server)

Jellyfin is a self-hosted media server for streaming your personal movie and TV collection from any device.

### 8.1 Create Stack in Dokploy

**Create Service → Compose** → **Name:** `jellyfin-stack` → **Type: Stack**

### 8.2 Compose Configuration

```yaml
version: "3.8"

services:
  jellyfin:
    image: jellyfin/jellyfin:latest
    volumes:
      - jellyfin-config:/config
      - jellyfin-cache:/cache
      - /mnt/external-media:/media:ro
    ports:
      - "8096:8096"
    restart: always

volumes:
  jellyfin-config:
  jellyfin-cache:
```

### 8.3 Domain Routing

| Field          | Value                |
| -------------- | -------------------- |
| Service        | `jellyfin`           |
| Host           | `stream.johndoe.dev` |
| Container Port | `8096`               |
| HTTPS          | Disabled             |

Deploy, then verify:

```bash
docker service ls
# jellyfin-stack_jellyfin should show 1/1 replicas
```

### 8.4 Add Media Library

At `https://stream.johndoe.dev`:

1. Create admin account → proceed through setup wizard
2. **Add Media Library** → choose type (Movies / TV Shows)
3. Folder path: `/media`
4. Complete wizard

---

## Phase 9 — App (Full-Stack App)

A full-stack web application with a Next.js frontend, FastAPI Python backend, and MongoDB database — all running on the internal Docker network with only the frontend exposed publicly.

### Architecture

```mermaid
flowchart LR
    Browser(["Browser"])

    subgraph Public
        CF["Cloudflare\njohndoe.dev"]
    end

    subgraph Jarvis["Docker Internal Network"]
        FE["app-frontend\nNext.js :3000"]
        BE["app-backend\nFastAPI :8000"]
        DB[("app-db\nMongoDB :27017")]
    end

    Browser --> CF --> FE
    FE -->|"internal only"| BE
    BE -->|"internal only"| DB
```

**This is a BFF (Backend-for-Frontend) pattern:**

- ✅ The Python API and MongoDB are never exposed to the internet
- ✅ Zero CORS issues — the browser only ever talks to the Next.js domain
- ✅ Container-to-container traffic stays on the local virtual network

### 9.1 Create Project & Services

**Dokploy → Create Project** (`app`)

Create three services inside:

| Service Name   | Type               |
| -------------- | ------------------ |
| `app-db`       | Database (MongoDB) |
| `app-backend`  | Application        |
| `app-frontend` | Application        |

### 9.2 Provision MongoDB

**Create Service → Database → MongoDB** → name it `app-db`

Leave external port **blank** (internal only). After deploy, copy the **Internal Connection String** from the database dashboard (format: `mongodb://user:pass@app-db:27017`).

### 9.3 Deploy FastAPI Backend

Connect to your Python backend GitHub repo.

**Environment Variables:**

```env
MONGO_URI=mongodb://user:pass@app-db:27017/app
PORT=8000
```

**No domain needed** — this service is internal only. Deploy via **General → Deploy**.

> Dokploy places all services in a project on the same internal Docker network. The service name becomes its internal hostname. Your backend reaches MongoDB at `app-db:27017`.

### 9.4 Deploy Next.js Frontend

Connect to your Next.js GitHub repo.

**Environment Variables:**

```env
NEXT_PUBLIC_API_URL=http://app-backend:8000
```

**Domain:**

| Field          | Value             |
| -------------- | ----------------- |
| Host           | `app.johndoe.dev` |
| Container Port | `3000`            |
| HTTPS          | Disabled          |

Deploy.

---

## Reliability & Boot Configuration

### Ensure Docker Starts on Boot

```bash
sudo systemctl enable docker.service
sudo systemctl enable containerd.service
```

Dokploy's Swarm services are configured with `restart: always`, so once Docker is up, all containers restore themselves automatically.

### Verify Everything is Running

```bash
docker service ls         # All services should show N/N replicas
docker ps                 # Individual container statuses
sudo systemctl status cloudflared   # Tunnel agent health
```

---

## Phase 10 — Secure Remote SSH via Cloudflare Tunnel

Opening port 22 to the public internet is an open invitation for bots to relentlessly brute-force your credentials. Since `cloudflared` is already running on Jarvis, we can route SSH through the same Zero Trust Tunnel — no router port forwarding, no exposed ports.

**How it works:** Cloudflare proxies the terminal connection over encrypted WebSockets. Your home router stays completely locked down. You can SSH into Jarvis from any network — coffee shop, university Wi-Fi, cellular — safely.

### 10.1 Add the SSH Hostname in Cloudflare

1. Go to **Zero Trust → Networks → Tunnels** → click your active tunnel
2. **Public Hostnames → Add a public hostname**

| Field     | Value          |
| --------- | -------------- |
| Subdomain | `ssh`          |
| Domain    | `johndoe.dev`  |
| Type      | `SSH`          |
| URL       | `localhost:22` |

Save.

> Unlike web hostnames which all point to Traefik on port 80, this one points directly to the SSH daemon at port 22. Cloudflare handles the WebSocket wrapping.

### 10.2 Install Cloudflared on Your Client Machine

Because the endpoint is behind Cloudflare, a standard SSH client can't reach it directly — you need the `cloudflared` helper to negotiate the handshake.

**Windows (PowerShell as Administrator):**

```powershell
winget install Cloudflare.cloudflared
```

**macOS:**

```bash
brew install cloudflared
```

**Linux:**

```bash
sudo apt install cloudflared
```

### 10.3 Configure Your Local SSH Config

Rather than typing a proxy command every time, add a one-time entry to your SSH config so the tunnel handshake happens automatically.

**Windows** — open or create `C:\Users\<YourUsername>\.ssh\config`:

```
Host ssh.johndoe.dev
    ProxyCommand cloudflared.exe access ssh --hostname %h
    User johndoe
```

**macOS / Linux** — open or create `~/.ssh/config`:

```
Host ssh.johndoe.dev
    ProxyCommand cloudflared access ssh --hostname %h
    User johndoe
```

### 10.4 Connect From Anywhere

From any terminal on any network:

```bash
ssh ssh.johndoe.dev
```

The client runs `cloudflared` as a proxy, which authenticates through the Cloudflare tunnel, then passes the connection through to Jarvis's SSH daemon. You'll be prompted for your server password and dropped into a bash shell.

---

## Phase 11 — Android Status Widget

A custom Android home screen widget + mobile app that polls the server every 4 minutes and shows the live status of every subdomain — green for online, red for offline. If the status server itself is unreachable, the widget flags the entire stack as down.

### Architecture

```mermaid
sequenceDiagram
    participant W as Android Widget
    participant H as /health
    participant A as /api/status
    participant CF as Cloudflare DNS
    participant S1 as photos.johndoe.dev
    participant S2 as stream.johndoe.dev
    participant SN as ...other subdomains

    W->>H: GET /health (no auth)
    H-->>W: { status: "ONLINE", uptime: 3920 }

    W->>A: GET /api/status?secret=xxx
    Note over A: Check cache (TTL 60s)
    A->>CF: Fetch all CNAME/A records
    CF-->>A: [photos, stream, panel, ...]
    par concurrent probes
        A->>S1: HEAD (fallback GET)
        A->>S2: HEAD (fallback GET)
        A->>SN: HEAD (fallback GET)
    end
    A-->>W: { "stream": "OFFLINE", "photos": "ONLINE", ... }
    Note over W: Re-render widget\nOffline services shown first
```

### How the Status Server Works

The backend (`status-server/index.js`) is a lightweight Node.js/Express app deployed on Jarvis. On each request to `/api/status` it:

1. **Fetches all subdomains automatically** from the Cloudflare API — no manual list to maintain. Add a new service and it appears in the widget on the next refresh.
2. **Probes each subdomain concurrently** with an HTTP HEAD request, falling back to a `GET` with `Range: bytes=0-0` if HEAD isn't supported.
3. **Retries twice** before marking a service offline, with an 800ms delay between attempts.
4. **Detects Cloudflare-specific failure modes** — including error 1033 (tunnel unreachable, served as a 530 HTML page) which a naive status checker would misread as a 200 OK.
5. **Caches results for 60 seconds** so rapid widget refreshes don't hammer the server.
6. **Sorts the response** — offline services bubble to the top.

A separate unauthenticated `/health` endpoint lets the widget distinguish between "server is up but a service is down" vs. "the status server itself is unreachable."

### Endpoints

| Endpoint                       | Auth                                   | Purpose                                                         |
| ------------------------------ | -------------------------------------- | --------------------------------------------------------------- |
| `GET /health`                  | None                                   | Liveness check — widget uses this to detect total server outage |
| `GET /api/status`              | `?secret=` or `X-Widget-Secret` header | Full subdomain status report                                    |
| `GET /api/status?refresh=true` | Same                                   | Force bypass cache and re-probe immediately                     |

### Status Server Deployment

Deploy it as an **Application** service inside Dokploy:

1. **Create Service → Application**, name it `status-server`
2. Connect to the GitHub repository containing `index.js`
3. **Environment Variables:**

```env
PORT=3001
CLOUDFLARE_API_TOKEN=your_cf_api_token_here
CLOUDFLARE_ZONE_ID=your_zone_id_here
DOMAIN=johndoe.dev
SECRET_KEY=a_long_random_secret_for_the_widget
```

4. **Domains tab:**

| Field          | Value                |
| -------------- | -------------------- |
| Host           | `status.johndoe.dev` |
| Container Port | `3001`               |
| HTTPS          | Disabled             |

5. Add `status` as a Public Hostname in your Cloudflare Tunnel (`localhost:3001`, type HTTP)
6. Deploy

### Sample API Response

```json
{
  "stream": "OFFLINE",
  "app": "ONLINE",
  "panel": "ONLINE",
  "photos": "ONLINE",
  "_meta": {
    "cached": true,
    "cacheAge": 34,
    "nextRefresh": 26
  }
}
```

Offline services sort to the top so the widget can render them prominently without any extra client-side logic.

### Android App & Widget

The mobile client is built with **Expo + React Native** and ships two surfaces: a full in-app status dashboard with a **Status** tab and a **Commands** tab (see Phase 14), and a native Android home screen widget that updates in the background every 4 minutes.

#### App structure

| File                                 | Purpose                                                      |
| ------------------------------------ | ------------------------------------------------------------ |
| `api.ts`                             | Network layer — health check, status fetch, command dispatch |
| `App.tsx`                            | Root screen with tab bar (Status / Commands)                 |
| `widgets/HomeServerHealthWidget.tsx` | React Native Android widget component                        |
| `page/CommandScreen.tsx`             | Commands tab UI                                              |
| `theme.ts`                           | Shared design tokens (colours, spacing, radii)               |
| `background-task.ts`                 | Registers the `expo-background-fetch` task                   |

#### `api.ts` — two-stage fetch with server-down detection

Before requesting service statuses, `fetchServerData()` calls the unauthenticated `/health` endpoint first. If that times out (4 s), the function short-circuits and returns a `serverDown: true` payload without hitting `/api/status` at all — so the widget shows "Unreachable" rather than stale data or a misleading partial result.

```
fetchServerData()
  ├── GET /health  (4s timeout, no auth)
  │    └── unreachable → return { serverDown: true }
  └── GET /api/status?secret=xxx  (10s timeout)
       ├── strip _meta keys
       ├── normalise each value → "ONLINE" | "OFFLINE" | "UNKNOWN"
       └── return ServerData { services, onlineCount, totalCount, ... }
```

`sendCommand()` shares the same base URL, using a 20 s timeout to accommodate slow operations like Docker daemon restarts.

Environment variables expected in `.env`:

```env
EXPO_PUBLIC_STATUS_URL=https://status.johndoe.dev/api/status
EXPO_PUBLIC_STATUS_SECRET=your_widget_secret_here
EXPO_PUBLIC_SERVER_NAME=Jarvis
```

#### `HomeServerHealthWidget.tsx` — home screen widget

Built with `react-native-android-widget`. Design decisions worth noting:

- **Offline-first sort** — services are ranked `OFFLINE → UNKNOWN → ONLINE` before rendering, so the things that need attention are always visible even when the list is truncated to the widget's 8-item cap.
- **Uptime bar** — a slim 4 px proportional bar below the header gives an immediate at-a-glance read on the online ratio before the service list is even scanned.
- **Stale badge** — if `minutesSinceCheck > 15`, a yellow `STALE` label appears in the footer so you know the data is old rather than assuming the server is healthy.
- **Server-down vs. service-down** — the header colour and summary text distinguish between "the status server itself is unreachable" (amber, "Unreachable") and "server is up but services are degraded" (rose, "N services need attention").
- **`clickAction="REFRESH"`** on the footer button wires back to the widget task handler, which re-runs the health check and calls `requestWidgetUpdate` to re-render.

#### `App.tsx` — in-app dashboard

The app mirrors the widget's visual language (same colour tokens from `theme.ts`) so both surfaces feel like one product. The root screen renders:

1. A header with server name and the same online/offline badge as the widget
2. A tab bar switching between **Status** (live service list) and **Commands** (Phase 14)
3. A **Refresh Now** button that re-fetches and also pushes the fresh data to the home screen widget via `requestWidgetUpdate`

Background polling is registered via `expo-background-fetch` on first mount so the widget stays current even when the app is closed.

---

## Phase 12 — Resilient HDD Mount (Emergency Mode Fix)

By default, Linux treats your external drive as a critical system dependency. If the cable is loose or the drive isn't detected during startup, the kernel panics and drops Jarvis into Emergency Mode — taking down every service with it.

Two `fstab` flags fix this permanently:

- **`nofail`** — If the drive isn't present at boot, skip it and finish starting up normally. No Emergency Mode.
- **`x-systemd.automount`** — Don't try to mount the drive eagerly at boot. Instead, wait until a process (Immich, Jellyfin, a container) actually tries to read `/mnt/external-media`, then mount on the fly.

Together they create a self-healing setup: if the cable gets jiggled loose and reconnected, systemd remounts the drive automatically the moment any container next requests a file — no manual terminal rescue required.

### 12.1 Update `/etc/fstab`

```bash
sudo nano /etc/fstab
```

Find your external media line at the bottom. It currently looks something like this:

```
UUID=<your-uuid-here>  /mnt/external-media  ext4  defaults,noatime  0  2
```

Append `,nofail,x-systemd.automount` to the options column (no spaces inside the comma-separated list):

```
UUID=<your-uuid-here>  /mnt/external-media  ext4  defaults,noatime,nofail,x-systemd.automount  0  2
```

Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X`).

### 12.2 Reload systemd

```bash
# Unmount the drive if it's currently locked in manually
sudo umount /mnt/external-media

# Tell systemd to process the fresh automount directives
sudo systemctl daemon-reload
sudo systemctl restart local-fs.target
```

### 12.3 Test the Automount

Run `lsblk` — even with the drive physically connected, the `MOUNTPOINT` column may show blank. That's expected: automount waits for a real access request.

```bash
lsblk
```

Now trigger a mount by listing the directory:

```bash
ls /mnt/external-media
```

The terminal will pause for a fraction of a second while systemd wakes the USB hardware channel and hooks the filesystem path, then your files appear. Run `lsblk` again and `/mnt/external-media` will have snapped back into place.

**What this changes going forward:**

| Scenario                           | Before                  | After                                |
| ---------------------------------- | ----------------------- | ------------------------------------ |
| Boot with cable unplugged          | Emergency Mode 🔴       | Normal boot, drive skipped ✅        |
| Boot with cable plugged in         | Normal boot ✅          | Normal boot, drive lazy-mounted ✅   |
| Cable drops at runtime, reconnects | Manual `mount` required | Auto-remounts on next file access ✅ |

---

## Phase 13 — Docker Boot Race Condition Fix

When Jarvis restarts, Docker often fires up faster than the USB hardware controller finishes initializing. Containers like Immich and Jellyfin look at `/mnt/external-media` immediately on start, find an empty folder (the drive hasn't mounted yet), and fail silently or misbehave.

Forcing Docker to restart ~30 seconds after boot solves this race condition.

### 13.1 The Problem with a Naive Cron Job

A plain `@reboot sleep 15 && systemctl restart docker` fails silently in cron. The reason: cron runs in a stripped environment with almost no `$PATH`. When it tries to run `sleep` or `systemctl`, it has no idea where those binaries live and exits with a silent "command not found" error.

The fix is to use absolute binary paths and add logging so you can diagnose any future issues.

### 13.2 Harden the Root Crontab

```bash
sudo crontab -e
```

Remove any existing reboot line and replace it with:

```
@reboot /bin/sleep 30 && /bin/systemctl restart docker >> /var/log/docker_reboot.log 2>&1
```

What each part does:

| Part                                 | Purpose                                                           |
| ------------------------------------ | ----------------------------------------------------------------- |
| `/bin/sleep 30`                      | Explicit path — cron-safe. 30s gives USB hardware time to settle. |
| `/bin/systemctl restart docker`      | Explicit path — cron-safe. Restarts Docker after the delay.       |
| `>> /var/log/docker_reboot.log 2>&1` | Logs all output (success or error) for diagnosis.                 |

Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X`).

### 13.3 Verify After Next Reboot

After restarting Jarvis (`sudo reboot`), wait ~45 seconds, then SSH back in and check the log:

```bash
cat /var/log/docker_reboot.log
```

- **Empty/blank** — the command ran cleanly and restarted Docker without complaint.
- **Any output** — the log will show exactly what systemd complained about.

Cross-check with container uptime to confirm the restart fired:

```bash
docker ps
# Immich and Jellyfin should show "Up Less than a minute" or similar fresh uptime
```

---

## Troubleshooting

### Wi-Fi not connecting after reboot

The most common cause is a space in the SSID not being quoted in netplan. See [Phase 2.2](#22-fix-wi-fi-ssid-with-spaces-common-issue).

### `docker ps` is empty after power loss

Docker service may not be set to start on boot. Run the commands in [Reliability & Boot Configuration](#reliability--boot-configuration).

### External HDD not mounting / Docker containers can't read files

Confirm `uid=1000` and `gid=1000` are set in your `/etc/fstab` entry. Run `id` to verify your user's UID is `1000`.

```bash
id johndoe
# uid=1000(johndoe) gid=1000(johndoe)
```

### Immich external library shows no files

Verify the volume path inside the container. The compose file maps `/mnt/external-media` on the host to `/external-media` inside the container. When adding the folder in Immich's admin panel, use `/external-media` (the container path, not the host path).

### Subdomain returns 502 Bad Gateway

1. Check the service is running: `docker service ls`
2. Confirm the container port in Dokploy's Domains tab matches the port your app actually listens on
3. Confirm the Cloudflare Tunnel hostname points to `http://127.0.0.1:80` (not directly to the container port)

---

## Phase 14 — Remote Commands Screen (Phone App)

An extension of the Android app that adds a **Commands** tab alongside the status widget. From your phone you can run live diagnostics (`docker ps`, `df -h`, `free -h`) and trigger host-level operations (restart Docker daemon, reboot Jarvis) — all over the same authenticated Cloudflare tunnel, from anywhere in the world.

### Architecture

```mermaid
flowchart LR
    Phone(["📱 Phone App"])

    subgraph StatusServer["status-server (Docker container)"]
        API["/api/command"]
        ContainerCmds["Container-level commands\ndocker ps, df -h, free -h\ndocker restart &lt;name&gt;"]
    end

    subgraph Host["Jarvis Host (systemd)"]
        Agent["host-agent\nNode.js :7070"]
        HostCmds["Host-level commands\nsystemctl restart docker\nreboot"]
    end

    Phone -->|"POST /api/command\n?secret=xxx"| API
    API -->|"runs inside container\nvia child_process"| ContainerCmds
    API -->|"POST 172.18.0.1:7070/run\nx-agent-secret header\n(docker_gwbridge)"| Agent
    Agent --> HostCmds
```

**Why two layers?** The status server runs inside a Docker container and can execute most commands directly. But some operations — like restarting the Docker daemon itself or rebooting the host — are impossible from inside a container. The host agent runs as a systemd service directly on Jarvis and handles only those privileged operations, reachable from containers via the `docker_gwbridge` interface at `172.18.0.1`.

### What didn't work (and why)

Many obvious approaches fail here. This table is useful to understand the constraints:

| Approach                                                        | Why it failed                                        |
| --------------------------------------------------------------- | ---------------------------------------------------- |
| `nsenter -t 1` inside container                                 | Container not privileged enough                      |
| `systemctl` inside container                                    | Alpine Linux has no systemd                          |
| Docker socket to restart Docker daemon                          | Can't restart the daemon managing your own container |
| Agent bound to `127.0.0.1`                                      | Loopback — not reachable from inside containers      |
| Agent IP `10.0.1.1` (Swarm gateway)                             | Virtual IP, not assigned to any real interface       |
| Host-network Docker container for agent                         | Still couldn't bridge the gap                        |
| **`172.18.0.1` (docker_gwbridge) + systemd agent on `0.0.0.0`** | ✅ **This is the solution**                          |

The `docker_gwbridge` interface is the one real host IP that Docker containers can reliably reach. Binding the agent to `0.0.0.0` on the host and calling it at `172.18.0.1:7070` from inside a container is the correct pattern for host-level privileged operations.

Find your own `docker_gwbridge` IP with:

```bash
ip addr show | grep "172.18"
# docker_gwbridge: 172.18.0.1
```

---

### Part 1 — Host Agent

A minimal Node.js HTTP server that runs as a systemd service on the host. It accepts only a fixed allowlist of commands and responds before executing, so the client gets an acknowledgement even if Docker restarts mid-request.

#### 1.1 Create the agent

```bash
mkdir -p /opt/host-agent && cd /opt/host-agent
npm init -y
nano agent.js
```

```js
const http = require("http");
const { exec } = require("child_process");

const PORT = 7070;
const SECRET = process.env.AGENT_SECRET;

const ALLOWED = {
  // Restarts the Docker daemon by stopping and restarting all containers
  // (true daemon restart isn't possible from inside Docker)
  "docker-daemon-restart": "docker restart $(docker ps -q)",
  reboot: "reboot",
};

http
  .createServer((req, res) => {
    if (req.method !== "POST" || req.url !== "/run") {
      return res.writeHead(404).end();
    }

    const secret = req.headers["x-agent-secret"];
    if (secret !== SECRET) {
      return res.writeHead(403).end(JSON.stringify({ error: "Unauthorized" }));
    }

    let body = "";
    req.on("data", (d) => (body += d));
    req.on("end", () => {
      const { commandKey } = JSON.parse(body);
      const command = ALLOWED[commandKey];

      if (!command) {
        res.writeHead(400).end(JSON.stringify({ error: "Unknown command" }));
        return;
      }

      console.log(`[AGENT] Running: ${command}`);
      res.writeHead(200, { "Content-Type": "application/json" });
      res.end(
        JSON.stringify({ success: true, output: `Executing: ${command}` }),
      );

      // Send response first, THEN execute — so client gets the ack before Docker dies
      setTimeout(() => exec(command), 500);
    });
  })
  .listen(PORT, "0.0.0.0", () => {
    console.log(`Host agent listening on 127.0.0.1:${PORT}`);
  });
```

#### 1.2 Register as a systemd service

```bash
sudo nano /etc/systemd/system/host-agent.service
```

```ini
[Unit]
Description=Home Server Host Agent
After=network.target

[Service]
ExecStart=/usr/bin/node /opt/host-agent/agent.js
Restart=always
Environment=AGENT_SECRET=your_secret_here
User=root
WorkingDirectory=/opt/host-agent

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable host-agent
sudo systemctl start host-agent
```

---

### Part 2 — Status Server additions

The existing status server gains a `/api/command` route that dispatches commands to either the container runtime or the host agent depending on the command type.

#### 2.1 Add to `.env`

```env
AGENT_SECRET=your_secret_here    # Must match host-agent.service Environment value
```

#### 2.2 Add to top of `index.js`

```js
const { exec } = require("child_process");
const util = require("util");
const execAsync = util.promisify(exec);

const AGENT_URL = "http://172.18.0.1:7070/run"; // docker_gwbridge IP
const AGENT_SECRET = process.env.AGENT_SECRET;

// Executed inside the container via child_process
const ALLOWED_COMMANDS = {
  "docker-ps": "docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'",
  "docker-ps-all":
    "docker ps -a --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'",
  "docker-stats":
    "docker stats --no-stream --format 'table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}'",
  "docker-images":
    "docker images --format 'table {{.Repository}}\t{{.Tag}}\t{{.Size}}'",
  "disk-usage": "df -h",
  "memory-usage": "free -h",
};

// Commands that must run on the HOST via the agent
const HOST_COMMANDS = new Set(["docker-daemon-restart", "reboot"]);
```

#### 2.3 Add the `/api/command` route

```js
app.post("/api/command", requireSecret, express.json(), async (req, res) => {
  const { commandKey } = req.body;

  if (!commandKey) {
    return res.status(400).json({ error: "Missing commandKey" });
  }

  // ── Route to host agent ──────────────────────────────────────────────────
  if (HOST_COMMANDS.has(commandKey)) {
    try {
      const agentRes = await axios.post(
        AGENT_URL,
        { commandKey },
        {
          headers: {
            "x-agent-secret": AGENT_SECRET,
            "Content-Type": "application/json",
          },
          timeout: 5000,
        },
      );
      return res.json(agentRes.data);
    } catch (err) {
      return res.status(500).json({
        success: false,
        output: `Host agent unreachable: ${err.message}`,
      });
    }
  }

  // ── Run inside container ─────────────────────────────────────────────────
  if (!ALLOWED_COMMANDS[commandKey]) {
    return res.status(400).json({
      error: "Invalid command",
      allowed: [...Object.keys(ALLOWED_COMMANDS), ...HOST_COMMANDS],
    });
  }

  const command = ALLOWED_COMMANDS[commandKey];
  console.log(`[COMMAND] Executing: ${command}`);

  try {
    const { stdout, stderr } = await execAsync(command, { timeout: 15000 });
    res.json({
      success: true,
      command,
      output: stdout || stderr || "(no output)",
    });
  } catch (err) {
    res.json({
      success: false,
      command,
      output: err.stderr || err.message,
    });
  }
});
```

---

### Part 3 — Phone App

The app gains a **Commands** tab alongside the existing Status tab.

#### `theme.ts` — shared design constants

```ts
export const C = {
  bg: "#0f172a",
  surface: "#1e293b",
  surfaceElevated: "#243044",
  border: "#334155",
  divider: "#1e2d3d",
  text: "#f1f5f9",
  muted: "#94a3b8",
  cyan: "#22d3ee",
  green: "#4ade80",
  amber: "#fbbf24",
  rose: "#fb7185",
};
export const SPACE = { xs: 4, sm: 8, md: 12, lg: 16, xl: 20 };
export const RADIUS = { card: 14, pill: 20 };
```

#### `api.ts` — add `sendCommand`

```ts
export async function sendCommand(commandKey: string) {
  const BASE_URL = STATUS_URL.replace("/api/status", "");
  const { signal, clear } = makeTimeout(20_000);
  try {
    const res = await fetch(`${BASE_URL}/api/command?secret=${STATUS_SECRET}`, {
      method: "POST",
      signal,
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ commandKey }),
    });
    clear();
    return await res.json();
  } catch (err: any) {
    clear();
    return { success: false, output: err.message ?? "Request failed" };
  }
}
```

#### `App.tsx` — tab bar

Add a bottom tab navigator that switches between the existing `StatusScreen` and the new `CommandsScreen`.

---

## Stack Summary

| Service           | Image / Type                       | Internal Port | Public URL                  |
| ----------------- | ---------------------------------- | ------------- | --------------------------- |
| Dokploy           | `dokploy/dokploy`                  | 3000          | `panel.johndoe.dev`         |
| Portfolio         | Custom (GitHub)                    | 3000          | `johndoe.dev`               |
| Immich            | `ghcr.io/immich-app/immich-server` | 2283          | `photos.johndoe.dev`        |
| Jellyfin          | `jellyfin/jellyfin`                | 8096          | `stream.johndoe.dev`        |
| app Frontend      | Custom (GitHub)                    | 3000          | `app.johndoe.dev`           |
| app Backend       | Custom (GitHub)                    | 8000          | _(internal only)_           |
| MongoDB           | `mongo`                            | 27017         | _(internal only)_           |
| Status Server     | Custom (GitHub, Node.js)           | 3001          | `status.johndoe.dev`        |
| Host Agent        | systemd service (Node.js)          | 7070          | _(host only, `172.18.0.1`)_ |
| Cloudflare Tunnel | `cloudflared`                      | —             | _(agent only)_              |
