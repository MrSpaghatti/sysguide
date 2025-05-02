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
# Combined Docker Compose file for various services
# Includes: Ollama, AnythingLLM, OwnCloud (with MariaDB & Redis), SyncThing, Caddy, Portainer

version: '3.8'

services:

  # --- AI Services ---
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    pull_policy: always
    tty: true
    restart: unless-stopped
    # --- GPU Acceleration (Optional - Uncomment/Configure if you have NVIDIA GPU & drivers) ---
    # deploy:
    #   resources:
    #     reservations:
    #       devices:
    #         - driver: nvidia
    #           count: all # Use 'all' or specify a count e.g., 1
    #           capabilities: [gpu]
    # environment:
    #   - NVIDIA_VISIBLE_DEVICES=all
    #   - NVIDIA_DRIVER_CAPABILITIES=compute,utility
    # --- End GPU Acceleration ---
    environment:
      - OLLAMA_KEEP_ALIVE=24h # Keep models loaded for 24 hours
    volumes:
      - ollama_data:/root/.ollama # Ollama models and data persistence
    ports:
      - "11434:11434" # Ollama API Port
    networks:
      - default_network # Internal communication

  anythingllm:
    image: mintplexlabs/anythingllm:latest
    container_name: anythingllm
    ports:
      - "3001:3001" # AnythingLLM Web UI Port
    volumes:
      - anythingllm_storage:/app/server/storage # Persistent storage for AnythingLLM
      - anythingllm_hotdir:/app/collector/hotdir # Directory for automatic file ingestion
    environment:
      # Connect to Ollama using its service name within the Docker network
      OLLAMA_BASE_URL: http://ollama:11434
      STORAGE_DIR: /app/server/storage # Ensure persistence uses the volume
    depends_on:
      - ollama
    restart: unless-stopped
    networks:
      - default_network # Internal communication with Ollama
      - proxy_network # Expose via Caddy

  # --- File Hosting & Sync ---
  owncloud:
    image: owncloud/server:latest # Use a specific version in production if needed, e.g., 10.13
    container_name: owncloud_server
    restart: unless-stopped
    ports:
      # Expose OwnCloud internally on 8080, Caddy will handle external access on 80/443
      # If you NEED direct host access, uncomment and change the host port (e.g., "8081:8080")
      # - "8081:8080"
      - "8080" # Expose port only to other containers (like Caddy) by default
    depends_on:
      - owncloud_mariadb
      - owncloud_redis
    environment:
      # --- IMPORTANT: Configure these OwnCloud variables ---
      - OWNCLOUD_DOMAIN=owncloud.example.com # Replace with your domain, managed by Caddy
      - OWNCLOUD_TRUSTED_DOMAINS=owncloud.example.com, caddy # Trust Caddy as proxy and the external domain
      - ADMIN_USERNAME=admin # Replace with your desired admin username
      - ADMIN_PASSWORD=changeme # Replace with a strong admin password
      # --- Database & Redis Connection (Uses service names) ---
      - OWNCLOUD_DB_TYPE=mysql
      - OWNCLOUD_DB_NAME=owncloud
      - OWNCLOUD_DB_USERNAME=owncloud
      - OWNCLOUD_DB_PASSWORD=owncloud # Should match MARIADB_PASSWORD below
      - OWNCLOUD_DB_HOST=owncloud_mariadb
      - OWNCLOUD_MYSQL_UTF8MB4=true
      - OWNCLOUD_REDIS_ENABLED=true
      - OWNCLOUD_REDIS_HOST=owncloud_redis
      # --- Caddy Proxy Headers (Essential when behind reverse proxy) ---
      - OWNCLOUD_OVERWRITEHOST=owncloud.example.com # Your external domain
      - OWNCLOUD_OVERWRITEPROTOCOL=https # Assuming Caddy handles HTTPS
      - OWNCLOUD_OVERWRITEWEBROOT=/
      - OWNCLOUD_OVERWRITECONDADDR=^172\.([1-3][0-9]|4[0-4])\.0\..*$ # Adjust if your proxy network subnet differs significantly
    healthcheck:
      test: ["CMD", "/usr/bin/healthcheck"]
      interval: 30s
      timeout: 10s
      retries: 5
    volumes:
      - owncloud_files:/mnt/data # User data persistence
    networks:
      - default_network # Internal communication with DB/Redis
      - proxy_network # Expose via Caddy

  owncloud_mariadb:
    image: mariadb:10.11 # ownCloud recommended version
    container_name: owncloud_mariadb
    restart: unless-stopped
    environment:
      - MYSQL_ROOT_PASSWORD=changeme_root # Replace with a strong root password
      - MYSQL_USER=owncloud
      - MYSQL_PASSWORD=owncloud # Should match OWNCLOUD_DB_PASSWORD above
      - MYSQL_DATABASE=owncloud
      - MARIADB_AUTO_UPGRADE=1
    command: ["--max-allowed-packet=128M", "--innodb-log-file-size=64M"]
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-u", "root", "--password=changeme_root"] # Use the root password set above
      interval: 10s
      timeout: 5s
      retries: 5
    volumes:
      - owncloud_mysql_data:/var/lib/mysql # Database persistence
    networks:
      - default_network # Internal communication only

  owncloud_redis:
    image: redis:6 # ownCloud recommended version
    container_name: owncloud_redis
    restart: unless-stopped
    command: ["--databases", "1"]
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    volumes:
      - owncloud_redis_data:/data # Redis data persistence
    networks:
      - default_network # Internal communication only

  syncthing:
    image: syncthing/syncthing:latest
    container_name: syncthing
    hostname: my-syncthing # Optional: Set a hostname for SyncThing UI clarity
    environment:
      - PUID=1000 # IMPORTANT: Set to the user ID that owns the host data folders
      - PGID=1000 # IMPORTANT: Set to the group ID that owns the host data folders
    volumes:
      # --- IMPORTANT: Configure these host paths ---
      # Map Syncthing config directory to host
      - ./syncthing_config:/var/syncthing/config
      # Map Syncthing data directory(ies) - add more as needed
      # Example: Map a 'Documents' folder on the host to /sync/Documents inside the container
      - ./syncthing_data/Documents:/var/syncthing/Documents
      # Example: Map a 'Photos' folder
      # - ./syncthing_data/Photos:/var/syncthing/Photos
    ports:
      # Web UI Port (Exposed via Caddy, but can be exposed directly if needed)
      # - "8384:8384"
      # Syncing Ports (Essential)
      - "22000:22000/tcp" # TCP file transfers
      - "22000:22000/udp" # QUIC file transfers
      - "21027:21027/udp" # Local discovery broadcasts
    restart: unless-stopped
    healthcheck:
      # Use service name 'localhost' or '127.0.0.1' inside the container
      test: curl -fkLsS -m 2 http://127.0.0.1:8384/rest/noauth/health | grep -q '"status": "OK"' || exit 1
      interval: 1m
      timeout: 10s
      retries: 3
    networks:
      - proxy_network # Expose UI via Caddy
      # Needs default network if it needs to interact with other non-proxied services,
      # but primarily needs host access via volumes and network access for sync ports.

  # --- Management & Proxy ---
  portainer:
    image: portainer/portainer-ce:lts # Use Long Term Support version
    container_name: portainer
    restart: unless-stopped
    ports:
      # Portainer Agent port (if needed, rarely used in single-node setup)
      # - "8000:8000"
      # Portainer Web UI Port (Exposed via Caddy, but can be exposed directly if needed)
      - "9443" # Expose port only to other containers (like Caddy) by default
      # If you NEED direct host access, uncomment and change the host port (e.g., "9444:9443")
      # - "9444:9443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock # Allow Portainer to manage Docker
      - portainer_data:/data # Portainer configuration persistence
    networks:
      - proxy_network # Expose via Caddy

  caddy:
    # Use a version with cloudflare plugin if you need DNS challenge for wildcard certs etc.
    # image: ghcr.io/caddybuilds/caddy-cloudflare:latest
    # Otherwise, use the official image:
    image: caddy:latest
    container_name: caddy
    restart: unless-stopped
    # Needed for Caddy to bind to low ports 80/443 without running as root (if using non-root user internally)
    # Also potentially needed for some network operations depending on Caddyfile features.
    # cap_add:
    #   - NET_ADMIN
    #   - NET_BIND_SERVICE
    ports:
      - "80:80"   # HTTP Port
      - "443:443" # HTTPS Port
      - "443:443/udp" # HTTP/3 QUIC Port
    volumes:
      # --- IMPORTANT: Create these files/folders next to docker-compose.yml ---
      - ./Caddyfile:/etc/caddy/Caddyfile # Mount your Caddyfile configuration
      # Optional: Mount a directory for serving static files if needed
      # - ./caddy_site:/srv
      - caddy_data:/data # Persists Caddy's state including issued certificates
      - caddy_config:/config # Persists Caddy's configuration backups etc.
    environment:
      # --- IMPORTANT: Configure these for ACME DNS Challenge (e.g., Cloudflare) ---
      # Replace with your actual Cloudflare API Token (scoped appropriately)
      # - CLOUDFLARE_API_TOKEN=YOUR_CLOUDFLARE_API_TOKEN
      # Replace with the email address associated with your domain/ACME registration
      # - ACME_EMAIL=your-email@example.com
      # Optional: Staging environment for testing ACME without hitting rate limits
      # - ACME_CA=https://acme-staging-v02.api.letsencrypt.org/directory
      # --- Caddy config ---
      - CADDY_CONFIG=/etc/caddy/Caddyfile # Tell Caddy where the config file is
    networks:
      - proxy_network # Caddy lives on this network to proxy other services

