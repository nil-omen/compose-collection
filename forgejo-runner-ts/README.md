# Forgejo runner with tailscale sidecar

**First-Time** Registration Command
Before you bring the stack up for the first time with docker `compose up -d`, you need to register the runner and generate the `config.yml`. Here is an adapted version of the registration command to automatically pull the variables from your `.env` file so you don't have to type them out manually:

using Bash:
```Bash
# 1. Source the .env file so Bash can read the variables
source .env

# 2. Run the command using double quotes so Bash injects the variables directly
docker compose run --rm --entrypoint /bin/sh runner -c \
  "/bin/forgejo-runner generate-config > /data/config.yml && \
   /bin/forgejo-runner register \
     --instance https://${FORGEJO_HOSTNAME} \
     --token ${RUNNER_TOKEN} \
     --name ${RUNNER_NAME} \
     --no-interactive"
```

or using Fish:
```sh
docker compose run --rm \
  -e FORGEJO_HOSTNAME \
  -e RUNNER_TOKEN \
  -e RUNNER_NAME \
  --entrypoint /bin/sh runner -c \
  '/bin/forgejo-runner generate-config > /data/config.yml && \
   /bin/forgejo-runner register \
     --instance https://${FORGEJO_HOSTNAME} \
     --token ${RUNNER_TOKEN} \
     --name ${RUNNER_NAME} \
     --no-interactive'
```

Once that command finishes executing successfully, the `config.yml` will be generated in your `./data` directory, and you can start the daemon:

```Bash
docker compose up -d
```

### Choosing labels

[Docker images for runners](https://github.com/catthehacker/docker_images)

1- Open the `/data/config.yml` file you generated earlier.

2- Scroll down to the `runner:` section and find the `labels: []` array.

3- Update it to look exactly like this:

```YAML
runner:
  # ... other settings ...
  
  labels:
      # --- The Rolling Tags (Pulls the absolute newest version automatically) ---
      - "ubuntu-latest:docker://ghcr.io/catthehacker/ubuntu:act-latest"
      - "act-latest:docker://ghcr.io/catthehacker/ubuntu:act-latest"
      
      # --- Specific OS Versions (Deterministic, highly recommended) ---
      - "ubuntu-24.04:docker://ghcr.io/catthehacker/ubuntu:act-24.04"
      - "ubuntu-22.04:docker://ghcr.io/catthehacker/ubuntu:act-22.04"
```

### The Diagnosis: A Tale of Two DNS Resolvers

Line 5: docker pull image=ghcr.io/catthehacker/ubuntu... (Success)

Line 10: git clone 'https://data.forgejo.org/actions/checkout' (Failed: Could not resolve host)

Why did it successfully download a public Docker image from the internet, but fail to clone a public git repository a second later?

Because two different systems are doing the work:

The Docker Pull is executed by your Fedora host machine's Docker daemon (because we mounted `/var/run/docker.sock`). Fedora server has normal internet access, so it resolved ghcr.io perfectly.

The Git Clone is executed by the Forgejo runner daemon itself (trying to download the Action logic into its `/data/.cache` directory). The runner is trapped inside the runner-ts Tailscale network.

Right now, your Tailscale sidecar is explicitly told to only ask Tailscale's MagicDNS (100.100.100.100) for directions. It knows exactly where your private forgejo.meteor-alphard.ts.net is, but when asked about the public internet (data.forgejo.org), it throws its hands up because it doesn't have a fallback.

#### The Fix
To fix this, we need to tell your Tailscale network how to browse the public internet. The cleanest and most reliable way to do this is globally through your Tailscale admin panel.

1- Go to your Tailscale Admin Console and navigate to the DNS tab.

2- Scroll down to the Global Nameservers section.

3- Click Add nameserver -> Custom... (or select Cloudflare/Google from the quick list).

4- Enter 1.1.1.1 (Cloudflare) or 9.9.9.9 (Quad9) and save it.

5- Crucial Step: Toggle the switch that says Override local DNS.

6- Rerun the compose file to load the dns settings correctly.
```bash
docker compose down
docker compose up -d
```

By doing this, you are telling your MagicDNS (100.100.100.100): "If someone asks for a .ts.net address, answer it yourself. If they ask for anything else, forward the question to 1.1.1.1."

- Next Steps
As soon as you save those settings in the Tailscale console, the changes will push to your sidecar almost instantly.
