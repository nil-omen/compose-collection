# Uptime Kuma Dual Deployment

This repository contains the configuration to deploy two independent instances of [Uptime Kuma](https://github.com/louislam/uptime-kuma), a self-hosted monitoring tool. Both instances are configured with access to the host's Docker socket to enable native container monitoring.

## 📁 Directory Structure

Ensure your project directory looks like this before starting:

```text
your-deployment-folder/
├── docker-compose.yml
├── .env
└── README.md
```

## ⚙️ Prerequisites

* Docker and Docker Compose installed on the host machine.
* The required host directories for data storage must be created or Docker will create them with root permissions.

## 🔧 Environment Variables (.env)

Create a `.env` file in the same directory as the `docker-compose.yml` file. This file manages the storage paths and timezone settings.

Example `.env`:
```ini
# The base directory on your host machine where all data will be stored.
DATA_DIR=/opt/uptime-kuma-data

# Time Zone setting for accurate logging and monitoring schedules
TZ=Africa/Cairo
```

## 🚀 Deployment Instructions

**1. Create the data directories:**
Before starting the containers, create the folders where the bind mounts will store your persistent data:
```bash
sudo mkdir -p /opt/uptime-kuma-data/main
sudo mkdir -p /opt/uptime-kuma-data/t3mia
```

**2. Start the services:**
Navigate to the directory containing your `docker-compose.yml` and run:
```bash
docker compose up -d
```

**3. Updating:**
To update to the latest version of the `louislam/uptime-kuma:2` image:
```bash
docker compose pull
docker compose up -d
```

## 🌐 Accessing the Interfaces

Once deployed, the services will be available at:

* **Main Instance:** `http://<YOUR_SERVER_IP>:3001`
* **T3mia Instance:** `http://<YOUR_SERVER_IP>:3002`

## 🐳 Docker Container Monitoring

The main instance have read-only access to the host's Docker socket (`/var/run/docker.sock`). This allows Uptime Kuma to monitor the status of other Docker containers running on this server.

- Optionally: you might bind the host podman container to the container's docker socket to be able to monitor podman containers.

**How to set up a Docker monitor:**
1. Open the Uptime Kuma web interface.
2. Click **Add New Monitor**.
3. Change the **Monitor Type** to `Docker Container`.
4. In the **Container Name / ID** field, type the exact `container_name` of the service you want to monitor (e.g., `jellyfin`, `portainer`).
5. Set the **Docker Daemon** field to `/var/run/docker.sock`.
6. Save the monitor.
```
