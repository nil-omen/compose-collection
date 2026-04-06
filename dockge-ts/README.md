# Dockge with Tailscale Sidecar

This deployment bundles [Dockge](https://dockge.kuma.pet/), a reactive Docker Compose manager, with a Tailscale sidecar. This setup securely exposes your Dockge dashboard exclusively to your Tailnet via HTTPS using Tailscale Serve, without exposing any ports directly to your local network or the public internet.

## Prerequisites

1. **Tailscale Auth Key:** Generate an Auth Key from the [Tailscale Admin Console](https://login.tailscale.com/admin/settings/keys). It is highly recommended to make it an **Ephemeral** and **Reusable** key.
2. **Stacks Directory:** Dockge requires a specific directory to store and read your `compose.yaml` files. It strictly requires absolute paths that match on both the host and the container.

Run the following command on your server to prepare the directory:
```bash
sudo mkdir -p /opt/stacks
```

Deployment Instructions
Clone or copy these files (compose.yaml and .env) into a directory on your server.


==============

**Edit the .env file:**

Add your Tailscale Auth Key to TS_AUTHKEY.

Update DATA_DIR to set the absolute path for the data directory.

Update STACKS_DIR to the absolute path of your stacks directory (e.g., /opt/stacks).

==============


Deploy the stack:

```Bash
docker compose up -d
```


## How to use your Home Directory
To safely use your home directory, you just need to change two sections in the compose.yml.

Important: You must use the full absolute path (e.g., /home/USER/stacks). Do not use the ~/stacks shortcut, as Docker can sometimes fail to expand the tilde correctly when passing paths through the socket.

1. Update the environment variable:

```YAML
    environment:
      # Tell Dockge where your stacks directory is located
      - DOCKGE_STACKS_DIR=/home/USER/stacks
```

2. Update the volume mount:

```YAML
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ${DATA_DIR}/${SERVICE}/data:/app/data 
      # Left Stacks Path === Right Stacks Path
      - /home/USER/stacks:/home/USER/stacks
```

Once you make that change, Dockge will happily read, write, and deploy your compose files straight from your home directory without any data-routing headaches!
