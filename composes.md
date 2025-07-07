# Combined Self-Hosted Services with Docker Compose

This document provides a unified `docker-compose.yml` configuration for deploying and managing a collection of self-hosted services. It utilizes pinned image versions for stability, Caddy for automatic HTTPS via reverse proxy, and `ddclient` for dynamic DNS updates through Cloudflare.

## ✨ Services Included

*   **Ollama:** Run large language models locally.
*   **AnythingLLM:** A private RAG (Retrieval Augmented Generation) solution that can utilize Ollama.
*   **OwnCloud:** File hosting and sharing platform (includes MariaDB as its database).
    *   > **Note:** Redis caching is omitted in this specific configuration for simplicity but is recommended for better performance in a production setup.
*   **SyncThing:** Continuous, decentralized file synchronization.
    *   > **Note:** Uses host networking for improved discovery.
*   **Portainer CE:** Management UI for Docker environments.
*   **Caddy:** Automatic HTTPS reverse proxy.
    *   > **Note:** Configured for Cloudflare DNS challenge, which may require a specific Caddy image build that includes the Cloudflare plugin.
*   **ddclient:** Dynamic DNS updater for Cloudflare.
    *   > **Note:** Uses host networking to accurately detect the public IP address.

---

## ⚠️ Important Notes and Changes

*   **Image Pinning:** Most services are pinned to specific image versions to ensure stability and predictability. It is crucial to schedule regular checks for security updates and new versions for these images.
*   **Host Networking:** `SyncThing` and `ddclient` are configured with `network_mode: host`. This allows them better network discovery (Syncthing) or accurate IP detection (ddclient) but reduces container isolation and means Caddy must proxy `Syncthing` via `localhost` rather than a container network alias.
*   **Caddy & Cloudflare DNS Challenge:** This setup assumes the use of Cloudflare's DNS challenge for obtaining SSL certificates with Caddy. The standard `caddy:alpine` image **does not** include the necessary Cloudflare DNS plugin by default. Refer to Configuration Step 4 for details on using a compatible image.
*   **OwnCloud Reverse Proxy Configuration:** Redis is not included in this version. For OwnCloud to operate correctly behind Caddy, essential environment variables related to reverse proxying (e.g., `OWNCLOUD_OVERWRITE*`) must be configured in the `owncloud` service definition within the `docker-compose.yml` file.

---

## Prerequisites

