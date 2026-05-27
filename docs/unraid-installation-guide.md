# HermesOS — Ultimate Unraid Installation Guide

> **HermesOS** is a Kali Linux Docker container purpose-built for [Hermes Agent](https://github.com/NousResearch/hermes-agent). It ships a full KDE Plasma desktop over the browser (via Selkies), alongside four pre-installed AI toolsuites: **hermes-agent**, **hermelinChat**, **CloakBrowser**, and **ai-hedge-fund**.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Step 1 — Build or Pull the Image](#2-step-1--build-or-pull-the-image)
3. [Step 2 — Add the Container in Unraid](#3-step-2--add-the-container-in-unraid)
4. [Step 3 — Port Mappings](#4-step-3--port-mappings)
5. [Step 4 — Volume / Path Mappings](#5-step-4--volume--path-mappings)
6. [Step 5 — Environment Variables](#6-step-5--environment-variables)
7. [Step 6 — GPU Passthrough (Optional but Recommended)](#7-step-6--gpu-passthrough-optional-but-recommended)
8. [Step 7 — Start and Access the Container](#8-step-7--start-and-access-the-container)
9. [Step 8 — First-Run HermesOS Configuration](#9-step-8--first-run-hermesos-configuration)
10. [Step 9 — Using the AI Tools](#10-step-9--using-the-ai-tools)
11. [Step 10 — Reverse Proxy & HTTPS (SWAG/Nginx Proxy Manager)](#11-step-10--reverse-proxy--https-swagnginx-proxy-manager)
12. [Unraid Template (XML)](#12-unraid-template-xml)
13. [Troubleshooting](#13-troubleshooting)

---

## 1. Prerequisites

Before you begin, make sure the following are in place on your Unraid server:

| Requirement | Notes |
|---|---|
| **Unraid 6.12+** | Earlier versions may lack required Docker features |
| **Community Applications (CA) plugin** | For easy plugin/template management |
| **Docker installed and running** | Default on all Unraid installs |
| **At least 30 GB free on your Docker storage share** | The image is large (~15 GB uncompressed) |
| **At least 8 GB RAM allocated to Docker** | 16 GB strongly recommended for running AI models |
| **Internet access from the Unraid server** | Required to pull the image and for AI API calls |
| **Nvidia GPU (optional)** | For hardware-accelerated streaming; see [Step 6](#7-step-6--gpu-passthrough-optional-but-recommended) |

### Required Unraid Plugins (install via Community Applications)

- **Nvidia Driver Plugin** — if you have an Nvidia GPU
- **Unraid-Nvidia** — exposes the GPU to Docker containers

---

## 2. Step 1 — Build or Pull the Image

### Option A: Build Locally (recommended for latest HermesOS features)

On your Unraid server terminal (`Tools → Terminal`):

```bash
# Clone the HermesOS repository
cd /mnt/user/appdata
git clone https://github.com/DocwatZ/HermesOS.git
cd HermesOS

# Build the image (this will take 20–40 minutes on first build)
docker build -t hermesos:latest .
```

> **Tip:** Building locally pins you to the exact HermesOS commit you cloned. This is the safest option because all four AI tools are baked into the image at known versions.

### Option B: Use the LinuxServer Kali Base Directly

If you only need the base Kali desktop and will layer HermesOS tools at runtime, you can pull from LinuxServer directly:

```bash
docker pull lscr.io/linuxserver/kali-linux:latest
```

Then tag it as `hermesos:latest` for consistency with the rest of this guide:

```bash
docker tag lscr.io/linuxserver/kali-linux:latest hermesos:latest
```

---

## 3. Step 2 — Add the Container in Unraid

1. In the Unraid Web UI, go to **Docker → Add Container**.
2. Click **Advanced View** (toggle in the top-right corner) to see all fields.
3. Fill in the **Basic Info** section:

| Field | Value |
|---|---|
| **Name** | `HermesOS` |
| **Repository** | `hermesos:latest` (local build) **or** `lscr.io/linuxserver/kali-linux:latest` |
| **Docker Hub URL** | *(leave blank for local builds)* |
| **Network Type** | `bridge` |
| **Console shell command** | `bash` |
| **Privileged** | **Yes** *(required for Docker-in-Docker, full device access, and Kali tools)* |
| **Extra Parameters** | `--shm-size=1gb --security-opt seccomp=unconfined` |

> **Why `--privileged`?**  
> Hermes Agent runs subprocesses that need raw socket access, and some Kali tools require kernel-level capabilities. CloakBrowser's Chromium binary also benefits from full privilege for sandbox bypass.

---

## 4. Step 3 — Port Mappings

Click **Add another Path, Port, Variable, Label or Device** → **Port** for each row below:

| Container Port | Host Port | Protocol | Description |
|---|---|---|---|
| `3001` | `3001` | TCP | Kali Desktop (HTTPS via Selkies) |
| `3002` | `3002` | TCP | HermelinChat Web UI (HTTP) |

> **Port conflict check:** Run `netstat -tlnp | grep -E '3001|3002'` in the Unraid terminal to verify these ports are free before starting the container.

---

## 5. Step 4 — Volume / Path Mappings

Add the following path mappings (click **Add another Path** for each):

| Container Path | Host Path | Access Mode | Description |
|---|---|---|---|
| `/config` | `/mnt/user/appdata/HermesOS` | Read/Write | All persistent user data — Hermes config, chat history, hedge fund API keys, KDE settings |

> **Important:** All HermesOS configuration lives under `/config` inside the container, which maps to `/mnt/user/appdata/HermesOS` on the host. This means:
>
> - `/mnt/user/appdata/HermesOS/.hermes/` — Hermes Agent state, memory, and `.env` (API keys)
> - `/mnt/user/appdata/HermesOS/.hermelin.env` — HermelinChat credentials
> - `/mnt/user/appdata/HermesOS/ai-hedge-fund/.env` — Hedge fund API keys
>
> These directories are created automatically on first start.

### Optional Additional Mounts

| Container Path | Host Path | Description |
|---|---|---|
| `/var/lib/docker` | `/mnt/user/appdata/HermesOS/docker-data` | Docker-in-Docker data (add `--privileged` if not already set) |
| `/var/run/docker.sock` | `/var/run/docker.sock` | Mount host Docker socket to manage host containers from within HermesOS |

---

## 6. Step 5 — Environment Variables

Add the following environment variables (click **Add another Path** → **Variable** for each):

### Required

| Variable | Value | Description |
|---|---|---|
| `PUID` | `99` | Unraid's `nobody` user ID |
| `PGID` | `100` | Unraid's `users` group ID |
| `TZ` | `America/New_York` | Your timezone (e.g., `Europe/London`, `Asia/Tokyo`) |
| `TITLE` | `HermesOS` | Browser tab title |

### Security (Strongly Recommended)

| Variable | Value | Description |
|---|---|---|
| `CUSTOM_USER` | `hermes` | HTTP Basic auth username for the desktop |
| `PASSWORD` | `your-strong-password` | HTTP Basic auth password for the desktop |
| `HERMELIN_ALLOW_INSECURE_HTTP` | `1` | Allows HermelinChat to serve over plain HTTP when not using TLS certs. Required unless `HERMELIN_SSL_CERTFILE`/`HERMELIN_SSL_KEYFILE` are configured. |

> **Warning:** Without `CUSTOM_USER` and `PASSWORD`, **anyone who can reach port 3001 has unauthenticated root access inside the container.** Always set these before exposing the container beyond localhost.
>
> If you want proper TLS for HermelinChat instead of plain HTTP, configure `HERMELIN_SSL_CERTFILE` and `HERMELIN_SSL_KEYFILE`.

### Display & Performance

| Variable | Value | Description |
|---|---|---|
| `NO_GAMEPAD` | `true` | Disables gamepad interposer (not needed for AI workloads) |
| `PIXELFLUX_WAYLAND` | `true` | Enables KDE Plasma Wayland + zero-copy GPU encoding (recommended) |
| `AUTO_GPU` | `true` | Automatically uses the first detected GPU for rendering and encoding |

### Optional Tuning

| Variable | Value | Description |
|---|---|---|
| `MAX_RES` | `3840x2160` | Caps virtual framebuffer at 4K (saves VRAM on lower-end GPUs) |
| `LC_ALL` | `en_US.UTF-8` | Desktop locale |
| `SUBFOLDER` | `/hermesos/` | Set this if running behind a reverse proxy subfolder |

---

## 7. Step 6 — GPU Passthrough (Optional but Recommended)

GPU acceleration dramatically improves the Selkies streaming quality and reduces CPU load. It also enables faster local LLM inference for Hermes Agent.

### Intel / AMD GPU

In the **Devices** section of the container add:

```
/dev/dri:/dev/dri
```

Then add environment variables:

| Variable | Value |
|---|---|
| `DRINODE` | `/dev/dri/renderD128` |
| `DRI_NODE` | `/dev/dri/renderD128` |

### Nvidia GPU

**Prerequisites (complete in order):**

1. Install the **Nvidia Driver Plugin** from Community Applications (use the **Production** branch).

2. Edit `/boot/syslinux/syslinux.cfg` via the Unraid Flash share and add to the `append` line:
   ```
   nvidia-drm.modeset=1 nvidia_drm.fbdev=1
   ```
   Then reboot.

3. Insert a **dummy HDMI/DisplayPort plug** into the GPU if the server is headless. This is required for Nvidia DRM to initialize.

4. Verify the driver is loaded after reboot:
   ```bash
   nvidia-smi
   ```

**Container configuration:**

In **Extra Parameters**, add:
```
--gpus all --runtime nvidia
```

Add environment variables:

| Variable | Value |
|---|---|
| `DRINODE` | `/dev/dri/renderD128` |
| `DRI_NODE` | `/dev/dri/renderD128` |
| `PIXELFLUX_WAYLAND` | `true` |

---

## 8. Step 7 — Start and Access the Container

1. Click **Apply** to save the container configuration.
2. Click **Start** next to the `HermesOS` container in the Docker tab.
3. Wait 60–90 seconds for the first-boot initialization (Hermes config files are generated, services start).
4. Check container logs by clicking the container icon → **Logs** to confirm startup.

**Access URLs:**

| Service | URL | Notes |
|---|---|---|
| KDE Plasma Desktop | `https://YOUR-UNRAID-IP:3001/` | Full desktop — accept the self-signed cert warning |
| HermelinChat Web UI | `http://YOUR-UNRAID-IP:3002/` | Hermes Agent browser interface |

---

## 9. Step 8 — First-Run HermesOS Configuration

On first start, the `50-hermes` init script creates config stubs under `/mnt/user/appdata/HermesOS/` on the host. You must populate API keys before the AI tools will function.

### Configure Hermes Agent

Edit the file at:
```
/mnt/user/appdata/HermesOS/.hermes/.env
```

Add at least one LLM provider key:

```bash
# Choose one or more:
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
GROQ_API_KEY=gsk_...
```

### Configure HermelinChat Password

Edit the file at:
```
/mnt/user/appdata/HermesOS/.hermelin.env
```

Change the default password:

```bash
HERMELIN_ALLOWED_IPS=*
HERMELIN_PASSWORD=your-strong-password   # ← CHANGE THIS
HERMELIN_COOKIE_SECRET=<auto-generated — do not change>
```

### Configure AI Hedge Fund

Edit the file at:
```
/mnt/user/appdata/HermesOS/ai-hedge-fund/.env
```

```bash
OPENAI_API_KEY=sk-...                          # or another supported provider
FINANCIAL_DATASETS_API_KEY=your-fda-key        # from financialdatasets.ai
```

After editing any `.env` file, **restart the container** from the Unraid Docker tab for the changes to take effect.

---

## 10. Step 9 — Using the AI Tools

Once the container is running and API keys are configured, access each tool from inside the KDE desktop (port 3001) or via HermelinChat (port 3002).

### Hermes Agent (CLI)

Open **Konsole** from the KDE taskbar and run:

```bash
hermes
```

This launches the Hermes Agent REPL. Hermes will self-improve over sessions, storing memories in `/config/.hermes/`.

### HermelinChat (Browser UI)

Navigate to `http://YOUR-UNRAID-IP:3002/` in any browser. Log in with the password from `/config/.hermelin.env`. HermelinChat provides:

- A live xterm.js terminal wired to the `hermes` process
- Session history sidebar (reads from `/config/.hermes/state.db`)
- Artifact panel for previewing code and HTML outputs

### CloakBrowser

CloakBrowser is installed as a Python library at `/opt/cloakbrowser-env`. Open a Konsole terminal inside the desktop and use it in any Python script:

```python
from cloakbrowser import launch

browser = launch()
page = browser.new_page()
page.goto("https://example.com")
print(page.title())
browser.close()
```

On first use the stealth Chromium binary (~200 MB) will auto-download and cache under `/config/.cache/cloakbrowser/`.

### AI Hedge Fund

Open a Konsole terminal inside the desktop and navigate to the install directory:

```bash
cd /opt/ai-hedge-fund

# Copy your API keys into place (only needed once)
cp /config/ai-hedge-fund/.env .env

# Run a simulated analysis
poetry run python src/main.py --ticker AAPL,MSFT,NVDA

# Run with a date range
poetry run python src/main.py --ticker AAPL --start-date 2024-01-01 --end-date 2024-12-31

# Run the backtester
poetry run python src/backtester.py --ticker AAPL,MSFT,NVDA
```

> **Reminder:** The AI Hedge Fund is a simulation tool only. No real trades are executed.

---

## 11. Step 10 — Reverse Proxy & HTTPS (SWAG/Nginx Proxy Manager)

To expose HermesOS securely over the internet (e.g., at `https://hermesos.yourdomain.com`), place it behind a reverse proxy.

### Option A: Nginx Proxy Manager

1. In NPM, create two **Proxy Hosts**:

   | Domain | Forward Host | Forward Port | SSL |
   |---|---|---|---|
   | `hermesos.yourdomain.com` | `YOUR-UNRAID-IP` | `3001` | Let's Encrypt, Force SSL |
   | `chat.yourdomain.com` | `YOUR-UNRAID-IP` | `3002` | Let's Encrypt, Force SSL |

2. Under **Advanced** for the desktop proxy, add:

   ```nginx
   proxy_set_header Upgrade $http_upgrade;
   proxy_set_header Connection "upgrade";
   proxy_read_timeout 3600s;
   proxy_send_timeout 3600s;
   ```

3. Enable **Disable Strict Certificate Validation** in NPM because the container uses a self-signed cert on port 3001.

### Option B: SWAG (LinuxServer)

Place a config file at `/mnt/user/appdata/swag/nginx/proxy-confs/hermesos.subdomain.conf`:

```nginx
server {
    listen 443 ssl;
    server_name hermesos.*;

    include /config/nginx/ssl.conf;
    client_max_body_size 0;

    location / {
        include /config/nginx/proxy.conf;
        include /config/nginx/resolver.conf;
        set $upstream_app YOUR-UNRAID-IP;
        set $upstream_port 3001;
        set $upstream_proto https;
        proxy_pass $upstream_proto://$upstream_app:$upstream_port;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

> **Security Note:** Set `CUSTOM_USER` and `PASSWORD` environment variables on the container and configure additional authentication (e.g., Authelia, Authentik) before exposing HermesOS to the public internet. The web terminal grants passwordless root inside the container.

---

## 12. Unraid Template (XML)

Save this as `/boot/config/plugins/dockerMan/templates-user/HermesOS.xml` on your Unraid flash drive to add HermesOS as a reusable Community Applications-style template:

```xml
<?xml version="1.0"?>
<Container version="2">
  <Name>HermesOS</Name>
  <Repository>hermesos:latest</Repository>
  <Registry></Registry>
  <Network>bridge</Network>
  <MyIP></MyIP>
  <Shell>bash</Shell>
  <Privileged>true</Privileged>
  <Support>https://github.com/DocwatZ/HermesOS</Support>
  <Project>https://github.com/DocwatZ/HermesOS</Project>
  <Overview>HermesOS — A Kali Linux AI environment built for Hermes Agent. Ships hermes-agent, hermelinChat, CloakBrowser, and ai-hedge-fund on a KDE Plasma desktop streamed over the browser.</Overview>
  <Category>Tools: Network:Other Security: Status:Stable</Category>
  <WebUI>https://[IP]:[PORT:3001]/</WebUI>
  <TemplateURL></TemplateURL>
  <Icon>https://raw.githubusercontent.com/linuxserver/docker-templates/master/linuxserver.io/img/kali-logo.png</Icon>
  <ExtraParams>--shm-size=1gb --security-opt seccomp=unconfined</ExtraParams>
  <PostArgs></PostArgs>
  <CPUset></CPUset>
  <DateInstalled></DateInstalled>
  <DonateText></DonateText>
  <DonateLink></DonateLink>
  <Requires></Requires>

  <!-- Ports -->
  <Config Name="Web Desktop (HTTPS)" Target="3001" Default="3001" Mode="tcp" Description="Kali KDE Plasma desktop — access via browser" Type="Port" Display="always" Required="true" Mask="false">3001</Config>
  <Config Name="HermelinChat UI (HTTP)" Target="3002" Default="3002" Mode="tcp" Description="HermelinChat web interface for Hermes Agent" Type="Port" Display="always" Required="true" Mask="false">3002</Config>

  <!-- Volumes -->
  <Config Name="Config &amp; Data" Target="/config" Default="/mnt/user/appdata/HermesOS" Mode="rw" Description="Persistent config, Hermes state/memory, KDE settings, and AI tool API keys" Type="Path" Display="always" Required="true" Mask="false">/mnt/user/appdata/HermesOS</Config>

  <!-- Environment Variables -->
  <Config Name="PUID" Target="PUID" Default="99" Mode="" Description="User ID (99 = nobody on Unraid)" Type="Variable" Display="always" Required="true" Mask="false">99</Config>
  <Config Name="PGID" Target="PGID" Default="100" Mode="" Description="Group ID (100 = users on Unraid)" Type="Variable" Display="always" Required="true" Mask="false">100</Config>
  <Config Name="TZ" Target="TZ" Default="America/New_York" Mode="" Description="Timezone" Type="Variable" Display="always" Required="true" Mask="false">America/New_York</Config>
  <Config Name="TITLE" Target="TITLE" Default="HermesOS" Mode="" Description="Browser tab title" Type="Variable" Display="always" Required="false" Mask="false">HermesOS</Config>
  <Config Name="CUSTOM_USER" Target="CUSTOM_USER" Default="hermes" Mode="" Description="HTTP Basic auth username for the desktop" Type="Variable" Display="always" Required="false" Mask="false">hermes</Config>
  <Config Name="PASSWORD" Target="PASSWORD" Default="" Mode="" Description="HTTP Basic auth password for the desktop — leave blank to disable auth (NOT recommended)" Type="Variable" Display="always" Required="false" Mask="true"></Config>
  <Config Name="NO_GAMEPAD" Target="NO_GAMEPAD" Default="true" Mode="" Description="Disable gamepad interposer" Type="Variable" Display="advanced" Required="false" Mask="false">true</Config>
  <Config Name="PIXELFLUX_WAYLAND" Target="PIXELFLUX_WAYLAND" Default="true" Mode="" Description="Enable KDE Wayland + zero-copy GPU encoding" Type="Variable" Display="advanced" Required="false" Mask="false">true</Config>
  <Config Name="AUTO_GPU" Target="AUTO_GPU" Default="true" Mode="" Description="Automatically use first detected GPU" Type="Variable" Display="advanced" Required="false" Mask="false">true</Config>
</Container>
```

To apply after saving the file, go to **Docker → Add Container → Select a template** and choose **HermesOS** from the user templates list.

---

## 13. Troubleshooting

### Container fails to start / exits immediately

Check the logs:
```bash
docker logs HermesOS
```

Common causes:
- **Port conflict** — another service is using 3001 or 3002. Change the host port mapping.
- **Insufficient memory** — ensure Docker has at least 8 GB RAM available in Unraid's settings.
- **Missing `shm-size`** — make sure `--shm-size=1gb` is in Extra Parameters.

### Desktop is blank / black screen

1. Wait up to 90 seconds after first start — KDE Plasma takes time to initialize.
2. Try disabling Wayland: set `PIXELFLUX_WAYLAND=false` and restart.
3. Check for AVX2 support (required for Wayland on x86_64):
   ```bash
   grep avx2 /proc/cpuinfo | head -1
   ```
   If no output, your CPU lacks AVX2 — Wayland will auto-fall back to X11.

### HermelinChat shows "hermes not found"

Hermes Agent must be initialized first. Open a terminal in the KDE desktop (port 3001) and run:
```bash
hermes
```
This creates `/config/.hermes/state.db`. Then refresh the HermelinChat page.

### HermelinChat crash loop: "refusing to serve insecure HTTP"

```bash
Add HERMELIN_ALLOW_INSECURE_HTTP=1 to /mnt/user/appdata/HermesOS/.hermelin.env and restart the container.
Alternatively, configure TLS by setting HERMELIN_SSL_CERTFILE and HERMELIN_SSL_KEYFILE to point to your certificate and key files.
```

### hermes-agent returns API errors

Edit `/mnt/user/appdata/HermesOS/.hermes/.env` and verify your API key is correct and has sufficient credits. Restart the container after editing.

### CloakBrowser fails to launch

On first run, the stealth Chromium binary must download (~200 MB). Ensure the container has internet access. If behind a firewall, whitelist `github.com` and `releases.cloakhq.com`.

### AI Hedge Fund: `FINANCIAL_DATASETS_API_KEY` errors

Register for a free API key at [financialdatasets.ai](https://financialdatasets.ai) and add it to `/mnt/user/appdata/HermesOS/ai-hedge-fund/.env`. Restart the container.

### GPU not detected / software rendering fallback

1. Verify the Nvidia Driver Plugin is installed and `nvidia-smi` works in the Unraid terminal.
2. Confirm `nvidia-drm.modeset=1 nvidia_drm.fbdev=1` is in `/boot/syslinux/syslinux.cfg` and the server has been rebooted.
3. Ensure `--gpus all --runtime nvidia` is in Extra Parameters.
4. Confirm the dummy HDMI plug is inserted into the GPU.

### Self-signed certificate warning in browser

This is expected when accessing port 3001 directly. Accept the certificate exception in your browser, or set up a proper TLS certificate via a reverse proxy ([Step 10](#11-step-10--reverse-proxy--https-swagnginx-proxy-manager)).

---

*Guide version: 1.0 — HermesOS based on `ghcr.io/linuxserver/baseimage-selkies:kali`*
