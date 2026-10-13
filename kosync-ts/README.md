# KOReader Sync Server (KOSync) with Tailscale Sidecar Configuration

This Docker Compose deployment sets up the official **[KOReader Sync Server (kosync)](https://github.com/koreader/koreader-sync-server)** with a **Tailscale sidecar container**. It securely exposes your sync server over your private Tailscale network (Tailnet) with automatic HTTPS termination using Tailscale Serve, without requiring open firewall ports or exposing services to the public internet.

This configuration is optimized for synchronizing reading progress, bookmarks, and statistics across multiple devices running [KOReader](https://koreader.rocks/)—such as an **Android phone** and a **jailbroken Kindle**—hosted on your Fedora server.

---

## Architecture Overview

```
                      ┌────────────────────────────────────────────────────────┐
                      │ Fedora Host Container Stack                            │
                      │                                                        │
                      │   ┌────────────────────────────────────────────────┐   │
                      │   │ ts-kosync (Tailscale Sidecar)                  │   │
                      │   │  - Connects to Tailnet (hostname: kosync)       │   │
                      │   │  - Tailscale Serve (HTTPS :443 with TLS cert)  │   │
                      │   │  - Proxies to http://127.0.0.1:17200           │   │
                      │   └───────────────────────┬────────────────────────┘   │
                      │                           │ network_mode:              │
                      │                           │ service:tailscale          │
                      │   ┌───────────────────────▼────────────────────────┐   │
                      │   │ app-kosync (KOReader Sync Server)              │   │
                      │   │  - OpenResty + Lua Gin API (:17200)            │   │
                      │   │  - Embedded Redis database persistence         │   │
                      │   └────────────────────────────────────────────────┘   │
                      └────────────────────────────────────────────────────────┘
                               ▲                                ▲
                               │ (Tailnet HTTPS :443)           │ (Home Wi-Fi :17200 optional)
                               │                                │
                      ┌────────┴────────┐              ┌────────┴────────┐
                      │  Android Phone  │              │ Jailbroken      │
                      │ (Tailscale App) │              │ Kindle          │
                      └─────────────────┘              └─────────────────┘
```

- **Sidecar Networking:** The `app-kosync` container shares the network stack of `ts-kosync` via `network_mode: service:tailscale`.
- **TLS Termination:** Tailscale Serve listens on port `443` of your Tailnet node, provides a trusted TLS certificate for your MagicDNS domain (e.g. `https://kosync.<tailnet-name>.ts.net`), and reverse-proxies requests directly to `http://127.0.0.1:17200`.
- **Data Persistence:** User credentials and book progress are stored in Redis at `./kosync-data/redis`.

---

## Prerequisites

1. **Tailscale Auth Key:**
   - Generate an auth key from the [Tailscale Admin Console](https://login.tailscale.com/admin/settings/keys).
   - Recommended settings: **Reusable** and **Ephemeral**.
2. **MagicDNS & HTTPS:**
   - Ensure MagicDNS and HTTPS Certificates are enabled in your [Tailscale DNS Settings](https://login.tailscale.com/admin/dns).

---

## Quick Start

### 1. Configure the `.env` File

Open `.env` and fill in your Tailscale Auth Key:

```env
# Tailscale Configuration
TS_AUTHKEY=tskey-auth-kXXXXX-XXXXXXXXXXXXXXXX

# Network & Hostname Configuration
SERVICE=kosync
IMAGE_URL=koreader/kosync:latest
ENABLE_USER_REGISTRATION=true
DATA_DIR=.
```

### 2. Start the Stack

Run the following command in the `kosync-ts` directory:

```bash
docker compose up -d
```

Check the status of both containers:

```bash
docker compose ps
```

Verify that Tailscale has joined your network and the healthcheck passes:

```bash
docker compose logs -f
```

---

## Configuring KOReader on Your Devices

### Device 1: Android Phone

1. Install and connect the **Tailscale** app on your Android device so you are connected to your Tailnet.
2. Open **KOReader**.
3. Open any book or document to reveal the top reader menu.
4. Tap the **Settings icon (gear ⚙️)** > **Progress sync** > **Custom sync server**.
5. Enter your Tailscale Serve address:
   ```text
   https://kosync.meteor-alphard.ts.net
   ```
6. Under **Progress sync**, tap **Register / Login**:
   - Choose a username and password.
   - Tap **Register** to create your account on the server.
7. Tap **Sync reading progress automatically** to turn automatic synchronization on.
8. Test by tapping **Push progress from this device now**.

---

### Device 2: Jailbroken Kindle

Kindles run a customized Linux environment. You have three flexible options to connect your Kindle to your server:

#### Option A: Run Tailscale directly on the Kindle (Recommended for on-the-go sync)
If you want to sync while away from home (e.g., using phone hotspot or public Wi-Fi):
1. Install the community [koreader-tailscale](https://github.com/vrb/koreader-tailscale) plugin or KUAL Tailscale extension on your jailbroken Kindle.
2. Authenticate the Kindle into your Tailscale network (`meteor-alphard.ts.net`).
3. In KOReader on the Kindle, go to **Settings ⚙️** > **Progress sync** > **Custom sync server**.
4. Set the URL to:
   ```text
   https://kosync.meteor-alphard.ts.net
   ```
5. Tap **Register / Login** and log in with the **same account** created on your Android device.

#### Option B: Local LAN / Home Wi-Fi (Simplest for home use)
If your Kindle is primarily used on your home Wi-Fi where the Fedora server resides, you can expose the port locally to your LAN:
1. In `compose.yml`, uncomment the `ports:` block under `tailscale`:
   ```yaml
   ports:
     - 0.0.0.0:${SERVICEPORT}:17200
   ```
2. Restart the containers:
   ```bash
   docker compose up -d
   ```
3. Find your Fedora server's local LAN IP (e.g. `ip a` -> `192.168.1.50`).
4. In KOReader on the Kindle, go to **Settings ⚙️** > **Progress sync** > **Custom sync server**.
5. Set the server URL to:
   ```text
   http://192.168.1.50:17200
   ```
6. Tap **Register / Login** and sign in with your credentials.

#### Option C: Tailscale Funnel (Public HTTPS without installing Tailscale on Kindle)
If you want to sync your Kindle over any Wi-Fi network without installing Tailscale on the Kindle itself, you can enable Tailscale Funnel in `compose.yml`:
1. Change `"AllowFunnel"` in `compose.yml` to `true`:
   ```json
   "AllowFunnel":{"$${TS_CERT_DOMAIN}:443":true}
   ```
2. Run `docker compose up -d`.
3. Now `https://kosync.meteor-alphard.ts.net` is accessible from any internet connection, secured with Tailscale's public Let's Encrypt certificate.
4. On your Kindle, configure the custom sync server to `https://kosync.meteor-alphard.ts.net`.

---

## Security & Hardening: Disable New Registrations

Once you have created your user account on KOReader:

1. Edit `.env`:
   ```env
   ENABLE_USER_REGISTRATION=false
   ```
2. Recreate the application container:
   ```bash
   docker compose up -d
   ```

This prevents any other users from registering accounts on your server while allowing your registered devices to continue syncing and authenticating normally.

---

## Useful KOReader Sync Tips

- **Identical Files:** For progress sync to match books across devices, ensure the documents on both devices have identical file contents (KOReader hashes document metadata/content to identify documents).
- **Auto-Sync:** In KOReader, keep **Sync reading progress automatically** checked. KOReader will automatically push your progress whenever you close a book or suspend the device, and pull the latest progress when you open a book.
- **Manual Sync:** If you switch devices immediately, you can trigger a manual sync anytime via **Progress sync** > **Push progress from this device now** or **Pull progress from this device now**.
