# Docker-Compose Templates for my Useful services

### Ollama & AnythingLLM Combo (ai)
```yaml
version: '3.8'

services:
  anythingllm:
    image: mintplexlabs/anythingllm:latest
    container_name: anythingllm
    ports:
      - "3001:3001" # AnythingLLM Web UI Port
    volumes:
      - anythingllm_storage:/app/server/storage
      - anythingllm_hotdir:/app/collector/hotdir
    environment:
      # Example: Configure AnythingLLM to connect to the Ollama service
      # You might need to configure this within the AnythingLLM UI after startup
      OLLAMA_BASE_URL: http://host.docker.internal:11434 # Check AnythingLLM docs for correct env var if needed
      STORAGE_DIR: /app/server/storage # Ensure persistence
    depends_on:
      - ollama
    restart: unless-stopped

  ollama:
  	pull_policy: always
  	tty: true
    image: ollama/ollama:latest
    container_name: ollama
    restart: unless-stopped
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu] # Request GPU access
    environment:
      - NVIDIA_VISIBLE_DEVICES=all # Make all GPUs visible
      - NVIDIA_DRIVER_CAPABILITIES=compute,utility # Specify required capabilities
      - OLLAMA_KEEP_ALIVE=24h
    volumes:
      - ollama_data:/root/.ollama # Ollama data persistence (adjust path if needed based on image user/setup)
    ports:
      - "11434:11434" # Ollama API Port

volumes:
  anythingllm_storage:
    driver: local
  anythingllm_hotdir:
    driver: local
  ollama_data:
    driver: local
```

### OwnCloud (ai?)
```yaml
version: "3"

volumes:
  files:
    driver: local
  mysql:
    driver: local
  redis:
    driver: local

services:
  owncloud:
    image: owncloud/server:${OWNCLOUD_VERSION}
    container_name: owncloud_server
    restart: always
    ports:
      - ${HTTP_PORT}:8080
    depends_on:
      - mariadb
      - redis
    environment:
      - OWNCLOUD_DOMAIN=${OWNCLOUD_DOMAIN}
      - OWNCLOUD_TRUSTED_DOMAINS=${OWNCLOUD_TRUSTED_DOMAINS}
      - OWNCLOUD_DB_TYPE=mysql
      - OWNCLOUD_DB_NAME=owncloud
      - OWNCLOUD_DB_USERNAME=owncloud
      - OWNCLOUD_DB_PASSWORD=owncloud
      - OWNCLOUD_DB_HOST=mariadb
      - OWNCLOUD_ADMIN_USERNAME=${ADMIN_USERNAME}
      - OWNCLOUD_ADMIN_PASSWORD=${ADMIN_PASSWORD}
      - OWNCLOUD_MYSQL_UTF8MB4=true
      - OWNCLOUD_REDIS_ENABLED=true
      - OWNCLOUD_REDIS_HOST=redis
    healthcheck:
      test: ["CMD", "/usr/bin/healthcheck"]
      interval: 30s
      timeout: 10s
      retries: 5
    volumes:
      - files:/mnt/data

  mariadb:
    image: mariadb:10.11 # minimum required ownCloud version is 10.9
    container_name: owncloud_mariadb
    restart: always
    environment:
      - MYSQL_ROOT_PASSWORD=owncloud
      - MYSQL_USER=owncloud
      - MYSQL_PASSWORD=owncloud
      - MYSQL_DATABASE=owncloud
      - MARIADB_AUTO_UPGRADE=1
    command: ["--max-allowed-packet=128M", "--innodb-log-file-size=64M"]
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-u", "root", "--password=owncloud"]
      interval: 10s
      timeout: 5s
      retries: 5
    volumes:
      - mysql:/var/lib/mysql

  redis:
    image: redis:6
    container_name: owncloud_redis
    restart: always
    command: ["--databases", "1"]
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    volumes:
      - redis:/data
```