# --- Volumes Definition ---
# Define all named volumes used by the services
volumes:
  ollama_data:
    driver: local
  anythingllm_storage:
    driver: local
  anythingllm_hotdir:
    driver: local
  owncloud_files:
    driver: local
  owncloud_mysql_data:
    driver: local
  owncloud_redis_data:
    driver: local
  # Syncthing volumes are bind mounts defined directly in the service above
  # Make sure ./syncthing_config and ./syncthing_data/* exist on the host
  portainer_data:
    driver: local
  caddy_data:
    driver: local # Persists certificates and other Caddy state
  caddy_config:
    driver: local # Persists Caddy configuration

# --- Networks Definition ---
# Define networks used for communication
networks:
  default_network:
    driver: bridge # Default network for internal service-to-service communication (e.g., web app to DB)
  proxy_network:
    driver: bridge # Network for services that Caddy will reverse proxy
```

```yaml
# Combined Docker Compose for Self-Hosted Services
# Manages: Ollama, AnythingLLM, OwnCloud, SyncThing, Portainer, Caddy
# See README.md for configuration and usage instructions.

version: '3.8'

services:

  # --- AI Services ---
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    pull_policy: always
    tty: true
    restart: unless-stopped
    # --- Optional: GPU Acceleration (Requires NVIDIA Container Toolkit) ---
    # deploy:
    #   resources:
    #     reservations:
    #       devices:
    #         - driver: nvidia
    #           count: all
    #           capabilities: [gpu]
    # environment:
    #   - NVIDIA_VISIBLE_DEVICES=all
    #   - NVIDIA_DRIVER_CAPABILITIES=compute,utility
    # --- End Optional: GPU Acceleration ---
    environment:
      # Keep models loaded, adjust as needed
      - OLLAMA_KEEP_ALIVE=${OLLAMA_KEEP_ALIVE:-24h}
    volumes:
      # Ollama model data persistence
      - ollama_data:/root/.ollama
    ports:
      # Ollama API port (typically accessed internally or via Caddy)
      - "11434:11434"
    networks:
      - default_network # Internal communication

  anythingllm:
    image: mintplexlabs/anythingllm:latest
    container_name: anythingllm
    restart: unless-stopped
    ports:
      # AnythingLLM Web UI port (exposed via Caddy)
      - "3001:3001"
    volumes:
      # AnythingLLM persistent storage
      - anythingllm_storage:/app/server/storage
      # AnythingLLM hot directory for file ingestion
      - anythingllm_hotdir:/app/collector/hotdir
    environment:
      # Connect to Ollama service within Docker network
      OLLAMA_BASE_URL: http://ollama:11434
      # Ensure storage uses the mounted volume
      STORAGE_DIR: /app/server/storage
    depends_on:
      - ollama
    networks:
      - default_network # Internal communication with Ollama
      - proxy_network   # Expose UI via Caddy

  # --- File Hosting & Sync ---
  owncloud:
    image: owncloud/server:latest # Use a specific version tag in production
    container_name: owncloud_server
    restart: unless-stopped
    ports:
      # Internal port 8080, exposed via Caddy
      - "8080"
    depends_on:
      - owncloud_mariadb
      - owncloud_redis
    environment:
      # --- Required OwnCloud Configuration (Set in .env file) ---
      - OWNCLOUD_DOMAIN=${OWNCLOUD_DOMAIN}
      - OWNCLOUD_TRUSTED_DOMAINS=${OWNCLOUD_DOMAIN},caddy # Trust Caddy proxy and external domain
      - ADMIN_USERNAME=${OWNCLOUD_ADMIN_USERNAME}
      - ADMIN_PASSWORD=${OWNCLOUD_ADMIN_PASSWORD}
      - OWNCLOUD_DB_PASSWORD=${OWNCLOUD_DB_PASSWORD} # Match DB password
      # --- Database & Redis Connection ---
      - OWNCLOUD_DB_TYPE=mysql
      - OWNCLOUD_DB_NAME=owncloud
      - OWNCLOUD_DB_USERNAME=owncloud
      - OWNCLOUD_DB_HOST=owncloud_mariadb
      - OWNCLOUD_MYSQL_UTF8MB4=true
      - OWNCLOUD_REDIS_ENABLED=true
      - OWNCLOUD_REDIS_HOST=owncloud_redis
      # --- Reverse Proxy Configuration ---
      - OWNCLOUD_OVERWRITEHOST=${OWNCLOUD_DOMAIN}
      - OWNCLOUD_OVERWRITEPROTOCOL=https
      - OWNCLOUD_OVERWRITEWEBROOT=/
      # Adjust regex if your docker network subnet differs significantly from 172.16.0.0/12
      - OWNCLOUD_OVERWRITECONDADDR=^172\.([1-3][0-9]|4[0-4])\.0\..*$
    healthcheck:
      test: ["CMD", "/usr/bin/healthcheck"]
      interval: 30s
      timeout: 10s
      retries: 5
    volumes:
      # User data persistence
      - owncloud_files:/mnt/data
    networks:
      - default_network # Internal communication with DB/Redis
      - proxy_network   # Expose UI via Caddy

  owncloud_mariadb:
    image: mariadb:10.11 # Recommended version for OwnCloud
    container_name: owncloud_mariadb
    restart: unless-stopped
    environment:
      # --- Required MariaDB Configuration (Set in .env file) ---
      - MYSQL_ROOT_PASSWORD=${MARIADB_ROOT_PASSWORD}
      - MYSQL_PASSWORD=${OWNCLOUD_DB_PASSWORD} # Match OwnCloud DB password
      # --- Standard DB Setup ---
      - MYSQL_USER=owncloud
      - MYSQL_DATABASE=owncloud
      - MARIADB_AUTO_UPGRADE=1
    command: ["--max-allowed-packet=128M", "--innodb-log-file-size=64M"]
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-u", "root", "--password=${MARIADB_ROOT_PASSWORD}"]
      interval: 10s
      timeout: 5s
      retries: 5
    volumes:
      # Database data persistence
      - owncloud_mysql_data:/var/lib/mysql
    networks:
      - default_network # Internal communication only

  owncloud_redis:
    image: redis:6 # Recommended version for OwnCloud
    container_name: owncloud_redis
    restart: unless-stopped
    command: ["--databases", "1"]
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    volumes:
      # Redis data persistence
      - owncloud_redis_data:/data
    networks:
      - default_network # Internal communication only

  syncthing:
    image: syncthing/syncthing:latest
    container_name: syncthing
    hostname: my-syncthing # Hostname inside the container network
    restart: unless-stopped
    environment:
      # Set to user/group ID owning host data directories
      - PUID=${SYNCTHING_PUID:-1000}
      - PGID=${SYNCTHING_PGID:-1000}
    volumes:
      # --- Host Path Mounts (Configure paths on host machine) ---
      # Syncthing config directory
      - ./syncthing_config:/var/syncthing/config
      # Example data sync directory (map desired host folders)
      - ./syncthing_data/Documents:/var/syncthing/Documents
      # Add more volumes for other directories to sync
      # - ./syncthing_data/Photos:/var/syncthing/Photos
    ports:
      # Web UI Port (exposed via Caddy)
      # - "8384:8384" # Uncomment for direct access if needed
      # Required Syncing Ports
      - "22000:22000/tcp" # TCP file transfers
      - "22000:22000/udp" # QUIC file transfers
      - "21027:21027/udp" # Local discovery
    healthcheck:
      test: curl -fkLsS -m 2 http://127.0.0.1:8384/rest/noauth/health | grep -q '"status": "OK"' || exit 1
      interval: 1m
      timeout: 10s
      retries: 3
    networks:
      # Needs host network access implicitly via ports for sync
      - proxy_network # Expose UI via Caddy

  # --- Management & Proxy ---
  portainer:
    image: portainer/portainer-ce:lts # Long Term Support version
    container_name: portainer
    restart: unless-stopped
    ports:
      # Portainer Agent port (rarely needed in single-node setup)
      # - "8000:8000"
      # Portainer Web UI port (exposed via Caddy)
      - "9443" # Internal port
      # - "9444:9443" # Uncomment and change host port for direct access
    volumes:
      # Mount Docker socket to allow Portainer control
      - /var/run/docker.sock:/var/run/docker.sock
      # Portainer configuration persistence
      - portainer_data:/data
    networks:
      - proxy_network # Expose UI via Caddy

  caddy:
    # Use official image or one with DNS plugins (e.g., Cloudflare)
    # image: ghcr.io/caddybuilds/caddy-cloudflare:latest
    image: caddy:latest
    container_name: caddy
    restart: unless-stopped
    # Required for binding low ports (80, 443)
    cap_add:
      - NET_BIND_SERVICE
    ports:
      - "80:80"       # HTTP
      - "443:443"     # HTTPS
      - "443:443/udp" # HTTP/3 (QUIC)
    volumes:
      # Mount Caddyfile configuration
      - ./Caddyfile:/etc/caddy/Caddyfile
      # Optional: Mount site data for static file serving
      # - ./caddy_site:/srv
      # Persists Caddy state (certs, etc.)
      - caddy_data:/data
      # Persists Caddy config backups
      - caddy_config:/config
    environment:
      # --- Optional: ACME DNS Challenge (Set in .env file if used) ---
      # - CLOUDFLARE_API_TOKEN=${CLOUDFLARE_API_TOKEN}
      # - ACME_EMAIL=${ACME_EMAIL}
      # --- Caddy Config Location ---
      - CADDY_CONFIG=/etc/caddy/Caddyfile
    networks:
      - proxy_network # Connects to services it proxies

# --- Volumes Definition ---
# Defines persistent storage areas used by services
volumes:
  ollama_data:
  anythingllm_storage:
  anythingllm_hotdir:
  owncloud_files:
  owncloud_mysql_data:
  owncloud_redis_data:
  portainer_data:
  caddy_data:
  caddy_config:
  # Note: syncthing volumes are bind mounts defined directly in the service

# --- Networks Definition ---
# Defines communication channels between services
networks:
  default_network: # Internal communication network
    driver: bridge
  proxy_network:   # Network for services exposed via Caddy
    driver: bridge
```