1.  **Docker Engine:** [Install Docker](https://docs.docker.com/engine/install/)
2.  **Docker Compose:** [Install Docker Compose](https://docs.docker.com/compose/install/)
3.  **Cloudflare Account & Domain:** A domain name managed through Cloudflare.
4.  **Cloudflare API Token (for ddclient):** Requires `Zone:DNS:Edit` permissions for your domain's zone. [How to create a Cloudflare API token](https://developers.cloudflare.com/fundamentals/api/reference/create-token/).
5.  **Cloudflare API Token (for Caddy ACME DNS Challenge):** If using the DNS challenge with Caddy, a separate API token might be needed, typically scoped with `Zone:DNS:Read` and `Zone:Zone:Read` permissions.
6.  **Host Directories:** Specific directories must be created on the host machine for persistent data and configuration (detailed in the Configuration section).
7.  **(Optional) NVIDIA Docker Toolkit:** Required if you intend to use GPU acceleration for Ollama. Relevant sections in the `docker-compose.yml` will need to be uncommented.

---

## ⚙️ Configuration Steps

**Critical:** Review and customize all configurations *before* attempting to start the services with `docker-compose up`.

### 1. Obtain Files
Clone the repository or manually create the `docker-compose.yml`, `Caddyfile`, and `.env` file structure locally.

### 2. Create `.env` File
In the same directory as your `docker-compose.yml`, create a file named `.env`. This file will store sensitive information and environment-specific settings.
**Never commit this file to version control (e.g., Git).**

```dotenv
# .env

# === General Settings ===
# Specify your local timezone (e.g., America/New_York, Europe/London)
# List of timezones: https://en.wikipedia.org/wiki/List_of_tz_database_time_zones
TZ=America/New_York

# === OwnCloud Configuration ===
# Your external domain for OwnCloud (e.g., cloud.yourdomain.com)
OWNCLOUD_DOMAIN=owncloud.example.com
# Administrator credentials for OwnCloud
OWNCLOUD_ADMIN_USERNAME=admin
OWNCLOUD_ADMIN_PASSWORD=changeme_secure_owncloud_admin_password
# Database password for OwnCloud (must match MARIADB_PASSWORD below)
OWNCLOUD_DB_PASSWORD=changeme_secure_owncloud_db_password
# Root password for the MariaDB instance
MARIADB_ROOT_PASSWORD=changeme_secure_mariadb_root_password

# === SyncThing Configuration ===
# User ID (PUID) and Group ID (PGID) for Syncthing container.
# These should match the owner of the host directories mapped to Syncthing.
# Use `id -u` and `id -g` on your host to find appropriate values.
SYNCTHING_PUID=1000
SYNCTHING_PGID=1000

# === Caddy Configuration ===
# Email address for Let's Encrypt registration and notifications
ACME_EMAIL=your-email@example.com
# Cloudflare API Token for Caddy's DNS challenge (if using a Caddy image with the Cloudflare plugin)
# The environment variable name (here CLOUDFLARE_API_TOKEN) must match what the Caddy plugin expects.
CLOUDFLARE_API_TOKEN=YOUR_CADDY_ACME_CLOUDFLARE_API_TOKEN

# === ddclient Configuration (Dynamic DNS) ===
# Your Cloudflare account login email address
DDCLIENT_CLOUDFLARE_EMAIL=your-cloudflare-login-email@example.com
# Your Cloudflare API Token (must have Zone:DNS:Edit permissions for the specified zone)
DDCLIENT_CLOUDFLARE_API_TOKEN=YOUR_DDCLIENT_DNS_EDIT_CLOUDFLARE_API_TOKEN
# The Cloudflare Zone name (e.g., yourdomain.com)
DDCLIENT_CLOUDFLARE_ZONE=example.com
# Comma-separated list of DNS records within the zone to update (e.g., "@,www,cloud,sync")
# This should include ALL hostnames managed by Caddy, and potentially the root record (@).
DDCLIENT_CLOUDFLARE_RECORDS=owncloud,syncthing,portainer,anythingllm

# === Ollama Configuration (Optional) ===
# Duration Ollama keeps models loaded in memory (e.g., 24h, 1h, 5m)
OLLAMA_KEEP_ALIVE=24h
```
After creating the `.env` file, set restrictive permissions:
```bash
chmod 600 .env
```

### 3. Prepare Host Directories
Create the necessary directories on your host machine. These paths are relative to your `docker-compose.yml` file.
```bash
mkdir -p ./syncthing_config
mkdir -p ./syncthing_data # This will be mapped to /var/syncthing/Sync inside the Syncthing container
# Create other subdirectories for Syncthing data if you plan to map them explicitly.
```
Ensure that the ownership of `./syncthing_config` and `./syncthing_data` (and any other Syncthing data directories) matches the `SYNCTHING_PUID` and `SYNCTHING_PGID` specified in your `.env` file. You can use `sudo chown -R $USER:$USER ./syncthing_config ./syncthing_data` if your current user's UID/GID matches, or `sudo chown -R 1000:1000 ./syncthing_config ./syncthing_data` if using UID/GID 1000.

### 4. Configure Caddy

*   **(IMPORTANT) Choose the Correct Caddy Image:**
    *   **If using Cloudflare DNS Challenge:** You *must* use a Caddy image that includes the Cloudflare DNS plugin. The standard `caddy:alpine` or `caddy:latest` images likely do **not** have it. Change `image: caddy:2.7.6-alpine` in `docker-compose.yml` to a specialized build, for example: `image: ghcr.io/caddy-dns/cloudflare:latest` or a version-pinned equivalent like `image: ghcr.io/caddybuilds/caddy-cloudflare:2.7.6`. Verify the exact image name and tag from a trusted source for Caddy builds with DNS plugins. Ensure the `CLOUDFLARE_API_TOKEN` variable name in your `.env` file and the `caddy` service's environment section in `docker-compose.yml` matches the one expected by the plugin (usually `CLOUDFLARE_API_TOKEN`).
    *   **If NOT using Cloudflare DNS Challenge:** You can use the standard `caddy:<version>-alpine` image. Remove the `CLOUDFLARE_API_TOKEN` (or similarly named) environment variable from the `caddy` service definition in `docker-compose.yml`. Caddy will then attempt certificate acquisition using HTTP or TLS-ALPN challenges, which require ports 80/443 to be directly accessible from the internet.

*   **Create `Caddyfile`:**
    Create a `Caddyfile` in the same directory as `docker-compose.yml`. **Replace all placeholder values (like `yourdomain.com`) with your actual domain information.**

    ```caddy
    # Caddyfile
    # This path should be relative to where you run docker-compose,
    # or an absolute path if preferred and mounted correctly.

    {
        email {$ACME_EMAIL}
        # Optional: Uncomment for testing against Let's Encrypt's staging environment
        # to avoid rate limits on the production CA.
        # acme_ca https://acme-staging-v02.api.letsencrypt.org/directory

        # Optional: Enable DNS Challenge if using a Caddy image with the Cloudflare plugin.
        # Ensure the environment variable {$CLOUDFLARE_API_TOKEN} is correctly passed from .env.
        # acme_dns cloudflare {$CLOUDFLARE_API_TOKEN}
    }

    # --- Service Definitions ---
    # Replace *.yourdomain.com with your actual hostnames.
    # These hostnames should correspond to the records listed in DDCLIENT_CLOUDFLARE_RECORDS in your .env file.

    # Portainer (Accessible via proxy_network)
    portainer.yourdomain.com {
        reverse_proxy portainer:9443
    }

    # OwnCloud (Accessible via proxy_network)
    # Ensure OWNCLOUD_DOMAIN in .env matches the hostname used here.
    owncloud.yourdomain.com {
        # Required headers for OwnCloud behind a reverse proxy.
        # Always verify these against the latest OwnCloud documentation.
        header_up X-Real-IP {remote_host}
        header_up X-Forwarded-For {remote_host}
        header_up X-Forwarded-Host {host}
        header_up X-Forwarded-Proto {scheme}
        # The following might also be needed depending on your OwnCloud version and setup:
        # header_up X-Forwarded-Port {server_port}

        reverse_proxy owncloud:8080

        # Add well-known redirects for CalDAV/CardDAV if needed, e.g.:
        # redir /.well-known/carddav /remote.php/dav 301
        # redir /.well-known/caldav /remote.php/dav 301
    }

    # SyncThing (Uses host networking)
    # Caddy proxies to localhost because Syncthing binds directly to the host's port.
    syncthing.yourdomain.com {
        reverse_proxy localhost:8384 # <-- IMPORTANT: Syncthing is on the host network
    }

    # AnythingLLM (Accessible via proxy_network)
    anythingllm.yourdomain.com {
        reverse_proxy anythingllm:3001
    }

    # Ollama API (Optional - on default network, proxied via Caddy)
    # If you expose the Ollama API externally, SECURE IT!
    # ollama.yourdomain.com {
    #    # Example: Basic Authentication (replace user and hashed_password)
    #    # basicauth {
    #    #    user JDJhJDEwJExqWER....
    #    # }
    #    reverse_proxy ollama:11434
    # }
    ```

### 5. Review OwnCloud Specific Configuration
*   **Add Redis (Recommended for Production):** For improved performance, consider adding a Redis service to your `docker-compose.yml` and configuring the necessary OwnCloud environment variables (e.g., `OWNCLOUD_REDIS_ENABLED=true`, `OWNCLOUD_REDIS_HOST=redis_service_name`). This is omitted here for simplicity.
*   **Add `OWNCLOUD_OVERWRITE*` Variables:** Ensure the following environment variables are present in the `owncloud` service definition within `docker-compose.yml`. Adjust the `OWNCLOUD_OVERWRITECONDADDR` regex if your Docker network's IP range differs. This regex is typically for networks like `172.17.0.0/16` through `172.31.0.0/16`.
    ```yaml
      environment:
        # ... other OwnCloud variables
        - OWNCLOUD_OVERWRITEHOST=${OWNCLOUD_DOMAIN}
        - OWNCLOUD_OVERWRITEPROTOCOL=https
        - OWNCLOUD_OVERWRITEWEBROOT=/
        # This regex allows requests from Caddy's Docker network.
        # Adjust if your proxy network uses a different range.
        - OWNCLOUD_OVERWRITECONDADDR=^172\.([1-3][0-9]|4[0-4])\.0\..*$
        # ... other OwnCloud variables
    ```
*   **Add Other Database Variables (Optional but Recommended):** For clarity and to avoid reliance on defaults, explicitly add all database connection variables to the `owncloud` service:
    ```yaml
      environment:
        # ...
        - OWNCLOUD_DB_TYPE=mysql
        - OWNCLOUD_DB_HOST=owncloud_mariadb # Service name of MariaDB container
        - OWNCLOUD_DB_NAME=owncloud         # Default DB name
        - OWNCLOUD_DB_USERNAME=owncloud     # Default DB user
        - OWNCLOUD_DB_PASSWORD=${OWNCLOUD_DB_PASSWORD} # From .env
        # ...
    ```

### 6. (Optional) Enable GPU for Ollama
If you have an NVIDIA GPU and have set up the NVIDIA Docker Toolkit, uncomment the `deploy` section under the `ollama` service in `docker-compose.yml` to enable GPU acceleration.

---

## 🚀 Usage Instructions

### 1. Start Services
Navigate to the directory containing your `docker-compose.yml` file and run:
```bash
docker-compose up -d
```
The `-d` flag runs the containers in detached mode (in the background).

### 2. Verify `ddclient` Operation
Check the logs for `ddclient` to ensure it's successfully updating your Cloudflare DNS records:
```bash
docker-compose logs -f ddclient
```
Look for messages indicating successful connection to Cloudflare and updates for your specified records.

### 3. Access Services
Once DNS propagation is complete (which can take a few minutes), you should be able to access your services via the hostnames configured in your `Caddyfile` (e.g., `https://owncloud.yourdomain.com`). Caddy will handle automatic HTTPS.

### 4. View Logs
To view logs for all running services:
```bash
docker-compose logs -f
```
To view logs for a specific service (e.g., `caddy`):
```bash
docker-compose logs -f caddy
```

### 5. Stop Services
To stop and remove the containers, networks, and (optionally) volumes:
```bash
docker-compose down
```
> **Warning:** To also remove named volumes (which will delete persistent data for services like Ollama, OwnCloud, etc.), use:
> `docker-compose down -v`
> **Use this with extreme caution as it leads to data loss if not intended.**

---

## 💾 Volumes and Data Persistence

*   **Named Volumes:** Services like `ollama` (`ollama_data`), `portainer` (`portainer_data`), `owncloud` (`owncloud_files`, `owncloud_mysql_data`), etc., use named Docker volumes. These volumes persist data even if containers are removed and recreated, unless explicitly deleted (e.g., with `docker-compose down -v` or `docker volume rm <volume_name>`).
*   **Bind Mounts:** Configurations like `./syncthing_config`, `./syncthing_data`, and `./Caddyfile` are bind mounts. These link directories or files from your host machine directly into the containers. Data in bind mounts persists on the host.

---

## 🌐 Networking Overview

*   **`host` Network:** Used by `ddclient` and `Syncthing`. This gives them direct access to the host's network interfaces, which is beneficial for IP detection (`ddclient`) and peer-to-peer discovery (`Syncthing`).
*   **`default` Network:** An internal Docker bridge network automatically created by Compose. Backend services like `ollama` and `owncloud_mariadb` primarily use this for communication among themselves, not typically exposed externally except through the proxy.
*   **`proxy_network` Network:** An explicitly defined Docker bridge network. Caddy is connected to this network, as are the frontend services it reverse proxies (e.g., `portainer`, `anythingllm`, `owncloud`). This allows Caddy to route traffic to these services using their service names as hostnames.

---

## Customization Notes

Feel free to adapt the `docker-compose.yml`, `.env`, and `Caddyfile` to your specific requirements. Remember to:
*   Regularly back up your persistent data (named volumes and important bind mounts).
*   Schedule checks for updates to the pinned Docker image versions and test updates in a non-production environment if possible.

---
<!-- Note: The docker-compose.yml content is below this line -->
```yaml
# Homelab Docker Compose Configuration
# Version: 3.8
#
# Purpose: Manages a suite of self-hosted services, utilizing Caddy for
#          reverse proxying with automatic HTTPS and ddclient for dynamic DNS
#          updates via Cloudflare.
#
# Network Strategy:
#   - Publicly accessible services are exposed through Caddy via the 'proxy_network'.
#   - Backend/internal services (like databases, Ollama) communicate on the 'default' Docker network.
#   - Syncthing and ddclient use 'network_mode: host' for optimal P2P performance and
#     accurate public IP detection, respectively.
#
# Image Update Strategy:
#   - Images are pinned to specific versions for stability and predictable deployments.
#   - Manual checks for updates should be performed periodically (e.g., quarterly).
#   - Always back up persistent volumes before upgrading image versions.

version: '3.8'

services:
  # --- Artificial Intelligence Services ---
  ollama:
    image: ollama/ollama:0.1.34  # Pinned version - check for updates
    container_name: ollama
    restart: unless-stopped
    user: "1000:1000"  # Recommended: Run as non-root; match host UID/GID for volume permissions
    environment:
      - OLLAMA_KEEP_ALIVE=${OLLAMA_KEEP_ALIVE:-24h} # Time to keep models loaded
      - TZ=${TZ:-America/New_York}                 # Timezone from .env
    volumes:
      - ollama_data:/root/.ollama  # Stores downloaded models and data
    networks:
      - default  # Primarily for internal access, e.g., from AnythingLLM
    # deploy: # Optional: Uncomment for GPU acceleration (NVIDIA)
    #   resources:
    #     reservations:
    #       devices:
    #         - driver: nvidia
    #           count: 1
    #           capabilities: [gpu]
    #     limits: # Optional: Resource limits
    #       cpus: '2'
    #       memory: 8G
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:11434"]
      interval: 30s
      timeout: 5s
      retries: 3

  anythingllm:
    image: mintplexlabs/anythingllm:1.5.1  # Pinned version - check for updates
    container_name: anythingllm
    restart: unless-stopped
    depends_on:
      ollama:
        condition: service_healthy # Wait for Ollama to be ready
    volumes:
      - anythingllm_storage:/app/server/storage # Persistent storage for AnythingLLM
      - anythingllm_hotdir:/app/collector/hotdir # Directory for document ingestion
    environment:
      - OLLAMA_BASE_URL=http://ollama:11434 # Internal URL to Ollama service
      - STORAGE_DIR=/app/server/storage
      - TZ=${TZ:-America/New_York}
    networks:
      - default       # For communication with Ollama
      - proxy_network # To be exposed via Caddy

  # --- File Hosting and Synchronization Services ---
  owncloud:
    image: owncloud/server:10.13.0  # Pinned LTS version - check for updates
    container_name: owncloud_server
    restart: unless-stopped
    depends_on:
      owncloud_mariadb:
        condition: service_healthy
      # owncloud_redis: # Uncomment if Redis is added
      #   condition: service_healthy
    environment:
      - OWNCLOUD_DOMAIN=${OWNCLOUD_DOMAIN} # External domain from .env
      - OWNCLOUD_TRUSTED_DOMAINS=${OWNCLOUD_DOMAIN} # Trust the external domain
      - ADMIN_USERNAME=${OWNCLOUD_ADMIN_USERNAME} # Admin user from .env
      - ADMIN_PASSWORD=${OWNCLOUD_ADMIN_PASSWORD} # Admin password from .env
      - OWNCLOUD_DB_TYPE=mysql
      - OWNCLOUD_DB_HOST=owncloud_mariadb
      - OWNCLOUD_DB_NAME=owncloud
      - OWNCLOUD_DB_USERNAME=owncloud
      - OWNCLOUD_DB_PASSWORD=${OWNCLOUD_DB_PASSWORD}
      - OWNCLOUD_OVERWRITEHOST=${OWNCLOUD_DOMAIN} # For reverse proxy
      - OWNCLOUD_OVERWRITEPROTOCOL=https     # For reverse proxy
      - OWNCLOUD_OVERWRITEWEBROOT=/          # For reverse proxy
      - OWNCLOUD_OVERWRITECONDADDR=^172\.([1-3][0-9]|4[0-4])\.0\..*$ # Adjust regex if proxy network differs
      - TZ=${TZ:-America/New_York}
      # - OWNCLOUD_REDIS_ENABLED=true # Uncomment if Redis is added
      # - OWNCLOUD_REDIS_HOST=owncloud_redis # Uncomment if Redis is added
    volumes:
      - owncloud_files:/mnt/data # User files and application data
    networks:
      - default       # For database connection
      - proxy_network # To be exposed via Caddy

  owncloud_mariadb:
    image: mariadb:10.11.6  # Version compatible with OwnCloud - check OwnCloud docs for updates
    container_name: owncloud_mariadb
    restart: unless-stopped
    environment:
      - MYSQL_ROOT_PASSWORD=${MARIADB_ROOT_PASSWORD} # Root DB password from .env
      - MYSQL_DATABASE=owncloud # Default DB name for OwnCloud
      - MYSQL_USER=owncloud     # Default DB user for OwnCloud
      - MYSQL_PASSWORD=${OWNCLOUD_DB_PASSWORD} # DB user password from .env
      - TZ=${TZ:-America/New_York}
    volumes:
      - owncloud_mysql_data:/var/lib/mysql # Persistent database storage
    networks:
      - default # Internal network for OwnCloud app to connect
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-u", "root", "--password=${MARIADB_ROOT_PASSWORD}"]
      interval: 10s
      timeout: 5s
      retries: 5

  # owncloud_redis: # Uncomment to add Redis for caching
  #   image: redis:7.2-alpine # Pinned version - check for updates
  #   container_name: owncloud_redis
  #   restart: unless-stopped
  #   networks:
  #     - default

  syncthing:
    image: syncthing/syncthing:1.27.7  # Pinned stable version - check for updates
    container_name: syncthing
    hostname: homelab-syncthing # Custom hostname within its network context
    restart: unless-stopped
    network_mode: host  # Uses host network for better P2P discovery & NAT traversal
    environment:
      - PUID=${SYNCTHING_PUID:-1000} # User ID from .env
      - PGID=${SYNCTHING_PGID:-1000} # Group ID from .env
      - TZ=${TZ:-America/New_York}
    volumes:
      - ./syncthing_config:/var/syncthing/config # Configuration files (bind mount)
      - ./syncthing_data:/var/syncthing/Sync    # Main data sync folder (bind mount)
      # Add more volumes here for other directories you want Syncthing to manage
      # - /path/on/host/photos:/var/syncthing/Photos
    # deploy: # Optional: Resource limits
    #   resources:
    #     limits:
    #       memory: 2G

  # --- Infrastructure and Management Services ---
  caddy:
    # IMPORTANT: Use an image with the Cloudflare DNS plugin if using DNS challenge.
    # Example: image: ghcr.io/caddy-dns/cloudflare:latest
    # Or a version-pinned one: image: ghcr.io/caddybuilds/caddy-cloudflare:2.7.6
    image: caddy:2.7.6-alpine  # Standard Alpine version (no DNS plugins by default)
    container_name: caddy
    restart: unless-stopped
    ports:
      - "80:80"    # HTTP
      - "443:443"  # HTTPS
      - "443:443/udp" # For HTTP/3
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro # Caddy configuration (bind mount, read-only)
      - caddy_data:/data                   # SSL certificates and other Caddy data
      - caddy_config:/config               # Caddy operational config
    environment:
      # Ensure this variable name matches what your Caddy Cloudflare plugin expects, if used.
      - CLOUDFLARE_API_TOKEN=${CLOUDFLARE_API_TOKEN} # API token from .env (for DNS challenge)
      - ACME_EMAIL=${ACME_EMAIL}                     # Email for Let's Encrypt from .env
      - TZ=${TZ:-America/New_York}
    networks:
      - proxy_network # Connects to services it will proxy
    cap_add:
      - NET_BIND_SERVICE # Allows Caddy to bind to privileged ports (80, 443)

  ddclient:
    image: linuxserver/ddclient:3.11.2-ls1 # Pinned version - check linuxserver.io for updates
    # The original compose file had image: linuxserver/ddclient:6.12.0.
    # Version 3.11.2-ls1 is a common stable tag for linuxserver ddclient.
    # Please verify the latest stable tag from linuxserver.io if 6.12.0 was intended and available.
    container_name: ddclient
    restart: unless-stopped
    network_mode: host # Uses host network to accurately determine the public IP
    environment:
      - PUID=1000 # Optional: User ID, if needed for config file permissions
      - PGID=1000 # Optional: Group ID
      - TZ=${TZ:-America/New_York}
      # ddclient specific config is usually done via a ddclient.conf file
      # For Cloudflare via environment variables with this image:
      - DDCLIENT_PROTOCOL=cloudflare
      - DDCLIENT_LOGIN=${DDCLIENT_CLOUDFLARE_EMAIL}    # Cloudflare email from .env
      - DDCLIENT_PASSWORD=${DDCLIENT_CLOUDFLARE_API_TOKEN} # Cloudflare API token for ddclient from .env
      - DDCLIENT_ZONE=${DDCLIENT_CLOUDFLARE_ZONE}      # Cloudflare zone from .env
      - DDCLIENT_RECORDS=${DDCLIENT_CLOUDFLARE_RECORDS}  # Records to update from .env
    volumes:
      - ddclient_config:/config # Persistent configuration and cache for ddclient

  portainer:
    image: portainer/portainer-ce:2.20.2 # Pinned CE version - check for updates
    container_name: portainer
    restart: unless-stopped
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro # Access to Docker socket (read-only for safety)
      - portainer_data:/data                         # Persistent Portainer data
    networks:
      - proxy_network # To be exposed via Caddy

volumes:
  ollama_data:
  anythingllm_storage:
  anythingllm_hotdir:
  owncloud_files:
  owncloud_mysql_data:
  # owncloud_redis_data: # Uncomment if Redis is added
  portainer_data:
  caddy_data:
  caddy_config:
  ddclient_config: # Changed from ddclient_cache for clarity in the original, ensure consistency

networks:
  default: # Default internal bridge network for backend communication
    driver: bridge
  proxy_network: # Explicit bridge network for services exposed via Caddy
    driver: bridge
```
