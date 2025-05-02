# Combined Self-Hosted Services with Docker Compose

This repository provides a unified `docker-compose.yml` file to easily deploy and manage a collection of useful self-hosted services using Docker.

✨ **Services Included:**

*   **Ollama:** Run large language models locally.
*   **AnythingLLM:** Private RAG (Retrieval-Augmented Generation) solution using Ollama.
*   **OwnCloud:** File hosting, sharing, and synchronization (includes MariaDB & Redis).
*   **SyncThing:** Continuous, decentralized file synchronization.
*   **Portainer CE:** Easy Docker environment management UI.
*   **Caddy:** Automatic HTTPS reverse proxy.

---

##  Prerequisites

Before you begin, ensure you have the following installed:

1.  **Docker:** [Install Docker](https://docs.docker.com/engine/install/)
2.  **Docker Compose:** (Usually included with Docker Desktop, otherwise [Install Docker Compose](https://docs.docker.com/compose/install/))
3.  **(Optional) NVIDIA Drivers & Container Toolkit:** If you intend to use GPU acceleration for Ollama. [NVIDIA Container Toolkit Installation](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html)
4.  **(Optional but Recommended for Caddy HTTPS):** A domain name pointed to your server's public IP address.

---

## ⚙️ Configuration

**Critical:** You **must** configure the setup before running `docker-compose up`.

1.  **Create `.env` File:**
    Create a file named `.env` in the same directory as `docker-compose.yml`. This file will store your secrets and environment-specific settings. Copy and paste the following template, replacing the placeholder values with your actual configuration:

    ```dotenv
    # === General ===
    # Timezone (optional, used by some containers)
    # TZ=Europe/London

    # === OwnCloud Configuration ===
    # Your external domain for OwnCloud access (e.g., owncloud.yourdomain.com)
    OWNCLOUD_DOMAIN=owncloud.example.com
    # Desired OwnCloud admin username
    OWNCLOUD_ADMIN_USERNAME=admin
    # STRONG password for OwnCloud admin user
    OWNCLOUD_ADMIN_PASSWORD=changeme_owncloud_admin_password
    # STRONG password for the OwnCloud database user (used by OwnCloud service and MariaDB service)
    OWNCLOUD_DB_PASSWORD=changeme_owncloud_db_password
    # STRONG root password for the MariaDB database (for management/healthcheck)
    MARIADB_ROOT_PASSWORD=changeme_mariadb_root_password

    # === SyncThing Configuration ===
    # User ID and Group ID of the user owning the host directories mapped into Syncthing
    # Find using `id -u` and `id -g` on Linux/macOS host
    SYNCTHING_PUID=1000
    SYNCTHING_PGID=1000

    # === Caddy Configuration (Optional - for ACME DNS Challenge) ===
    # Your email address for Let's Encrypt registration
    ACME_EMAIL=your-email@example.com
    # Your Cloudflare API Token (if using Cloudflare DNS challenge) - Scope appropriately!
    # CLOUDFLARE_API_TOKEN=YOUR_CLOUDFLARE_API_TOKEN

    # === Ollama Configuration (Optional) ===
    # How long Ollama should keep models loaded in memory (e.g., 24h, 1h, 5m)
    # OLLAMA_KEEP_ALIVE=24h

    ```
    **Security Note:** Ensure the `.env` file has restrictive permissions and is **never** committed to public Git repositories. Add `.env` to your `.gitignore` file.

2.  **Prepare Host Directories:**
    *   **SyncThing:** Create the directories on your host machine that you intend to map into the SyncThing container. By default, the compose file expects:
        *   `./syncthing_config` (for configuration)
        *   `./syncthing_data/Documents` (example data folder)
        *   Adjust or add volumes in the `syncthing` service definition within `docker-compose.yml` to match your desired host folders.
        *   **Crucially:** Ensure the `SYNCTHING_PUID` and `SYNCTHING_PGID` in your `.env` file match the owner of these host directories for correct permissions.
    *   **(Optional) Caddy Site:** If you plan to serve static files directly with Caddy (using the example in `Caddyfile`), create a `./caddy_site` directory.

3.  **Configure `Caddyfile`:**
    *   Create a file named `Caddyfile` in the same directory.
    *   Use the example below as a starting point.
    *   **Replace ALL placeholders** like `your-email@example.com` and `*.yourdomain.com` with your actual domain(s) and email.
    *   Customize the proxy configurations or add/remove services as needed.

    ```caddyfile
    # /path/to/your/project/Caddyfile
    # See Caddy documentation for full options: https://caddyserver.com/docs/caddyfile

    # --- Global Options ---
    {
        # Email for ACME certificate registration (Reads from ACME_EMAIL env var if set)
        # email {$ACME_EMAIL}

        # Optional: Use Let's Encrypt staging server for testing
        # acme_ca https://acme-staging-v02.api.letsencrypt.org/directory
    }

    # --- Service Definitions ---
    # Replace *.yourdomain.com with your actual domains/subdomains

    # Portainer - Accessible at portainer.yourdomain.com
    portainer.yourdomain.com {
        reverse_proxy portainer:9443
    }

    # OwnCloud - Accessible at owncloud.yourdomain.com
    # IMPORTANT: Domain must match OWNCLOUD_DOMAIN in .env
    owncloud.yourdomain.com {
        header Strict-Transport-Security "max-age=15552000;"
        reverse_proxy owncloud:8080 {
             # Send essential headers for OwnCloud behind proxy
             header_up X-Real-IP {remote_host}
             header_up X-Forwarded-For {remote_host}
             header_up X-Forwarded-Host {host}
             header_up X-Forwarded-Port {server_port}
             header_up X-Forwarded-Proto {scheme}
        }
        # OwnCloud well-known discovery endpoints
        redir /.well-known/carddav /remote.php/dav permanent
        redir /.well-known/caldav /remote.php/dav permanent
        # Optional: Increase max request body size (e.g., 10g)
        # request_body max_size 10g
    }

    # SyncThing UI - Accessible at syncthing.yourdomain.com
    syncthing.yourdomain.com {
        reverse_proxy syncthing:8384
    }

    # AnythingLLM UI - Accessible at anythingllm.yourdomain.com
    anythingllm.yourdomain.com {
        reverse_proxy anythingllm:3001
    }

    # Ollama API (Optional - Expose with EXTREME caution, add authentication!)
    # ollama.yourdomain.com {
    #    # Example: Basic Authentication (generate hash with `caddy hash-password`)
    #    # basicauth /* {
    #    #     your_user JDJhJDE0JEQuOmittedPASSWORDhash...
    #    # }
    #    reverse_proxy ollama:11434
    # }

    # Example: Serve static files from ./caddy_site at static.yourdomain.com
    # static.yourdomain.com {
    #    root * /srv
    #    file_server
    # }
    ```

4.  **(Optional) Enable GPU for Ollama:**
    *   If you have compatible NVIDIA hardware and the NVIDIA Container Toolkit installed, uncomment the `deploy` and related `environment` sections within the `ollama` service in `docker-compose.yml`.

---

## 🚀 Usage

1.  **Start Services:** Navigate to the directory containing `docker-compose.yml` and `.env` in your terminal and run:
    ```bash
    docker-compose up -d
    ```
    The `-d` flag runs the containers in detached mode (in the background).

2.  **Access Services:** Access your services via the domains configured in your `Caddyfile`. Caddy automatically handles HTTPS:
    *   **Portainer:** `https://portainer.yourdomain.com`
    *   **OwnCloud:** `https://owncloud.yourdomain.com` (Log in with the admin credentials set in `.env`)
    *   **SyncThing:** `https://syncthing.yourdomain.com`
    *   **AnythingLLM:** `https://anythingllm.yourdomain.com`
    *   **Ollama API:** (If exposed via Caddy) `https://ollama.yourdomain.com` or internally at `http://<server-ip>:11434`

3.  **View Logs:**
    ```bash
    # View logs for all services (Ctrl+C to stop)
    docker-compose logs -f

    # View logs for a specific service (e.g., caddy)
    docker-compose logs -f caddy
    ```

4.  **Stop Services:**
    ```bash
    # Stop and remove containers, networks, default volumes
    docker-compose down

    # Stop and remove containers, networks, AND removes named volumes (DATA LOSS!)
    # docker-compose down -v
    ```

---

## 💾 Volumes & Data Persistence

*   Named volumes (`ollama_data`, `anythingllm_storage`, `owncloud_files`, etc.) are used to persist data across container restarts. Docker manages these volumes.
*   SyncThing uses bind mounts (`./syncthing_config`, `./syncthing_data/*`) which link directly to directories on your host machine. **Ensure these host directories exist and have correct permissions.**

---

## 🌐 Networking

*   `default_network`: Used for internal communication between services that don't need external exposure (e.g., OwnCloud connecting to its database).
*   `proxy_network`: Used for services that will be accessed via the Caddy reverse proxy. Caddy needs to be on this network to reach the internal ports of other services (like Portainer's `9443` or OwnCloud's `8080`).

---

## Customization

Feel free to modify the `docker-compose.yml` and `Caddyfile` to add/remove services, change ports (if not using Caddy for a service), or adjust resource limits. Remember to back up your volumes before making significant changes.

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
