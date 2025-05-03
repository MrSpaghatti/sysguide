# Combined Self-Hosted Services with Docker Compose

This file provides a unified `docker-compose.yml` file using pinned versions to deploy and manage a collection of self-hosted services. It leverages Caddy for automatic HTTPS and `ddclient` for dynamic DNS updates via Cloudflare.

✨ **Services Included:**

*   **Ollama:** Run large language models locally.
*   **AnythingLLM:** Private RAG solution using Ollama.
*   **OwnCloud:** File hosting and sharing (includes MariaDB). **Note:** Redis caching is omitted in this configuration; add it back for better performance if needed.
*   **SyncThing:** Continuous, decentralized file synchronization (uses host networking).
*   **Portainer CE:** Docker environment management UI.
*   **Caddy:** Automatic HTTPS reverse proxy (configured for Cloudflare DNS challenge - **requires specific Caddy image, see below**).
*   **ddclient:** Dynamic DNS updater for Cloudflare (uses host networking).

---

## ⚠️ Important Notes & Changes

*   **Image Pinning:** Most services use specific image versions for stability. Schedule regular checks for updates.
*   **Host Networking:** `SyncThing` and `ddclient` use `network_mode: host`. This improves their network discovery/IP detection but reduces isolation and requires Caddy to proxy `Syncthing` via `localhost`.
*   **Caddy & Cloudflare:** This setup assumes you want Caddy to use the Cloudflare DNS challenge for SSL certificates. The standard `caddy:alpine` image **does not** include the necessary plugin. See Configuration Step 4.
*   **OwnCloud Configuration:** Redis is not included. Key environment variables for reverse proxy setup (`OWNCLOUD_OVERWRITE*`) need to be added back to the `owncloud` service in `docker-compose.yml` for correct operation behind Caddy.

---

## Prerequisites

