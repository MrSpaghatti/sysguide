# Docker Compose Collection

This document is intended to be a collection of individual, officially sourced (where possible) Docker Compose files or snippets for various useful tools and services. Each entry should include:

*   A brief description of the tool/service.
*   A link to the official documentation or source of the Docker Compose example.
*   The `docker-compose.yml` content.
*   Key setup notes or essential environment variables.

---

## Portainer CE

**Description:** Portainer Community Edition (CE) is a lightweight service delivery platform for containerized applications that can be used to manage Docker, Docker Swarm, Kubernetes, and ACI environments. It provides a simple and intuitive web UI.

**Source:** [Official Portainer CE Docker Installation Documentation](https://docs.portainer.io/start/install-ce/server/docker/linux)

**Docker Compose (`docker-compose.yml`):**
```yaml
version: '3.8'

services:
  portainer:
    image: portainer/portainer-ce:lts # Using Long Term Support tag, can be 'latest'
    container_name: portainer
    ports:
      - "8000:8000" # Optional: For Edge Agent tunnel server
      - "9443:9443" # Main UI port (HTTPS)
      # - "9000:9000" # Optional: Legacy HTTP port
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
    restart: always

volumes:
  portainer_data:
```

**Key Setup Notes:**
*   **Access:** The Portainer Server UI will be available at `https://<your-host-ip>:9443`.
*   **Initial Setup:** On first access, you will be prompted to create an admin user and password.
*   **Docker Socket:** The line `-v /var/run/docker.sock:/var/run/docker.sock` is crucial for Portainer to manage your Docker environment. Ensure this path is correct for your system (it's standard for Linux).
*   **Persistent Data:** Portainer's data (configurations, user settings, etc.) is stored in the named Docker volume `portainer_data`.
*   **Ports:**
    *   `9443` is the default HTTPS port for the UI.
    *   `8000` is used for the optional Edge Agent tunnel server. If you don't plan to use Edge Agents, you can omit this port mapping.
    *   The documentation mentions a legacy `9000` HTTP port, which can be added if needed but HTTPS on `9443` is recommended.
*   **Image Tag:** The example uses `portainer/portainer-ce:lts` for the Long Term Support version. You can use `portainer/portainer-ce:latest` to get the newest features, but `lts` is generally more stable for production-like environments.
*   **SELinux:** If SELinux is enabled on your host, you might need to run the container with the `--privileged` flag or configure appropriate SELinux policies. The official documentation usually provides guidance on this.

---

## Caddy

**Description:** Caddy is a powerful, enterprise-ready, open-source web server with automatic HTTPS. It's known for its ease of use and modern features.

**Source:** [Caddy Docker Hub Page](https://hub.docker.com/_/caddy) (Primary source for image info) and [Caddy Docker Documentation](https://caddyserver.com/docs/running#docker)

**Docker Compose (`docker-compose.yml`):**
```yaml
version: '3.8'

services:
  caddy:
    image: caddy:latest # Or a specific version e.g., caddy:2.7-alpine
    container_name: caddy
    ports:
      - "80:80"    # For HTTP
      - "443:443"  # For HTTPS
      - "443:443/udp" # For HTTP/3
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile   # Mount your Caddyfile
      - caddy_data:/data                   # For SSL certificates and other Caddy data
      - caddy_config:/config               # For Caddy's operational configuration
    restart: unless-stopped
    # Optional: Add network_mode or networks if integrating with other services
    # networks:
    #   - my_proxy_network

volumes:
  caddy_data:
  caddy_config:

# Optional: Define a shared network if Caddy will proxy other containers
# networks:
#   my_proxy_network:
#     driver: bridge
```

**Key Setup Notes:**
*   **Caddyfile:**
    *   You **must** create a `Caddyfile` in the same directory as your `docker-compose.yml` (or adjust the volume path). This file defines how Caddy serves your sites.
    *   **Example `Caddyfile`:**
        ```caddy
        yourdomain.com {
            respond "Hello, world!"
        }

        # To reverse proxy another Docker container on the same Docker network:
        # (Assuming the other service is named 'myservice' and is on 'my_proxy_network')
        # service.yourdomain.com {
        #     reverse_proxy myservice:service_port
        # }
        ```
*   **Persistent Data:**
    *   `caddy_data` volume: Stores SSL/TLS certificates obtained by Caddy. Crucial for persistence so certs aren't re-fetched on every restart.
    *   `caddy_config` volume: Stores Caddy's internal configuration and state.
*   **Ports:**
    *   `80:80` and `443:443` are standard for web traffic (HTTP and HTTPS). Caddy will automatically handle HTTP to HTTPS redirection.
    *   `443:443/udp` is for enabling HTTP/3.
*   **Image Tag:**
    *   `caddy:latest` will use the latest stable version.
    *   For production, it's often recommended to pin to a specific version tag (e.g., `caddy:2.7.6` or `caddy:2.7-alpine` for a smaller image).
*   **Networking:**
    *   If Caddy is intended to reverse proxy other Docker containers, they should all be on the same Docker network. You can define a network in your compose file (like `my_proxy_network` in the commented-out example) and attach Caddy and your other services to it.
*   **Automatic HTTPS:** Caddy automatically provisions and renews SSL certificates from Let's Encrypt (and other CAs) for any site defined with a domain name in your `Caddyfile`. Ensure your domain's DNS A/AAAA records point to the server running Caddy.
*   **Custom Caddy Builds (with plugins):** If you need Caddy plugins (e.g., for specific DNS providers for the DNS challenge), you'll need to use a custom Caddy image or build your own. The official `caddy` image is vanilla. For example, for Cloudflare DNS challenge: `image: ghcr.io/caddy-dns/cloudflare`.

---

## ddclient

**Description:** ddclient is a Perl client used to update dynamic DNS entries for accounts on various dynamic DNS service providers. It can fetch your WAN IP address in several ways and supports many popular services. This example is based on the LinuxServer.io image.

**Source:** [LinuxServer.io ddclient Documentation](https://docs.linuxserver.io/images/docker-ddclient) and [ddclient Official Website](https://ddclient.net/)

**Docker Compose (`docker-compose.yml`):**
```yaml
version: '3.8'

services:
  ddclient:
    image: lscr.io/linuxserver/ddclient:latest
    container_name: ddclient
    environment:
      - PUID=1000 # Your user ID
      - PGID=1000 # Your group ID
      - TZ=Etc/UTC # Your timezone, e.g., America/New_York
    volumes:
      - ./ddclient_config:/config # Path to store ddclient.conf and cache
    restart: unless-stopped
    # For ddclient to accurately detect the public IP, host networking can be beneficial,
    # especially if it's running on the same machine as your router or modem.
    # However, it can often work without it by using web-based IP checkers.
    # network_mode: host # Uncomment if needed, be aware of port conflicts.
```

**Key Setup Notes:**
*   **Configuration (`ddclient.conf`):**
    *   You **must** create a `ddclient.conf` file inside the host directory mapped to `/config` (e.g., `./ddclient_config/ddclient.conf`).
    *   This file contains the configuration for your dynamic DNS provider(s).
    *   The LinuxServer.io image will automatically restart `ddclient` if it detects changes to this file.
    *   **Example for Cloudflare (`./ddclient_config/ddclient.conf`):**
        ```ini
        # ddclient.conf for Cloudflare
        daemon=300                            # Check every 300 seconds (5 minutes)
        syslog=yes                            # Log to syslog
        #mail=root                            # Mail all msgs to root
        #mail-failure=root                    # Mail failed updates to root
        pid=/var/run/ddclient.pid             # Record PID in file.
        ssl=yes                               # Use SSL/TLS.
        use=web, web=checkip.dyndns.org       # Get IP from web, using dyndns' checker
        #use=if, if=eth0                      # Get IP from interface eth0 (replace if needed)

        # Cloudflare specific configuration
        protocol=cloudflare
        login=your-cloudflare-email@example.com
        password=YOUR_CLOUDFLARE_GLOBAL_API_KEY_OR_API_TOKEN # Use a scoped API Token if possible
        zone=yourdomain.com
        records=subdomain1,subdomain2,@       # Comma separated hostnames to update. '@' for root domain.
        # proxied=true                        # Optional: Set to true to enable Cloudflare proxy (orange cloud)
        ```
*   **Cloudflare API Token vs. Global API Key:**
    *   It is **highly recommended** to use a Cloudflare API Token with specific permissions (`Zone:DNS:Edit` for the relevant zone) instead of your Global API Key for better security.
    *   If using an API Token, the `login` field in `ddclient.conf` is typically your Cloudflare account email, and `password` is the API Token itself. Some configurations might vary, so check ddclient's official documentation for Cloudflare if you encounter issues.
*   **PUID/PGID:** Set `PUID` and `PGID` to the user and group ID that should own the configuration files on the host. Use `id $(whoami)` to find your current user's IDs.
*   **Timezone (TZ):** Set your correct [timezone](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones).
*   **Volume Mapping:** The `./ddclient_config:/config` line maps a directory from your host to the container for persistent configuration and cache. Create this directory on your host.
*   **Network Mode:**
    *   The example compose file comments out `network_mode: host`. `ddclient` often needs to determine your public IP. It can do this via web services (like `use=web, web=checkip.dyndns.org`).
    *   If it has trouble, or if you prefer it to use the host's network directly (e.g., if it's on your router or a machine with a direct public IP), you can uncomment `network_mode: host`. Be mindful this gives the container more network privileges and direct access to host ports.
*   **Official ddclient Documentation:** For detailed configuration options for various providers and advanced setups, refer to the [official ddclient documentation](https://ddclient.net/protocols.html).

---

## Ollama

**Description:** Ollama is a tool for running large language models (LLMs) locally. It provides a simple API and command-line interface to download, manage, and run models like Llama 2, Code Llama, and others.

**Source:** [Ollama Docker Hub](https://hub.docker.com/r/ollama/ollama) and [Ollama GitHub Repository](https://github.com/ollama/ollama) (often contains Docker examples or links to them).

**Docker Compose (`docker-compose.yml`):**
```yaml
version: '3.8'

services:
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    ports:
      - "11434:11434" # Ollama API port
    volumes:
      - ollama_data:/root/.ollama # Persistent storage for models
    restart: unless-stopped
    # --- Optional: GPU Acceleration (NVIDIA) ---
    # Ensure you have the NVIDIA Docker Toolkit installed on your host.
    # Then, uncomment the following 'deploy' section.
    # deploy:
    #   resources:
    #     reservations:
    #       devices:
    #         - driver: nvidia
    #           count: all # Or specify specific GPU IDs e.g., "device=0,1"
    #           capabilities: [gpu]
    # --- End Optional: GPU Acceleration ---

volumes:
  ollama_data:
```

**Key Setup Notes:**
*   **API Port:** Ollama serves its API on port `11434`. This is mapped to the host.
*   **Model Storage:** Models downloaded by Ollama are stored in the `/root/.ollama` directory within the container. This is mapped to a named Docker volume `ollama_data` to ensure models persist across container restarts.
*   **Running Models:** After starting the container, you can interact with Ollama using its CLI through `docker exec` or via its API:
    *   **CLI Example (pull Llama 2):** `docker exec -it ollama ollama pull llama2`
    *   **CLI Example (run Llama 2):** `docker exec -it ollama ollama run llama2`
    *   **API Endpoint:** `http://localhost:11434`
*   **GPU Acceleration (NVIDIA):**
    *   To use an NVIDIA GPU, you must have the [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html) installed on your Docker host.
    *   Uncomment the `deploy` section in the `docker-compose.yml`.
    *   The `count: all` will attempt to make all available GPUs accessible to Ollama. You can specify particular GPUs if needed.
    *   Verify GPU access within the container by running a model and observing logs or using `nvidia-smi` if it were included in the base image (Ollama's official image is minimal, so direct `nvidia-smi` inside might not work, but model loading logs should indicate GPU usage).
*   **Other GPU Vendors (AMD, Intel):** Refer to the official Ollama documentation for the latest guidance on enabling acceleration for other GPU types, as this often requires different Docker image variants or host configurations.
*   **Resource Usage:** LLMs can be very resource-intensive (CPU, RAM, GPU VRAM). Monitor your system's resources. You might want to add resource limits to the service definition in `docker-compose.yml` if running on a shared system:
    ```yaml
    # Example resource limits (adjust as needed)
    # deploy:
    #   resources:
    #     limits:
    #       cpus: '4.0'  # Limit to 4 CPU cores
    #       memory: 16G # Limit to 16GB RAM
    #     reservations: # if not using GPU reservations above
    #       cpus: '2.0'
    #       memory: 8G
    ```

---

## AnythingLLM

**Description:** AnythingLLM is a full-stack application for creating private ChatGPT-like experiences. It allows you to chat with your documents, use AI agents, and connect to various LLMs (like Ollama) and vector databases.

**Source:** [AnythingLLM GitHub Repository](https://github.com/Mintplex-Labs/anything-llm) (The official `docker-compose.yml` is for building from source, this example uses their pre-built Docker Hub image.)

**Docker Compose (`docker-compose.yml`):**
```yaml
version: '3.8'

services:
  anythingllm:
    image: mintplexlabs/anythingllm:latest
    container_name: anythingllm
    ports:
      - "3001:3001" # Frontend and API port
    volumes:
      # Persists application data, vector stores, and configurations
      - ./anythingllm_storage:/app/server/storage
      # Directory for documents to be automatically processed
      - ./anythingllm_hotdir:/app/collector/hotdir
      # Optional: If you need to inspect collector outputs
      # - ./anythingllm_outputs:/app/collector/outputs
    environment:
      # This must match the internal path of the storage volume
      - STORAGE_DIR=/app/server/storage
      # Example: If Ollama is running in another container named 'ollama'
      # on the same Docker network, uncomment and set this:
      # - OLLAMA_BASE_URL=http://ollama:11434
      # If Ollama is on the host machine and Docker can resolve host.docker.internal:
      # - OLLAMA_BASE_URL=http://host.docker.internal:11434
    restart: unless-stopped
    # --- Optional: Define a network to connect with Ollama ---
    # networks:
    #   - llm_network

# Optional: Define the network if connecting to other services like Ollama
# networks:
#   llm_network:
#     driver: bridge

# Note: The official docker-compose.yml from the repository includes a build context
# and an .env file mapping. This example is simplified for pre-built images.
# You might need an .env file for more advanced configurations.
# Example .env content (place in the same directory as docker-compose.yml):
#
# OLLAMA_BASE_URL=http://ollama:11434  # If Ollama is a service on the same Docker network
# JWT_SECRET=           # Generate a strong random string (e.g., openssl rand -hex 32)
#                          # Required for multi-user mode if you enable it.
# ANYTHING_LLM_PASSWORD= # Optional: Set a master password for the instance.
```

**Key Setup Notes:**
*   **Image:** Uses the pre-built image `mintplexlabs/anythingllm:latest` from Docker Hub. You can pin to a specific version tag for stability.
*   **Ports:** Exposes port `3001` for the web UI and API.
*   **Volumes:**
    *   `./anythingllm_storage:/app/server/storage`: **Crucial.** This maps a host directory (create `./anythingllm_storage` or choose your path) to the container's storage location for all persistent data, including vector embeddings, conversation history, and system settings.
    *   `./anythingllm_hotdir:/app/collector/hotdir`: Maps a host directory for document ingestion. Files placed here will be automatically processed by AnythingLLM.
*   **Environment Variables:**
    *   `STORAGE_DIR=/app/server/storage`: **Required.** Tells AnythingLLM where its storage path inside the container is. This must match the target of your main storage volume.
    *   `OLLAMA_BASE_URL`: **Important for connecting to Ollama.**
        *   If Ollama is running as another Docker container (e.g., named `ollama`) on the same Docker network (e.g., `llm_network`), set this to `http://ollama:11434`. Make sure both services are attached to this network (uncomment the `networks` sections in the YAML).
        *   If Ollama is running directly on your host machine, you might use `http://host.docker.internal:11434` (Docker Desktop, some Linux setups) or the host's actual IP address on the Docker bridge network.
    *   `.env` **File (Optional but Recommended):** For more complex setups or to keep secrets out of the compose file, you can use an `.env` file in the same directory as your `docker-compose.yml`. The compose example has a commented-out `env_file` line. If used, variables like `OLLAMA_BASE_URL`, `JWT_SECRET` (for multi-user mode), and `ANYTHING_LLM_PASSWORD` can be defined there.
*   **Networking:**
    *   If running Ollama (or other LLMs/VectorDBs) as separate Docker containers, ensure they are on the same Docker network as AnythingLLM for easy communication using service names. The example includes commented-out sections for defining and using a network named `llm_network`.
*   **Initial Setup:** After starting, access the UI at `http://<your-host-ip>:3001`. You'll be guided through setting up your LLM provider (e.g., connecting to Ollama by providing its base URL), embedding model, and vector database.
*   **User Management:** The Docker version supports multi-user mode. This typically requires setting a `JWT_SECRET` environment variable.

---

## ownCloud Infinite Scale

**Description:** ownCloud Infinite Scale (oCIS) is a modern file-sync and share platform. This example sets up oCIS using its simple Docker deployment, which runs without a traditional database like MariaDB or PostgreSQL, relying on its internal store by default for simpler setups. For production, external storage and user backends are typically configured.

**Source:** [ownCloud Infinite Scale Docker Deployment](https://doc.owncloud.com/ocis/next/deployment/docker/docker-simple.html) and [Docker Hub `owncloud/ocis`](https://hub.docker.com/r/owncloud/ocis)

**Docker Compose (`docker-compose.yml`):**
```yaml
version: '3.8'

services:
  owncloud-ocis:
    image: owncloud/ocis:latest # Or a specific version tag
    container_name: owncloud_ocis
    ports:
      - "9200:9200" # Main oCIS port
    volumes:
      - ocis_data:/var/lib/ocis # Persistent data for oCIS
    environment:
      - OCIS_URL=https://your.domain.com # Replace with your public domain
      - OCIS_ADMIN_PASSWORD=yoursecureadminpassword # Change this!
      # - OCIS_INSECURE=true # Uncomment for HTTP only if not using a reverse proxy with HTTPS
    restart: unless-stopped

volumes:
  ocis_data:
```

**Key Setup Notes:**
*   **Image:** Uses `owncloud/ocis:latest`. Pin to a specific version for production.
*   **oCIS vs. ownCloud Server (10.x):** This example is for **ownCloud Infinite Scale (oCIS)**, which is the newer generation platform. The older `owncloud/server` (version 10.x) has a different setup, typically requiring an external SQL database (like MariaDB/PostgreSQL) and often Redis. If you need ownCloud 10.x, refer to its specific documentation.
*   **Port:** Exposes port `9200` for the oCIS instance.
*   **Volume:** `ocis_data:/var/lib/ocis` stores all oCIS data, including user files if using the default internal storage.
*   **Environment Variables:**
    *   `OCIS_URL`: **Crucial.** Set this to the public URL where your oCIS instance will be accessible (e.g., `https://cloud.example.com`). This is used for internal routing and link generation.
    *   `OCIS_ADMIN_PASSWORD`: **Required.** Set a strong password for the initial admin user (default username is often `admin` or `ocis`).
    *   `OCIS_INSECURE=true`: **For testing/HTTP only.** If you are running oCIS without a reverse proxy providing HTTPS, you might need to set this to `true`. For production, always use HTTPS, typically handled by a reverse proxy like Caddy or Nginx. If using a reverse proxy, ensure it correctly passes headers like `X-Forwarded-Proto`.
*   **Initial Login:** Access oCIS via the `OCIS_URL` you configured. Log in with the admin user (often `admin`) and the `OCIS_ADMIN_PASSWORD`.
*   **Reverse Proxy:** For production, it's highly recommended to run oCIS behind a reverse proxy (like Caddy or Nginx) that handles SSL/TLS termination (HTTPS). The reverse proxy would forward traffic to `http://localhost:9200` (or whatever host/port oCIS is running on from the proxy's perspective).
*   **Data Storage & User Backend:** The simple Docker setup uses an internal data store. For more scalable or production deployments, oCIS can be configured to use external S3-compatible storage, NFS, local POSIX filesystems with more control, and external user backends (LDAP, OpenID Connect). These are advanced configurations beyond this basic compose example. Refer to the [oCIS documentation](https://doc.owncloud.com/ocis/next/) for details.

---

## Syncthing

**Description:** Syncthing is a continuous file synchronization program. It synchronizes files between two or more computers in real time, safely protected from prying eyes. Your data is your data alone and you deserve to choose where it is stored, if it is shared with some third party, and how it's transmitted over the Internet. This example uses the LinuxServer.io image.

**Source:** [LinuxServer.io Syncthing Documentation](https://docs.linuxserver.io/images/docker-syncthing) and [Syncthing Official Website](https://syncthing.net/)

**Docker Compose (`docker-compose.yml`):**
```yaml
version: '3.8'

services:
  syncthing:
    image: lscr.io/linuxserver/syncthing:latest
    container_name: syncthing
    hostname: syncthing # Optional: for easier identification in Syncthing network
    environment:
      - PUID=1000 # Your user ID
      - PGID=1000 # Your group ID
      - TZ=Etc/UTC # Your timezone, e.g., America/New_York
    volumes:
      - ./syncthing_config:/config   # For Syncthing's configuration files
      - ./syncthing_data1:/data1     # Example data directory 1 to sync
      - ./syncthing_data2:/data2     # Example data directory 2 to sync
      # Add more volumes as needed for other directories you want to sync
    ports:
      - "8384:8384"    # Web UI
      - "22000:22000/tcp" # TCP file transfers & protocol messages
      - "22000:22000/udp" # QUIC file transfers & protocol messages
      - "21027:21027/udp" # For local discovery
    restart: unless-stopped
    # For improved discovery, especially across different networks or with NAT,
    # host networking can be considered, but it increases security exposure.
    # network_mode: host
```

**Key Setup Notes:**
*   **PUID/PGID:** Set to the user/group IDs that should own the configuration and data files on the host. Use `id $(whoami)` to find your current user's IDs. This is important for file permissions.
*   **Timezone (TZ):** Set your correct [timezone](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones).
*   **Volumes:**
    *   `./syncthing_config:/config`: **Crucial.** Maps a host directory (e.g., create `./syncthing_config`) to store Syncthing's configuration, device keys, and database.
    *   `/data1`, `/data2`, etc.: These are examples. You should map host directories that you want Syncthing to manage and synchronize. For example, if you want to sync your `~/Documents` folder, you might use `- ./my_documents:/sync/documents` inside the container (and then add `/sync/documents` as a folder within the Syncthing UI). The paths inside the container (e.g., `/data1`) are what you'll configure in the Syncthing Web UI.
*   **Ports:**
    *   `8384:8384`: The port for Syncthing's Web UI. Access it at `http://<your-host-ip>:8384`.
    *   `22000:22000/tcp` and `22000:22000/udp`: Main protocol listening port for incoming connections (both TCP and UDP for QUIC). Essential for communication between devices.
    *   `21027:21027/udp`: Used for local peer discovery.
    *   Ensure these ports are allowed through your host's firewall if you want to sync with devices outside your local network. Port forwarding on your router might be necessary for port `22000` (TCP/UDP).
*   **Web UI Security:** The LinuxServer.io documentation strongly suggests setting a username and password for the Syncthing Web UI, especially if the UI port (8384) is exposed. Do this via `Actions -> Settings -> GUI` in the Syncthing UI after initial setup.
*   **Hostname:** Setting a `hostname` in the compose file can make it easier to identify this Syncthing instance in the Syncthing network and UI of other devices.
*   **Network Mode:**
    *   The default bridge network mode is generally fine.
    *   For potentially better peer discovery, especially in complex network setups or when dealing with NAT, `network_mode: host` can be used. However, this exposes all of the container's ports directly on the host and gives the container more network privileges, which is a security consideration. If using `host` mode, the `ports` mapping section is ignored (as all container ports are automatically mapped).
*   **Initial Setup:**
    1.  Access the Web UI at `http://<your-host-ip>:8384`.
    2.  Set a GUI username and password.
    3.  Add remote devices by their Device ID.
    4.  Add folders you want to sync (pointing to the paths you mapped as volumes, e.g., `/data1`).
    5.  Share folders with your remote devices.

---
