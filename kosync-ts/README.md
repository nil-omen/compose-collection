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
1. Refer to the Tailscale guide: [Jailbroken Kindles can now do more with Tailscale](https://tailscale.com/blog/jailbroken-kindle-proxy-tun-modes).
2. Install the community [koreader-tailscale plugin](https://github.com/victoria-riley-barnett/koreader-tailscale) by Victoria Riley-Barnett into KOReader's `plugins/` directory (or use the KUAL Tailscale extension).
3. In KOReader, trigger **Install/Update Tailscale**, authenticate with an auth key, and toggle Tailscale **On**.
4. **Enable the HTTP Proxy:**
   In KOReader, open **Settings ⚙️** > **Network** > **Tailscale VPN** > **Plugin config** and turn on **Automatically configure HTTP proxy** (or manually set **Settings** > **Network** > **Proxy** to `http://127.0.0.1:1056`).
   > [!NOTE]
   > Kindles lack a kernel TUN module, so Tailscale runs in userspace proxy mode. KOReader must route through `127.0.0.1:1056` to reach the Tailnet.
5. In KOReader on the Kindle, go to **Settings ⚙️** > **Progress sync** > **Custom sync server**.
6. Set the URL to your **container's Tailscale IP directly over HTTP port 17200**:
   ```text
   http://100.103.89.2:17200
   ```
   *(Using the direct Tailscale IP on port 17200 is the most reliable method on Kindle because it completely bypasses Kindle DNS resolution and TLS certificate handshake limitations while keeping the connection securely encrypted within your Tailnet).*
   
   Alternatively, if your Kindle resolves MagicDNS properly, you can also use `https://kosync.meteor-alphard.ts.net`.
7. Tap **Register / Login**, enter your username (`king`) and password, and tap **[ Login ]**.
   > [!IMPORTANT]
   > Make sure to tap **[ Login ]**, NOT **[ Register ]**. Because your account was already created from your first device, tapping "Register" will return error 2002, which KOReader displays as a generic "Unknown server error".

#### Option B: Local LAN / Home Wi-Fi (Simplest for home use without Tailscale on Kindle)
If your Kindle is primarily used on your home Wi-Fi where the Fedora server resides, you can expose the port locally to your LAN without running Tailscale on the Kindle:
1. In `compose.yml`, uncomment the `ports:` block under `tailscale`:
   ```yaml
   ports:
     - 0.0.0.0:${SERVICEPORT}:17200
   ```
2. Restart the containers:
   ```bash
   docker compose up -d
   ```
3. Find your Fedora server's local LAN IP (e.g. `ip a` -> `192.168.1.250`).
4. In KOReader on the Kindle, go to **Settings ⚙️** > **Progress sync** > **Custom sync server**.
5. Set the server URL to:
   ```text
   http://192.168.1.250:17200
   ```
6. Tap **Register / Login** and sign in with your credentials.

#### Option C: Tailscale Funnel (Public HTTPS without installing Tailscale on Kindle)
If you want to sync your Kindle over any Wi-Fi network without installing Tailscale on the Kindle itself, you can enable Tailscale Funnel in `compose.yml`:
1. In `compose.yml`, change `"AllowFunnel"` from `false` to `true`:
   ```json
   "AllowFunnel":{"$${TS_CERT_DOMAIN}:443":true}
   ```
2. Run `docker compose up -d`.
3. Now `https://kosync.meteor-alphard.ts.net` is accessible from any internet connection, secured with Tailscale's public Let's Encrypt certificate.
4. On your Kindle, configure the custom sync server to `https://kosync.meteor-alphard.ts.net`.

---

## Frequently Asked Questions

### Can I enable and use all 3 options at the same time?

**Yes, absolutely on the server side!**
You do not have to choose just one on your Fedora server. All 3 methods can run simultaneously:
1. **Tailscale Serve (Option A)** runs on `https://kosync.meteor-alphard.ts.net` for devices inside your Tailnet.
2. **Local Port Mapping (Option B)** can be active at the same time by uncommenting `ports: - 0.0.0.0:17200:17200`, allowing direct connections over your local home Wi-Fi at `http://<fedora-local-ip>:17200`.
3. **Tailscale Funnel (Option C)** can be set to `true` on the same domain, allowing external internet connections without disrupting internal Tailnet traffic.

**Client side (in KOReader):**
Each device has one "Custom sync server" URL field:
- If **Funnel is enabled (Option C)** or **Tailscale is active on the Kindle (Option A)**, you can put `https://kosync.meteor-alphard.ts.net` on **both** your phone and your Kindle, and it will sync seamlessly everywhere (at home and away).
- If you use the local IP `http://192.168.1.50:17200` (Option B), that specific device will only sync while connected to your home Wi-Fi.

---

### How do I cancel / disable Tailscale Funnel?

If you decide you no longer want your sync server accessible publicly via Funnel and want to restrict it strictly to your private Tailnet:

#### Method 1: Via `compose.yml` (Recommended)
1. Edit `compose.yml` and set `"AllowFunnel"` back to `false`:
   ```json
   "AllowFunnel":{"$${TS_CERT_DOMAIN}:443":false}
   ```
2. Re-apply the compose file:
   ```bash
   docker compose up -d
   ```
   This automatically updates the Tailscale Serve configuration inside the container back to `(tailnet only)` mode.

#### Method 2: Via Container CLI
You can immediately disable Funnel from the command line while the container is running:
```bash
docker exec -it ts-kosync tailscale funnel --https=443 off
```
Or reset all funnel rules:
```bash
docker exec -it ts-kosync tailscale funnel reset
```

To confirm that Funnel is off and the service is back to tailnet-only access, run:
```bash
docker exec -it ts-kosync tailscale serve status
```
It will display `(tailnet only)` next to your domain, confirming that external traffic is blocked.

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