1.  **Docker:** [Install Docker](https://docs.docker.com/engine/install/)
2.  **Docker Compose:** [Install Docker Compose](https://docs.docker.com/compose/install/)
3.  **Cloudflare Account & Domain:** A domain managed by Cloudflare.
4.  **Cloudflare API Token (for ddclient):** Needs `Zone:DNS:Edit` permissions for your domain zone. [Create Token](https://developers.cloudflare.com/fundamentals/api/reference/create-token/).
5.  **Cloudflare API Token (for Caddy ACME):** If using the DNS challenge, you likely need a *separate* token scoped appropriately (often `Zone:DNS:Read` and `Zone:Zone:Read`).
6.  **Host Directories:** You will need to create configuration/data directories on the host machine (see Configuration).
7.  **(Optional) NVIDIA Toolkit:** If using GPU acceleration for Ollama (uncomment relevant sections in `docker-compose.yml`).

---

## ⚙️ Configuration

**Critical:** Review and configure *before* running `docker-compose up`.

1.  **Clone Repository / Create Files:** Get `docker-compose.yml` and create the necessary config files and directories locally.
2.  **Create `.env` File:** Create `.env` in the same directory as `docker-compose.yml`. **Never commit this file to Git.**

    ```dotenv
    # === General ===
    # Specify your timezone (e.g., America/New_York, Europe/London)
    # See https://en.wikipedia.org/wiki/List_of_tz_database_time_zones
    TZ=America/New_York

    # === OwnCloud Configuration ===
    # Your external domain (e.g., cloud.yourdomain.com)
    OWNCLOUD_DOMAIN=owncloud.example.com
    # OwnCloud admin user credentials
    OWNCLOUD_ADMIN_USERNAME=admin
    OWNCLOUD_ADMIN_PASSWORD=changeme_secure_owncloud_admin_password
    # OwnCloud database user password (must match MARIADB variable below)
    OWNCLOUD_DB_PASSWORD=changeme_secure_owncloud_db_password
    # MariaDB root user password
    MARIADB_ROOT_PASSWORD=changeme_secure_mariadb_root_password

    # === SyncThing Configuration ===
    # User/Group ID owning host directories mapped into Syncthing (use `id -u`/`id -g`)
    SYNCTHING_PUID=1000
    SYNCTHING_PGID=1000

    # === Caddy Configuration ===
    # Email for Let's Encrypt registration
    ACME_EMAIL=your-email@example.com
    # Cloudflare API Token *for Caddy's DNS challenge* (if using a Caddy image with the plugin)
    # Ensure this matches the variable name expected by your Caddy image (usually CLOUDFLARE_API_TOKEN)
    CLOUDFLARE_API_TOKEN=YOUR_CADDY_ACME_CLOUDFLARE_API_TOKEN

    # === ddclient Configuration (Dynamic DNS) ===
    # Your Cloudflare account email address
    DDCLIENT_CLOUDFLARE_EMAIL=your-cloudflare-login-email@example.com
    # Your Cloudflare API Token (*NEEDS Zone:DNS:Edit permissions*)
    DDCLIENT_CLOUDFLARE_API_TOKEN=YOUR_DDCLIENT_DNS_EDIT_CLOUDFLARE_API_TOKEN
    # The Cloudflare Zone name (your root domain, e.g., example.com)
    DDCLIENT_CLOUDFLARE_ZONE=example.com
    # Comma-separated DNS records within the zone to update (e.g., "@,www,cloud,sync")
    # Should include ALL hostnames managed by Caddy + potentially the root (@)
    DDCLIENT_CLOUDFLARE_RECORDS=owncloud,syncthing,portainer,anythingllm

    # === Ollama Configuration (Optional) ===
    # How long Ollama keeps models loaded (e.g., 24h, 1h, 5m)
    OLLAMA_KEEP_ALIVE=24h
    ```
    *Set appropriate file permissions:* `chmod 600 .env`

3.  **Prepare Host Directories:** Create the necessary directories *relative to* your `docker-compose.yml`:
    ```bash
    mkdir -p ./syncthing_config
    mkdir -p ./syncthing_data # Syncthing volume maps to /var/syncthing/Sync inside container
    # Add other syncthing data subdirectories if you map them explicitly
    ```
    Ensure the ownership of `./syncthing_config` and `./syncthing_data` matches the `SYNCTHING_PUID`/`PGID` set in `.env`.

4.  **Configure Caddy:**
    *   **(IMPORTANT) Choose Caddy Image:** Decide if you need the Cloudflare DNS challenge.
        *   **If YES:** Change `image: caddy:2.7.6-alpine` in `docker-compose.yml` to an image with the plugin, e.g., `image: ghcr.io/caddybuilds/caddy-cloudflare:2.7.6` (or `:latest`). Ensure the `CLOUDFLARE_API_TOKEN` variable name in `.env` and `docker-compose.yml` matches what the plugin expects (usually `CLOUDFLARE_API_TOKEN`).
        *   **If NO:** Remove the `CADDY_CLOUDFLARE_API_TOKEN` (or similar) environment variable from the `caddy` service in `docker-compose.yml`. Caddy will use HTTP or TLS-ALPN challenges.
    *   **Create `Caddyfile`:** Create `Caddyfile` in the same directory. **Replace placeholders.**

        ```caddyfile
        # /path/to/your/project/Caddyfile

        {
            email {$ACME_EMAIL}
            # Optional: Staging CA for testing
            # acme_ca https://acme-staging-v02.api.letsencrypt.org/directory

            # Optional: Enable DNS Challenge if using compatible Caddy image
            # acme_dns cloudflare {$CLOUDFLARE_API_TOKEN}
        }

        # --- Service Definitions ---
        # Replace *.yourdomain.com with your actual domains from DDCLIENT_CLOUDFLARE_RECORDS

        # Portainer (on proxy_network)
        portainer.yourdomain.com {
            reverse_proxy portainer:9443
        }

        # OwnCloud (on proxy_network)
        # Ensure OWNCLOUD_DOMAIN in .env matches here
        owncloud.yourdomain.com {
            # Add back necessary headers/config from OwnCloud docs for reverse proxy
            # Example headers (verify with OwnCloud docs):
             header_up X-Real-IP {remote_host}
             header_up X-Forwarded-For {remote_host}
             header_up X-Forwarded-Host {host}
             header_up X-Forwarded-Port {server_port}
             header_up X-Forwarded-Proto {scheme}
            reverse_proxy owncloud:8080
            # Add back well-known redirects etc. if needed
        }

        # SyncThing (on network_mode: host)
        # Proxy to localhost as Syncthing binds directly to the host port
        syncthing.yourdomain.com {
            reverse_proxy localhost:8384 # <-- IMPORTANT CHANGE
        }

        # AnythingLLM (on proxy_network)
        anythingllm.yourdomain.com {
            reverse_proxy anythingllm:3001
        }

        # Ollama API (Optional - on default network, proxied via Caddy)
        # ollama.yourdomain.com {
        #    # Add authentication (e.g., basicauth) if exposing publicly!
        #    reverse_proxy ollama:11434
        # }
        ```

5.  **Review OwnCloud Configuration:**
    *   **Add Redis Back (Recommended):** If performance matters, add a Redis service back to `docker-compose.yml` and configure OwnCloud environment variables (`OWNCLOUD_REDIS_ENABLED`, `OWNCLOUD_REDIS_HOST`) accordingly.
    *   **Add Overwrite Variables:** Add the following environment variables back to the `owncloud` service in `docker-compose.yml` (adjust `OWNCLOUD_OVERWRITECONDADDR` regex if needed):
        ```yaml
          - OWNCLOUD_OVERWRITEHOST=${OWNCLOUD_DOMAIN}
          - OWNCLOUD_OVERWRITEPROTOCOL=https
          - OWNCLOUD_OVERWRITEWEBROOT=/
          - OWNCLOUD_OVERWRITECONDADDR=^172\.([1-3][0-9]|4[0-4])\.0\..*$
        ```
    *   **Add Other DB Vars:** Add missing DB connection variables for clarity/safety (even if defaults work):
        ```yaml
          - OWNCLOUD_DB_TYPE=mysql
          - OWNCLOUD_DB_NAME=owncloud
          - OWNCLOUD_DB_USERNAME=owncloud
          # OWNCLOUD_DB_PASSWORD is already there
        ```

6.  **(Optional) Enable GPU for Ollama:** Uncomment the `deploy` section if applicable.

---

## 🚀 Usage

1.  **Start Services:**
    ```bash
    docker-compose up -d
    ```

2.  **Check `ddclient`:** Verify DNS updates are working:
    ```bash
    docker-compose logs -f ddclient
    ```
    (Look for success messages regarding Cloudflare updates).

3.  **Access Services:** Use the domains configured in your `Caddyfile` (e.g., `https://owncloud.yourdomain.com`).

4.  **View Logs:**
    ```bash
    docker-compose logs -f # All services
    docker-compose logs -f caddy # Specific service
    ```

5.  **Stop Services:**
    ```bash
    docker-compose down # Stops and removes containers/networks
    # docker-compose down -v # WARNING: Also removes named volumes (DATA LOSS!)
    ```

---

## 💾 Volumes & Data Persistence

*   Named volumes (`ollama_data`, `portainer_data`, etc.) store persistent application data.
*   Bind mounts (`./syncthing_config`, `./syncthing_data`, `./Caddyfile`) link directly to host files/folders.

---

## 🌐 Networking

*   **`host` network:** Used by `ddclient` and `Syncthing` for direct host network access.
*   **`default` network:** Internal bridge network for backend services (Ollama, MariaDB).
*   **`proxy_network` network:** Bridge network connecting Caddy to the services it reverse proxies (Portainer, AnythingLLM, OwnCloud).

---

## Customization

Adapt the `docker-compose.yml`, `.env`, and `Caddyfile` to your needs. Regularly back up volumes, especially before updates.

```yaml
# Homelab Docker Compose v3.8
# Purpose: Self-hosted services with Caddy reverse proxy, DDNS via Cloudflare
# Network Architecture:
#   - Public services exposed via Caddy (proxy_network)
#   - Internal services on default network (Ollama, DBs)
#   - Syncthing uses host networking for better P2P performance
# Update Strategy: 
#   - Manual version pinning for stability (update intentionally)
#   - Backup volumes before updating images

version: '3.8'

services:
  # --- AI Services ---
  ollama:
    image: ollama/ollama:0.1.34  # Pinned version (check for updates quarterly)
    container_name: ollama
    restart: unless-stopped
    user: "1000:1000"  # Non-root user (match host UID/GID)
    environment:
      - OLLAMA_KEEP_ALIVE=${OLLAMA_KEEP_ALIVE:-24h}
      - TZ=${TZ:-America/New_York}  # Set in .env
    volumes:
      - ollama_data:/root/.ollama  # Model storage
    networks:
      - default  # Internal-only communication
    deploy:
      resources:
        limits:
          cpus: '2'  # Dedicate 2 CPU cores
          memory: 8G   # Limit memory usage
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:11434"]
      interval: 30s
      timeout: 5s
      retries: 3

  anythingllm:
    image: mintplexlabs/anythingllm:1.5.1  # Pinned version
    container_name: anythingllm
    restart: unless-stopped
    depends_on:
      ollama:
        condition: service_healthy
    volumes:
      - anythingllm_storage:/app/server/storage
      - anythingllm_hotdir:/app/collector/hotdir
    environment:
      - OLLAMA_BASE_URL=http://ollama:11434
      - STORAGE_DIR=/app/server/storage
      - TZ=${TZ:-America/New_York}
    networks:
      - default      # Connect to Ollama
      - proxy_network  # Expose via Caddy

  # --- File Services ---
  owncloud:
    image: owncloud/server:10.13.0  # LTS version
    container_name: owncloud_server
    restart: unless-stopped
    depends_on:
      owncloud_mariadb:
        condition: service_healthy
      owncloud_redis:
        condition: service_healthy
    environment:
      - OWNCLOUD_DOMAIN=${OWNCLOUD_DOMAIN}
      - OWNCLOUD_TRUSTED_DOMAINS=${OWNCLOUD_DOMAIN}
      - ADMIN_USERNAME=${OWNCLOUD_ADMIN_USERNAME}
      - ADMIN_PASSWORD=${OWNCLOUD_ADMIN_PASSWORD}
      - OWNCLOUD_DB_HOST=owncloud_mariadb
      - TZ=${TZ:-America/New_York}
    volumes:
      - owncloud_files:/mnt/data
    networks:
      - default
      - proxy_network

  owncloud_mariadb:
    image: mariadb:10.11.6  # Match ownCloud requirements
    container_name: owncloud_mariadb
    restart: unless-stopped
    environment:
      - MYSQL_ROOT_PASSWORD=${MARIADB_ROOT_PASSWORD}
      - MYSQL_PASSWORD=${OWNCLOUD_DB_PASSWORD}
      - TZ=${TZ:-America/New_York}
    volumes:
      - owncloud_mysql_data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-u", "root", "--password=${MARIADB_ROOT_PASSWORD}"]
      interval: 10s

  syncthing:
    image: syncthing/syncthing:1.27.7  # Stable version
    container_name: syncthing
    hostname: homelab-syncthing  # Identify in network
    restart: unless-stopped
    network_mode: host  # Better NAT traversal
    environment:
      - PUID=${SYNCTHING_PUID:-1000}
      - PGID=${SYNCTHING_PGID:-1000}
      - TZ=${TZ:-America/New_York}
    volumes:
      - ./syncthing_config:/var/syncthing/config
      - ./syncthing_data:/var/syncthing/Sync
    deploy:
      resources:
        limits:
          memory: 2G

  # --- Infrastructure Services ---
  caddy:
    image: caddy:2.7.6-alpine  # With Cloudflare DNS support
    container_name: caddy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
      - "443:443/udp"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy_data:/data
      - caddy_config:/config
    environment:
      - CADDY_CLOUDFLARE_API_TOKEN=${ACME_CLOUDFLARE_API_TOKEN}
      - ACME_EMAIL=${ACME_EMAIL}
      - TZ=${TZ:-America/New_York}
    networks:
      - proxy_network
    cap_add:
      - NET_BIND_SERVICE

  ddclient:
    image: linuxserver/ddclient:6.12.0  # Pinned version
    container_name: ddclient
    restart: unless-stopped
    network_mode: host  # Get accurate public IP
    environment:
      - DDCLIENT_PROTOCOL=cloudflare
      - DDCLIENT_LOGIN=${DDCLIENT_CLOUDFLARE_EMAIL}
      - DDCLIENT_PASSWORD=${DDCLIENT_CLOUDFLARE_API_TOKEN}
      - TZ=${TZ:-America/New_York}
    volumes:
      - ddclient_cache:/config

  portainer:
    image: portainer/portainer-ce:2.20.2  # LTS version
    container_name: portainer
    restart: unless-stopped
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro  # Read-only
      - portainer_data:/data
    networks:
      - proxy_network

volumes:
  ollama_data:
  anythingllm_storage:
  anythingllm_hotdir:
  owncloud_files:
  owncloud_mysql_data:
  portainer_data:
  caddy_data:
  caddy_config:
  ddclient_cache:

networks:
  default:  # Implicit bridge network for internal services
    driver: bridge
  proxy_network:  # Explicit network for reverse-proxied services
    driver: bridge
```
