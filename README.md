# HermesOS

[![GitHub Stars](https://img.shields.io/github/stars/DocwatZ/HermesOS.svg?color=2e86c1&labelColor=1a1a2e&logoColor=ffffff&style=for-the-badge&logo=github)](https://github.com/DocwatZ/HermesOS)
[![GitHub Release](https://img.shields.io/github/release/DocwatZ/HermesOS.svg?color=2e86c1&labelColor=1a1a2e&logoColor=ffffff&style=for-the-badge&logo=github)](https://github.com/DocwatZ/HermesOS/releases)
[![License](https://img.shields.io/github/license/DocwatZ/HermesOS?color=2e86c1&labelColor=1a1a2e&logoColor=ffffff&style=for-the-badge)](LICENSE)

> **HermesOS** is a Kali Linux Docker container purpose-built for AI agents. It layers four open-source AI toolsuites on top of a full KDE Plasma desktop streamed over the browser — giving [Hermes Agent](https://github.com/NousResearch/hermes-agent) a rich, persistent, browser-accessible operating environment with penetration-testing capabilities, stealth browsing, a conversational UI, and AI-driven financial analysis — all in one container.

[![Hermes-Agent]([https://raw.githubusercontent.com/linuxserver/docker-templates/master/linuxserver.io/img/kali-logo.png](https://github.com/NousResearch/hermes-agent/raw/main/assets/banner.png))](https://github.com/DocwatZ/HermesOS)

---

## What's Inside

| Tool | Description | Access |
|---|---|---|
| 🧠 **[hermes-agent](https://github.com/NousResearch/hermes-agent)** | Self-improving AI agent with persistent memory, skill creation, and multi-platform messaging (Telegram, Discord, Slack, CLI) | `hermes` in any terminal |
| 🖥️ **[hermelinChat](https://github.com/quarker1337/hermelinChat)** | Browser-based UI for Hermes — xterm.js PTY terminal, session history sidebar, and artifact preview panel | `http://host:3000` |
| 🕵️ **[CloakBrowser](https://github.com/CloakHQ/CloakBrowser)** | Stealth Chromium with 58 source-level fingerprint patches — drop-in Playwright/Puppeteer replacement that passes Cloudflare, reCAPTCHA v3, and FingerprintJS | Python lib at `/opt/cloakbrowser-env` |
| 📈 **[ai-hedge-fund](https://github.com/virattt/ai-hedge-fund)** | 19 AI agents modelled on legendary investors (Buffett, Munger, Burry, Wood, etc.) that analyse stocks and simulate trading decisions | `poetry run python src/main.py` |

**Base:** [`ghcr.io/linuxserver/baseimage-selkies:kali`](https://github.com/linuxserver/docker-baseimage-selkies) — Kali Linux with KDE Plasma streamed via Selkies (WebRTC). All Kali tools (`kali-linux-default`, `kali-tools-top10`, Hydra, Metasploit, Burp Suite, etc.) remain fully intact.

---

## Supported Architectures

| Architecture | Available | Build file |
| :----: | :----: | ---- |
| x86-64 | ✅ | `Dockerfile` |
| arm64 | ✅ | `Dockerfile.aarch64` |

---

## Quick Start

### docker compose (recommended)

```yaml
---
services:
  hermesos:
    build: .                         # build locally from source
    # image: hermesos:latest         # or use a pre-built local tag
    container_name: HermesOS
    privileged: true                 # required for Kali tools + DinD + CloakBrowser
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Etc/UTC
      - TITLE=HermesOS
      - CUSTOM_USER=hermes           # desktop HTTP Basic auth username
      - ******            # desktop HTTP Basic auth password
      - NO_GAMEPAD=true
      - PIXELFLUX_WAYLAND=true
    volumes:
      - /path/to/data:/config
    ports:
      - 3000:3000   # HermelinChat web UI
      - 3001:3001   # KDE desktop (HTTPS)
    shm_size: "1gb"
    security_opt:
      - seccomp:unconfined
    restart: unless-stopped
```

### docker cli

```bash
docker run -d \
  --name=HermesOS \
  --privileged \
  -e PUID=1000 \
  -e PGID=1000 \
  -e TZ=Etc/UTC \
  -e TITLE=HermesOS \
  -e CUSTOM_USER=hermes \
  -e ****** \
  -e NO_GAMEPAD=true \
  -e PIXELFLUX_WAYLAND=true \
  -p 3000:3000 \
  -p 3001:3001 \
  -v /path/to/data:/config \
  --shm-size="1gb" \
  --security-opt seccomp=unconfined \
  --restart unless-stopped \
  hermesos:latest
```

### Building the image

```bash
git clone https://github.com/DocwatZ/HermesOS.git
cd HermesOS
docker build -t hermesos:latest .
# ARM64:
docker build -t hermesos:latest -f Dockerfile.aarch64 .
```

> **Build time:** 20–40 minutes on first build. The image is large (~15 GB uncompressed) because it bundles a full Kali toolset plus Python/Node AI stacks.

---

## Accessing the Container

| Service | URL | Notes |
|---|---|---|
| KDE Plasma Desktop | `https://YOUR-HOST:3001/` | Full Kali desktop in the browser. Accept the self-signed cert. |
| HermelinChat | `http://YOUR-HOST:3000/` | Hermes Agent web UI. Set password in `/config/.hermelin.env`. |

> **HTTPS is required** for full desktop functionality (WebCodecs, audio, clipboard). Port 3001 is the HTTPS endpoint. Port 3000 (HermelinChat) is HTTP and should be placed behind a reverse proxy for remote access.

---

## First-Run Configuration

On the first container start, `50-hermes` automatically creates config stubs under `/config` (your mounted volume). Edit these files to add API keys, then **restart the container**.

### Hermes Agent — `/config/.hermes/.env`

```bash
# Add one or more LLM provider keys:
OPENAI_API_KEY=sk-...
# ANTHROPIC_API_KEY=sk-ant-...
# GROQ_API_KEY=gsk_...
```

Then open a terminal in the KDE desktop and run:

```bash
hermes
```

### HermelinChat — `/config/.hermelin.env`

```bash
HERMELIN_ALLOWED_IPS=*
HERMELIN_PASSWORD=your-strong-password   # Change this before exposing port 3000
HERMELIN_COOKIE_SECRET=<auto-generated>  # Do not change
```

HermelinChat starts automatically as a background service and is available at `http://YOUR-HOST:3000/`.

### AI Hedge Fund — `/config/ai-hedge-fund/.env`

```bash
OPENAI_API_KEY=sk-...                         # or GROQ_API_KEY / ANTHROPIC_API_KEY
FINANCIAL_DATASETS_API_KEY=your-fda-key       # from financialdatasets.ai
```

Run from a Konsole terminal inside the desktop:

```bash
cd /opt/ai-hedge-fund
cp /config/ai-hedge-fund/.env .env            # copy keys into place (once)
poetry run python src/main.py --ticker AAPL,MSFT,NVDA
```

### CloakBrowser

CloakBrowser is ready to use with no additional configuration. Use it in any Python script inside the container:

```python
from cloakbrowser import launch

browser = launch()
page = browser.new_page()
page.goto("https://example.com")
print(page.title())
browser.close()
```

The stealth Chromium binary (~200 MB) auto-downloads on first use to `/config/.cache/cloakbrowser/`.

---

## Installed AI Tool Locations

| Tool | Install Path | Entry Point |
|---|---|---|
| hermes-agent | `/opt/hermes-agent` | `/usr/local/bin/hermes` |
| hermelinChat | `/opt/hermelinChat` | `/usr/local/bin/hermelin` (auto-service) |
| CloakBrowser | `/opt/cloakbrowser-env` | `from cloakbrowser import launch` |
| ai-hedge-fund | `/opt/ai-hedge-fund` | `poetry run python src/main.py` |

---

## Security

> [!WARNING]
> This container provides privileged access to the host system. Do not expose it to the Internet without proper authentication and a reverse proxy.

- By default, the desktop (port 3001) has **no auth**. Always set `CUSTOM_USER` and `PASSWORD`.
- HermelinChat (port 3000) is password-protected via `/config/.hermelin.env`.
- The web terminal has passwordless `sudo` access inside the container.
- For internet exposure, place both ports behind [SWAG](https://github.com/linuxserver/docker-swag) or Nginx Proxy Manager with TLS.

See the [Unraid Installation Guide](docs/unraid-installation-guide.md) for a detailed reverse proxy setup.

---

## Hardware Acceleration & Wayland

HermesOS uses KDE Plasma on a Wayland stack (default). Wayland enables zero-copy GPU encoding, dramatically reducing CPU load and stream latency.

**Wayland requires AVX2** on x86_64 (Intel Haswell / 2013+ or any AMD Ryzen). Older CPUs automatically fall back to X11.

Disable Wayland manually if needed:

```bash
-e PIXELFLUX_WAYLAND=false
```

### Intel / AMD GPU

```yaml
devices:
  - /dev/dri:/dev/dri
environment:
  - PIXELFLUX_WAYLAND=true
  - DRINODE=/dev/dri/renderD128
  - DRI_NODE=/dev/dri/renderD128
```

### Nvidia GPU

**Prerequisites:**
1. Install Nvidia proprietary drivers **580+** (use the `.run` installer from nvidia.com — not apt).
2. Add `nvidia-drm.modeset=1 nvidia_drm.fbdev=1` to your kernel boot parameters.
3. Insert a **dummy HDMI/DP plug** on headless systems.
4. Configure the Docker runtime:
   ```bash
   sudo nvidia-ctk runtime configure --runtime=docker
   sudo systemctl restart docker
   ```

**Compose:**

```yaml
services:
  hermesos:
    image: hermesos:latest
    environment:
      - PIXELFLUX_WAYLAND=true
      - DRINODE=/dev/dri/renderD128
      - DRI_NODE=/dev/dri/renderD128
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [compute,video,graphics,utility]
```

**Extra parameters for Unraid:** `--gpus all --runtime nvidia`

---

## Parameters

| Parameter | Function |
| :----: | --- |
| `-p 3000:3000` | HermelinChat web UI |
| `-p 3001:3001` | KDE Plasma desktop (HTTPS) |
| `-e PUID=1000` | User ID |
| `-e PGID=1000` | Group ID |
| `-e TZ=Etc/UTC` | Timezone ([list](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones#List)) |
| `-e TITLE=HermesOS` | Browser tab title |
| `-e CUSTOM_USER=hermes` | Desktop HTTP Basic auth username |
| `-e ****** | Desktop HTTP Basic auth password (no auth if unset) |
| `-e NO_GAMEPAD=true` | Disable gamepad interposer |
| `-e PIXELFLUX_WAYLAND=true` | Enable KDE Wayland + zero-copy GPU encoding |
| `-e AUTO_GPU=true` | Auto-detect and use first GPU for rendering and encoding |
| `-e DRINODE=/dev/dri/renderD128` | GPU for rendering (EGL) |
| `-e DRI_NODE=/dev/dri/renderD128` | GPU for encoding (VAAPI/NVENC) |
| `-v /config` | Persistent data: Hermes state, AI keys, KDE settings, HermelinChat config |
| `--shm-size=1gb` | Required for desktop stability |
| `--privileged` | Required for Kali tools, Docker-in-Docker, and full device access |
| `--security-opt seccomp=unconfined` | Required for some Kali tools and CloakBrowser on older kernels |

<details>
<summary>Click to expand: All Selkies-based GUI environment variables</summary>

| Variable | Description |
| :----: | --- |
| PIXELFLUX_WAYLAND | Enable Wayland mode with Smithay/Labwc and zero-copy GPU encoding |
| SELKIES_DESKTOP | Show a simple panel in Wayland mode |
| CUSTOM_PORT | Internal HTTP port (default `3000`) |
| CUSTOM_HTTPS_PORT | Internal HTTPS port (default `3001`) |
| CUSTOM_WS_PORT | Internal WebSocket port (default `8082`) |
| CUSTOM_USER | HTTP Basic auth username |
| PASSWORD | HTTP Basic auth password |
| DRI_NODE | **Encoding GPU**: `/dev/dri/renderD128` |
| DRINODE | **Rendering GPU**: `/dev/dri/renderD129` |
| AUTO_GPU | Auto-configure first detected GPU for zero-copy |
| SUBFOLDER | Subfolder for reverse proxy, e.g. `/hermesos/` |
| TITLE | Page title |
| DASHBOARD | Dashboard style: `selkies-dashboard`, `selkies-dashboard-zinc`, `selkies-dashboard-wish` |
| FILE_MANAGER_PATH | Default upload/download path |
| START_DOCKER | Set to `false` to disable DinD auto-start |
| DISABLE_IPV6 | Disable IPv6 |
| LC_ALL | Desktop locale, e.g. `fr_FR.UTF-8` |
| NO_DECOR | Run without window borders (PWA mode) |
| NO_GAMEPAD | Disable gamepad interposer |
| DISABLE_ZINK | Disable Zink even if a GPU is detected |
| DISABLE_DRI3 | Disable DRI3 acceleration |
| MAX_RES | Maximum virtual resolution, e.g. `3840x2160` |
| WATERMARK_PNG | Path to watermark PNG inside container |
| WATERMARK_LOCATION | Watermark position: 1=TL, 2=TR, 3=BL, 4=BR, 5=Center, 6=Animated |

</details>

<details>
<summary>Click to expand: Optional run configurations (DinD & GPU mounts)</summary>

| Argument | Description |
| :----: | --- |
| `--privileged` | Enables Docker-in-Docker (DinD). For performance, also mount `-v /path/to/docker-data:/var/lib/docker`. |
| `-v /var/run/docker.sock:/var/run/docker.sock` | Mount the host Docker socket to manage host containers from within HermesOS. |
| `--device /dev/dri:/dev/dri` | Pass a GPU into the container. Use with `DRINODE`. |

</details>

---

## Language Support

Launch the desktop in a different language:

* `-e LC_ALL=zh_CN.UTF-8` — Chinese
* `-e LC_ALL=ja_JP.UTF-8` — Japanese
* `-e LC_ALL=ko_KR.UTF-8` — Korean
* `-e LC_ALL=ar_AE.UTF-8` — Arabic
* `-e LC_ALL=ru_RU.UTF-8` — Russian
* `-e LC_ALL=es_MX.UTF-8` — Spanish (Latin America)
* `-e LC_ALL=de_DE.UTF-8` — German
* `-e LC_ALL=fr_FR.UTF-8` — French

---

## Persistence

All AI tool data persists through the `/config` volume mapping:

| Host path (example) | Container path | Contents |
|---|---|---|
| `/path/to/data/.hermes/` | `/config/.hermes/` | Hermes Agent state DB, memory, skills, `.env` |
| `/path/to/data/.hermelin.env` | `/config/.hermelin.env` | HermelinChat credentials |
| `/path/to/data/ai-hedge-fund/.env` | `/config/ai-hedge-fund/.env` | Hedge fund API keys |
| `/path/to/data/.config/` | `/config/.config/` | KDE Plasma settings |
| `/path/to/data/.cache/` | `/config/.cache/` | CloakBrowser Chromium binary cache |

Recreating the container does **not** lose any Hermes memories, chat sessions, or KDE configuration as long as the volume is preserved.

---

## Application Management (Kali Tools)

### PRoot Apps (Persistent)

Native `apt-get install` packages do not persist across container recreations. For persistent Kali tool additions:

```bash
proot-apps install filezilla
```

See the [supported app list](https://github.com/linuxserver/proot-apps?tab=readme-ov-file#supported-apps).

### Native Apps (Non-Persistent)

```yaml
environment:
  - DOCKER_MODS=linuxserver/mods:universal-package-install
  - INSTALL_PACKAGES=libfuse2|nmap|sqlmap
```

---

## Support & Shell Access

```bash
# Open a shell inside the running container
docker exec -it HermesOS /bin/bash

# Stream container logs
docker logs -f HermesOS
```

---

## Unraid

See the dedicated **[Unraid Installation Guide](docs/unraid-installation-guide.md)** for step-by-step instructions including:
- Building or pulling the image in Unraid
- Full container template configuration
- Nvidia GPU passthrough
- Reverse proxy setup (SWAG / Nginx Proxy Manager)
- A ready-to-paste Unraid XML template

---

## Building Locally

```bash
git clone https://github.com/DocwatZ/HermesOS.git
cd HermesOS

# x86-64
docker build --no-cache -t hermesos:latest .

# arm64
docker run --rm --privileged lscr.io/linuxserver/qemu-static --reset
docker build --no-cache -t hermesos:latest -f Dockerfile.aarch64 .
```

---

## Versions

* **25.05.26:** — Inject hermes-agent, hermelinChat, CloakBrowser, and ai-hedge-fund. Add HermelinChat autostart service and first-run config scaffold. Expose port 3000.
* **29.03.26:** — Make Wayland default disable with `PIXELFLUX_WAYLAND=false` (upstream).
* **28.12.25:** — Add Wayland init logic (upstream).
* **19.06.25:** — Rebase to Selkies baseimage (upstream).
* **24.01.25:** — Fix SVG icons not rendering (upstream).
* **18.07.24:** — Initial upstream Kali Linux release.
