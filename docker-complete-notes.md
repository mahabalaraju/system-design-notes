# Docker — Complete Notes

> Advanced reference covering Docker architecture, images, containers, networking, volumes, Docker Compose, and production best practices.

---

## Table of Contents

- [What is Docker?](#what-is-docker)
- [Architecture](#architecture)
- [Installation & Setup](#installation--setup)
- [Images](#images)
  - [Dockerfile](#dockerfile)
  - [Dockerfile Instructions](#dockerfile-instructions)
  - [Multi-Stage Builds](#multi-stage-builds)
  - [Build Cache](#build-cache)
  - [Image Management](#image-management)
- [Containers](#containers)
  - [Lifecycle](#lifecycle)
  - [Run Options](#run-options)
  - [Exec & Inspect](#exec--inspect)
  - [Logs](#logs)
  - [Resource Limits](#resource-limits)
- [Networking](#networking)
  - [Network Types](#network-types)
  - [DNS & Service Discovery](#dns--service-discovery)
  - [Port Mapping](#port-mapping)
- [Volumes & Storage](#volumes--storage)
  - [Volume Types](#volume-types)
  - [Volume Management](#volume-management)
- [Docker Compose](#docker-compose)
  - [Compose File Structure](#compose-file-structure)
  - [Full Java Microservice Example](#full-java-microservice-example)
  - [Compose Commands](#compose-commands)
  - [Profiles](#profiles)
  - [Override Files](#override-files)
- [Docker for Java Applications](#docker-for-java-applications)
  - [Optimized Java Dockerfile](#optimized-java-dockerfile)
  - [JVM in Containers](#jvm-in-containers)
  - [Spring Boot Layered Jars](#spring-boot-layered-jars)
- [Registry & Image Distribution](#registry--image-distribution)
- [Security Best Practices](#security-best-practices)
- [Production Best Practices](#production-best-practices)
- [Debugging & Troubleshooting](#debugging--troubleshooting)
- [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## What is Docker?

Docker is a platform for packaging, distributing, and running applications in **containers** — lightweight, isolated environments that include the application and all its dependencies.

**VM vs Container:**

```
Virtual Machine:                    Container:
┌─────────────────────┐            ┌─────────────────────┐
│   App A  │  App B   │            │   App A  │  App B   │
│  Libs    │  Libs    │            │  Libs    │  Libs    │
│  Guest OS│  Guest OS│            ├──────────┴──────────┤
│──────────┴──────────│            │   Container Runtime  │
│     Hypervisor       │            │   (Docker Engine)    │
│     Host OS          │            │   Host OS            │
│     Hardware         │            │   Hardware           │
└─────────────────────┘            └─────────────────────┘
  Heavy (~GBs), slow boot            Light (~MBs), instant start
  Full OS isolation                  Process-level isolation
  Strong security boundary           Shared kernel
```

**Core concepts:**
- **Image** — read-only template (blueprint). Built from a `Dockerfile`.
- **Container** — running instance of an image. Isolated process with its own filesystem, network, PID namespace.
- **Registry** — storage for images (Docker Hub, ECR, GCR, private registries).
- **Layer** — each instruction in a Dockerfile creates a read-only layer. Layers are cached and shared.

---

## Architecture

```
┌──────────────────────────────────────────────────────┐
│                   Docker Client (CLI)                 │
│   docker build / run / push / pull / compose up      │
└────────────────────┬─────────────────────────────────┘
                     │ REST API (Unix socket / TCP)
┌────────────────────▼─────────────────────────────────┐
│                  Docker Daemon (dockerd)               │
│   Manages: images, containers, networks, volumes      │
└────┬───────────────┬──────────────────────┬──────────┘
     │               │                      │
┌────▼───┐    ┌──────▼──────┐    ┌──────────▼────────┐
│containerd│  │  Image Store │    │  Network / Volume  │
│(runtime) │  │  (layers)    │    │  Plugins           │
└────┬───┘    └─────────────┘    └────────────────────┘
     │
┌────▼───────┐
│  runc       │  ← OCI runtime — creates actual Linux namespaces & cgroups
└────────────┘
```

**Key components:**
- **dockerd** — daemon process; manages containers
- **containerd** — container lifecycle management (start, stop, pull)
- **runc** — low-level OCI runtime; creates namespaces and cgroups
- **Docker CLI** — sends commands to dockerd via REST API over Unix socket

---

## Installation & Setup

```bash
# Ubuntu / Debian
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER   # run docker without sudo (re-login required)
newgrp docker                   # apply group change without re-login

# Verify
docker --version
docker info
docker run hello-world

# Docker Compose (v2 — plugin, not standalone)
docker compose version          # v2 — built into Docker CLI
# legacy: docker-compose (v1)

# Configure daemon (optional)
sudo nano /etc/docker/daemon.json
```

```json
// /etc/docker/daemon.json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "default-ulimits": {
    "nofile": { "Name": "nofile", "Hard": 64000, "Soft": 64000 }
  },
  "storage-driver": "overlay2"
}
```

---

## Images

### Dockerfile

A `Dockerfile` is a text file with instructions to build an image. Each instruction creates a new layer.

```dockerfile
# Base image
FROM eclipse-temurin:21-jre-alpine

# Metadata
LABEL maintainer="mahabalaraju@example.com"
LABEL version="1.0"
LABEL description="Order Service"

# Build arguments (available only during build)
ARG APP_VERSION=1.0.0
ARG BUILD_DATE

# Environment variables (available at runtime too)
ENV APP_HOME=/app \
    SPRING_PROFILES_ACTIVE=prod \
    JAVA_OPTS="-Xms256m -Xmx512m"

# Working directory — creates if not exists
WORKDIR /app

# Copy files (invalidates cache if file changes)
COPY target/order-service.jar app.jar

# Run commands during build
RUN addgroup -S appgroup && adduser -S appuser -G appgroup \
    && mkdir -p /app/logs \
    && chown -R appuser:appgroup /app

# Switch to non-root user
USER appuser

# Document which port the app listens on (informational only)
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8080/actuator/health || exit 1

# Default command
ENTRYPOINT ["java", "-jar", "/app/app.jar"]

# Default args to ENTRYPOINT (can be overridden)
CMD ["--spring.profiles.active=prod"]
```

---

### Dockerfile Instructions

| Instruction | Purpose | Notes |
|---|---|---|
| `FROM` | Base image | Always first. `FROM scratch` = empty image |
| `RUN` | Execute command during build | Each RUN creates a layer — chain commands with `&&` |
| `COPY` | Copy files from host to image | Prefer over ADD for simple copies |
| `ADD` | Copy + auto-extract tarballs, supports URLs | Use only when extraction needed |
| `WORKDIR` | Set working directory | Creates dir if missing; affects subsequent instructions |
| `ENV` | Set environment variable | Persists in image and running container |
| `ARG` | Build-time variable | Not available at runtime; use for build config |
| `EXPOSE` | Document port | Informational only — doesn't publish the port |
| `VOLUME` | Create mount point | Declares a volume; data persists beyond container |
| `USER` | Set user for subsequent instructions | Always use non-root in production |
| `ENTRYPOINT` | Main command (not easily overridden) | Use for the executable |
| `CMD` | Default arguments to ENTRYPOINT | Easily overridden at `docker run` |
| `HEALTHCHECK` | Container health probe | Used by Docker and orchestrators |
| `LABEL` | Key-value metadata | Useful for filtering, automation |
| `ONBUILD` | Trigger instruction for child images | For base image builders |
| `.dockerignore` | Exclude files from build context | Like `.gitignore` |

**ENTRYPOINT vs CMD:**

```dockerfile
# ENTRYPOINT — fixed executable
# CMD — default args, can be overridden at docker run

# Exec form (preferred — no shell, signals work correctly)
ENTRYPOINT ["java", "-jar", "app.jar"]
CMD ["--server.port=8080"]

# Shell form (avoid — wraps in /bin/sh -c, signals don't reach process)
ENTRYPOINT java -jar app.jar    # ❌ PID 1 is shell, not java

# Override at runtime
docker run myapp --server.port=9090   # replaces CMD
docker run --entrypoint /bin/sh myapp # replaces ENTRYPOINT
```

**.dockerignore:**

```
# .dockerignore
.git
.gitignore
**/*.md
target/
!target/app.jar          # exception — include this
.env
*.log
Dockerfile*
docker-compose*
.DS_Store
node_modules/
```

---

### Multi-Stage Builds

Separate build environment from runtime image — dramatically smaller final image.

```dockerfile
# ── Stage 1: Build ──────────────────────────────────────
FROM maven:3.9-eclipse-temurin-21 AS builder

WORKDIR /build

# Cache dependency downloads separately from source changes
COPY pom.xml .
COPY .mvn/ .mvn/
RUN mvn dependency:go-offline -B          # download all deps into cache layer

# Now copy source — this layer only invalidates when source changes
COPY src/ src/
RUN mvn package -DskipTests -B            # build the jar

# ── Stage 2: Extract layers (Spring Boot layered jar) ───
FROM builder AS extractor
RUN java -Djarmode=layertools -jar target/app.jar extract --destination /layers

# ── Stage 3: Runtime ─────────────────────────────────────
FROM eclipse-temurin:21-jre-alpine AS runtime

# Security: non-root user
RUN addgroup -S spring && adduser -S spring -G spring
USER spring

WORKDIR /app

# Copy Spring Boot layers in order of change frequency (least → most)
COPY --from=extractor /layers/dependencies/          ./
COPY --from=extractor /layers/spring-boot-loader/    ./
COPY --from=extractor /layers/snapshot-dependencies/ ./
COPY --from=extractor /layers/application/           ./

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=5s --start-period=40s --retries=3 \
    CMD wget -qO- http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

**Size comparison:**
```
maven:3.9-eclipse-temurin-21      ~600MB  (build stage — discarded)
eclipse-temurin:21-jdk-alpine     ~340MB  (with JDK — overkill for runtime)
eclipse-temurin:21-jre-alpine     ~185MB  (JRE only — use this)
Final multi-stage image            ~220MB  (JRE + app layers)
```

---

### Build Cache

Docker caches each layer. A cache miss invalidates all subsequent layers.

```dockerfile
# ❌ Bad — COPY source invalidates dependency cache on every code change
COPY . .
RUN mvn package

# ✅ Good — dependencies cached separately from source
COPY pom.xml .
RUN mvn dependency:go-offline    # cached as long as pom.xml unchanged
COPY src/ src/
RUN mvn package                  # only this layer rebuilds on source change
```

```bash
# Build with no cache
docker build --no-cache -t myapp .

# Show build steps and cache usage
docker build --progress=plain -t myapp .

# BuildKit (enabled by default in Docker 23+)
DOCKER_BUILDKIT=1 docker build -t myapp .

# Build with cache from registry (CI/CD optimization)
docker build \
  --cache-from myrepo/myapp:latest \
  --tag myrepo/myapp:latest .
```

---

### Image Management

```bash
# Build
docker build -t myapp:1.0.0 .
docker build -t myapp:1.0.0 -f Dockerfile.prod .
docker build --build-arg APP_VERSION=1.0.0 -t myapp:1.0.0 .

# List
docker images
docker image ls --filter dangling=true    # untagged images (<none>)

# Tag
docker tag myapp:1.0.0 myrepo/myapp:1.0.0
docker tag myapp:1.0.0 myrepo/myapp:latest

# Push / Pull
docker push myrepo/myapp:1.0.0
docker pull eclipse-temurin:21-jre-alpine

# Inspect
docker image inspect myapp:1.0.0
docker image history myapp:1.0.0          # show layers and sizes

# Remove
docker rmi myapp:1.0.0
docker image prune                        # remove dangling images
docker image prune -a                     # remove all unused images

# Save / Load (for air-gapped environments)
docker save -o myapp.tar myapp:1.0.0
docker load -i myapp.tar

# Export / Import (flattens layers — loses metadata)
docker export container_name | docker import - myapp:flat
```

---

## Containers

### Lifecycle

```
Created → Running → Paused → Running
              ↓
           Stopped → Removed
              ↓
            Exited
```

```bash
# Create (does not start)
docker create --name mycontainer myapp:1.0.0

# Start
docker start mycontainer

# Run = create + start
docker run myapp:1.0.0
docker run -d myapp:1.0.0              # detached (background)
docker run -it ubuntu:22.04 bash       # interactive + TTY

# Stop (SIGTERM → wait → SIGKILL after timeout)
docker stop mycontainer                # 10s default
docker stop -t 30 mycontainer         # 30s timeout

# Kill (SIGKILL immediately)
docker kill mycontainer

# Restart
docker restart mycontainer
docker restart -t 5 mycontainer

# Pause / Unpause (freeze processes — SIGSTOP)
docker pause mycontainer
docker unpause mycontainer

# Remove
docker rm mycontainer                  # must be stopped first
docker rm -f mycontainer               # force remove running container
docker container prune                 # remove all stopped containers

# List
docker ps                              # running
docker ps -a                           # all (including stopped)
docker ps --filter status=exited
docker ps --filter name=order
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

---

### Run Options

```bash
# Naming & detach
docker run -d --name order-service myapp:1.0.0

# Port mapping (host:container)
docker run -p 8080:8080 myapp:1.0.0
docker run -p 127.0.0.1:8080:8080 myapp:1.0.0    # bind to localhost only
docker run -P myapp:1.0.0                          # auto-assign host ports

# Environment variables
docker run -e SPRING_PROFILES_ACTIVE=prod myapp:1.0.0
docker run --env-file .env myapp:1.0.0

# Volume mounts
docker run -v myvolume:/app/data myapp:1.0.0       # named volume
docker run -v /host/path:/app/data myapp:1.0.0     # bind mount
docker run --mount type=volume,src=myvolume,dst=/app/data myapp:1.0.0

# Network
docker run --network my-network myapp:1.0.0
docker run --network host myapp:1.0.0              # use host network stack

# Resource limits
docker run --memory=512m --cpus=1.0 myapp:1.0.0

# Restart policy
docker run --restart=unless-stopped myapp:1.0.0
docker run --restart=on-failure:3 myapp:1.0.0      # retry up to 3 times

# Remove on exit
docker run --rm myapp:1.0.0                        # auto-remove when stopped

# Working directory
docker run -w /app myapp:1.0.0

# User
docker run -u 1001:1001 myapp:1.0.0
docker run -u appuser myapp:1.0.0

# Override command
docker run myapp:1.0.0 --spring.profiles.active=dev

# Override entrypoint
docker run --entrypoint /bin/sh myapp:1.0.0
```

**Restart policies:**

| Policy | Behaviour |
|---|---|
| `no` (default) | Never restart |
| `always` | Always restart (even on manual stop — starts on daemon restart) |
| `unless-stopped` | Restart unless manually stopped |
| `on-failure[:max]` | Restart only on non-zero exit code |

---

### Exec & Inspect

```bash
# Execute command in running container
docker exec -it mycontainer bash
docker exec -it mycontainer sh              # alpine — no bash
docker exec mycontainer cat /app/config.yml
docker exec -e DEBUG=true mycontainer env   # with env var

# Inspect container metadata (JSON)
docker inspect mycontainer
docker inspect mycontainer --format '{{.State.Status}}'
docker inspect mycontainer --format '{{.NetworkSettings.IPAddress}}'
docker inspect mycontainer --format '{{json .Config.Env}}' | jq

# Copy files
docker cp mycontainer:/app/logs/app.log ./app.log   # container → host
docker cp ./config.yml mycontainer:/app/             # host → container

# Stats — real-time resource usage
docker stats
docker stats mycontainer --no-stream     # snapshot

# Top — processes in container
docker top mycontainer

# Diff — changed files since container start
docker diff mycontainer
```

---

### Logs

```bash
# View logs
docker logs mycontainer
docker logs -f mycontainer               # follow (tail -f)
docker logs --tail 100 mycontainer       # last 100 lines
docker logs --since 10m mycontainer      # last 10 minutes
docker logs --since 2024-01-01T00:00:00 mycontainer
docker logs -t mycontainer               # with timestamps

# Log drivers (configure in daemon.json or per container)
docker run --log-driver=json-file \
           --log-opt max-size=10m \
           --log-opt max-file=3 \
           myapp:1.0.0

# Other log drivers
--log-driver=fluentd
--log-driver=gelf        # Graylog
--log-driver=awslogs     # CloudWatch
--log-driver=none        # disable logging
```

---

### Resource Limits

```bash
# Memory
docker run --memory=512m myapp             # hard limit — OOM kill at 512MB
docker run --memory=512m --memory-swap=512m myapp  # disable swap
docker run --memory-reservation=256m myapp  # soft limit

# CPU
docker run --cpus=1.5 myapp               # 1.5 CPU cores
docker run --cpu-shares=512 myapp         # relative weight (default 1024)
docker run --cpuset-cpus="0,1" myapp      # pin to specific cores

# Combined
docker run \
  --memory=512m \
  --memory-swap=512m \
  --cpus=1.0 \
  --pids-limit=100 \
  myapp:1.0.0

# Check limits
docker inspect mycontainer | grep -i memory
docker stats mycontainer
```

---

## Networking

### Network Types

```bash
# List networks
docker network ls

# Default networks:
# bridge    — default for standalone containers; isolated from host
# host      — container shares host network stack (no isolation)
# none      — no network access
```

```bash
# Bridge network (default)
docker run -d --name app1 nginx              # on default bridge
# Containers on default bridge can only communicate by IP (not name)

# User-defined bridge — enables DNS by container name ✅
docker network create my-network
docker run -d --name app1 --network my-network nginx
docker run -d --name app2 --network my-network curlimages/curl
docker exec app2 curl http://app1            # works — DNS by name

# Inspect network
docker network inspect my-network

# Connect / disconnect running container
docker network connect my-network mycontainer
docker network disconnect my-network mycontainer

# Remove
docker network rm my-network
docker network prune                          # remove unused networks

# Host network (Linux only — maximum performance, no isolation)
docker run --network host nginx               # nginx listens on host port 80

# None — no network
docker run --network none myapp              # completely isolated
```

---

### DNS & Service Discovery

```bash
# User-defined bridge networks provide automatic DNS:
# container name → container IP resolution

docker network create backend
docker run -d --name postgres --network backend postgres:16
docker run -d --name order-service --network backend \
  -e DB_HOST=postgres \                      # use container name as hostname
  myapp:1.0.0

# Custom DNS alias
docker run -d --name db1 --network backend \
  --network-alias database \                 # additional hostname
  postgres:16

# Connect to multiple networks
docker network create frontend
docker run -d --name api \
  --network backend \
  myapi:1.0.0
docker network connect frontend api          # api reachable from both networks
```

---

### Port Mapping

```bash
# -p hostPort:containerPort
docker run -p 8080:8080 myapp          # all interfaces
docker run -p 127.0.0.1:8080:8080 myapp  # localhost only (more secure)
docker run -p 8080:8080/udp myapp      # UDP

# Multiple ports
docker run -p 8080:8080 -p 9090:9090 myapp

# Random host port
docker run -p 8080 myapp               # Docker assigns random host port
docker port myapp                      # check assigned port

# -P (capital) — publish all EXPOSE'd ports to random host ports
docker run -P myapp
```

---

## Volumes & Storage

### Volume Types

```
Named Volume:
  docker run -v myvolume:/app/data
  - Managed by Docker (/var/lib/docker/volumes/)
  - Best for: persistent data (DB, uploads)
  - Not dependent on host directory structure

Bind Mount:
  docker run -v /host/path:/container/path
  - Maps host directory into container
  - Best for: development (hot reload), config files
  - Host path must exist

tmpfs Mount:
  docker run --tmpfs /app/temp
  - In-memory only, not persisted
  - Best for: secrets, temporary scratch space
  - Cleared when container stops
```

---

### Volume Management

```bash
# Create
docker volume create myvolume
docker volume create --driver local \
  --opt type=nfs \
  --opt o=addr=192.168.1.1,rw \
  --opt device=:/path/to/share \
  nfs-volume

# List
docker volume ls
docker volume ls --filter dangling=true     # unused volumes

# Inspect
docker volume inspect myvolume

# Remove
docker volume rm myvolume
docker volume prune                          # remove all unused volumes

# Use in run
docker run -v myvolume:/app/data myapp
docker run --mount type=volume,src=myvolume,dst=/app/data myapp

# Bind mount
docker run -v $(pwd)/config:/app/config:ro myapp   # :ro = read-only
docker run -v $(pwd)/src:/app/src myapp             # dev hot reload

# Backup a volume
docker run --rm \
  -v myvolume:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/myvolume-backup.tar.gz -C /data .

# Restore
docker run --rm \
  -v myvolume:/data \
  -v $(pwd):/backup \
  alpine tar xzf /backup/myvolume-backup.tar.gz -C /data
```

---

## Docker Compose

### Compose File Structure

```yaml
# docker-compose.yml
version: "3.9"    # optional in Compose v2+

services:
  service-name:
    image: image:tag              # use existing image
    build:                        # OR build from Dockerfile
      context: .
      dockerfile: Dockerfile
      args:
        APP_VERSION: "1.0.0"
    container_name: my-container
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=dev
      - DB_HOST=postgres
    env_file:
      - .env
    volumes:
      - myvolume:/app/data
      - ./config:/app/config:ro
    networks:
      - backend
    depends_on:
      postgres:
        condition: service_healthy   # wait for health check
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:8080/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: "1.0"
        reservations:
          memory: 256M

volumes:
  myvolume:
    driver: local

networks:
  backend:
    driver: bridge
  frontend:
    driver: bridge
```

---

### Full Java Microservice Example

```yaml
# docker-compose.yml — Order Service with dependencies
version: "3.9"

services:

  # ── PostgreSQL ──────────────────────────────────────────
  postgres:
    image: postgres:16-alpine
    container_name: postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${DB_USER:-appuser}
      POSTGRES_PASSWORD: ${DB_PASSWORD:-secret}
      POSTGRES_DB: ${DB_NAME:-orderdb}
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./db/init:/docker-entrypoint-initdb.d:ro   # init SQL scripts
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER:-appuser} -d ${DB_NAME:-orderdb}"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s

  # ── Redis ───────────────────────────────────────────────
  redis:
    image: redis:7-alpine
    container_name: redis
    restart: unless-stopped
    command: redis-server --requirepass ${REDIS_PASSWORD:-redispass} --appendonly yes
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    networks:
      - backend
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD:-redispass}", "ping"]
      interval: 10s
      timeout: 5s
      retries: 3

  # ── Kafka ───────────────────────────────────────────────
  zookeeper:
    image: confluentinc/cp-zookeeper:7.6.0
    container_name: zookeeper
    restart: unless-stopped
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    networks:
      - backend

  kafka:
    image: confluentinc/cp-kafka:7.6.0
    container_name: kafka
    restart: unless-stopped
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    networks:
      - backend
    healthcheck:
      test: ["CMD", "kafka-broker-api-versions", "--bootstrap-server", "localhost:9092"]
      interval: 30s
      timeout: 10s
      retries: 5

  # ── Order Service ────────────────────────────────────────
  order-service:
    build:
      context: .
      dockerfile: Dockerfile
      target: runtime             # stop at specific multi-stage stage
    container_name: order-service
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      SPRING_PROFILES_ACTIVE: docker
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/${DB_NAME:-orderdb}
      SPRING_DATASOURCE_USERNAME: ${DB_USER:-appuser}
      SPRING_DATASOURCE_PASSWORD: ${DB_PASSWORD:-secret}
      SPRING_DATA_REDIS_HOST: redis
      SPRING_DATA_REDIS_PASSWORD: ${REDIS_PASSWORD:-redispass}
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:29092
      JAVA_OPTS: "-Xms256m -Xmx512m -XX:+UseContainerSupport"
    volumes:
      - order_logs:/app/logs
    networks:
      - backend
      - frontend
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
      kafka:
        condition: service_healthy
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://localhost:8080/actuator/health || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
    deploy:
      resources:
        limits:
          memory: 600M
          cpus: "1.0"

  # ── Nginx Reverse Proxy ──────────────────────────────────
  nginx:
    image: nginx:alpine
    container_name: nginx
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
    networks:
      - frontend
    depends_on:
      order-service:
        condition: service_healthy

volumes:
  postgres_data:
  redis_data:
  order_logs:

networks:
  backend:
    driver: bridge
  frontend:
    driver: bridge
```

---

### Compose Commands

```bash
# Start services
docker compose up                         # foreground
docker compose up -d                      # detached (background)
docker compose up --build                 # rebuild images before start
docker compose up --force-recreate        # recreate containers even if config unchanged
docker compose up order-service           # start specific service only

# Stop
docker compose down                       # stop and remove containers + networks
docker compose down -v                    # also remove named volumes ⚠️
docker compose down --rmi all             # also remove images
docker compose stop                       # stop without removing

# Scale
docker compose up -d --scale order-service=3   # run 3 replicas

# Logs
docker compose logs
docker compose logs -f order-service      # follow specific service
docker compose logs --tail 100

# Exec
docker compose exec order-service bash
docker compose exec postgres psql -U appuser -d orderdb

# Build
docker compose build
docker compose build order-service
docker compose build --no-cache

# Status
docker compose ps
docker compose top

# Config
docker compose config                     # validate and print resolved config

# Pull latest images
docker compose pull

# Restart
docker compose restart order-service
```

---

### Profiles

```yaml
# docker-compose.yml
services:
  order-service:
    image: myapp:1.0.0
    # no profile — always starts

  pgadmin:
    image: dpage/pgadmin4
    profiles: ["tools"]             # only starts with --profile tools
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@example.com
      PGADMIN_DEFAULT_PASSWORD: admin

  prometheus:
    image: prom/prometheus
    profiles: ["monitoring"]

  grafana:
    image: grafana/grafana
    profiles: ["monitoring"]
```

```bash
# Start with specific profiles
docker compose --profile tools up -d
docker compose --profile monitoring up -d
docker compose --profile tools --profile monitoring up -d

# Or via env var
COMPOSE_PROFILES=tools,monitoring docker compose up -d
```

---

### Override Files

```bash
# docker-compose.yml          — base (committed to git)
# docker-compose.override.yml — auto-applied dev overrides (gitignored)
# docker-compose.prod.yml     — production overrides

# Base
# docker-compose.yml
services:
  order-service:
    image: myapp:1.0.0
    ports:
      - "8080:8080"
```

```yaml
# docker-compose.override.yml (dev — auto-loaded)
services:
  order-service:
    build: .                      # build locally in dev
    environment:
      SPRING_PROFILES_ACTIVE: dev
      SPRING_JPA_SHOW_SQL: "true"
    volumes:
      - ./target:/app/target      # hot reload in dev
```

```yaml
# docker-compose.prod.yml
services:
  order-service:
    restart: always
    deploy:
      resources:
        limits:
          memory: 1G
    logging:
      driver: awslogs
      options:
        awslogs-group: /order-service
```

```bash
# Dev (auto-loads override)
docker compose up

# Production (explicit files)
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

---

## Docker for Java Applications

### Optimized Java Dockerfile

```dockerfile
# ── Stage 1: Download dependencies ──────────────────────
FROM maven:3.9-eclipse-temurin-21-alpine AS deps
WORKDIR /build
COPY pom.xml .
COPY .mvn/ .mvn/
RUN mvn dependency:go-offline -B --no-transfer-progress

# ── Stage 2: Build ───────────────────────────────────────
FROM deps AS build
COPY src/ src/
RUN mvn package -DskipTests -B --no-transfer-progress

# ── Stage 3: Extract Spring Boot layers ─────────────────
FROM eclipse-temurin:21-jre-alpine AS extractor
WORKDIR /extract
COPY --from=build /build/target/*.jar app.jar
RUN java -Djarmode=layertools -jar app.jar extract

# ── Stage 4: Runtime ─────────────────────────────────────
FROM eclipse-temurin:21-jre-alpine AS runtime

# Security hardening
RUN addgroup -S spring && adduser -S spring -G spring \
    && mkdir -p /app/logs \
    && chown -R spring:spring /app

USER spring
WORKDIR /app

# Copy layers — least to most frequently changed
COPY --from=extractor /extract/dependencies/          ./
COPY --from=extractor /extract/spring-boot-loader/   ./
COPY --from=extractor /extract/snapshot-dependencies/ ./
COPY --from=extractor /extract/application/           ./

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=5s --start-period=45s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["java", \
    "-XX:+UseContainerSupport", \
    "-XX:MaxRAMPercentage=75.0", \
    "-XX:InitialRAMPercentage=50.0", \
    "-Djava.security.egd=file:/dev/./urandom", \
    "org.springframework.boot.loader.launch.JarLauncher"]
```

---

### JVM in Containers

Before JDK 10, the JVM couldn't detect container memory/CPU limits — it read host values and over-allocated heap.

```bash
# ✅ Modern JVM flags for containers (JDK 11+)
-XX:+UseContainerSupport          # enables container awareness (default on)
-XX:MaxRAMPercentage=75.0         # use 75% of container memory as max heap
-XX:InitialRAMPercentage=50.0     # start with 50% of container memory
-XX:MinRAMPercentage=25.0

# Example: 512MB container → max heap ~384MB

# ❌ Old approach — hardcoded heap (breaks when container limits change)
-Xmx512m -Xms256m

# Verify JVM sees container limits
docker run --memory=512m eclipse-temurin:21 java -XX:+PrintFlagsFinal -version | grep MaxHeapSize
# Should show: MaxHeapSize = 402653184 (≈384MB = 75% of 512MB)

# GC tuning for containers
-XX:+UseG1GC                       # default in JDK 9+ (good general choice)
-XX:+UseZGC                        # low-latency (JDK 15+, pauseless)
-XX:+UseShenandoahGC               # low-latency alternative

# Faster startup (serverless / short-lived containers)
-XX:TieredStopAtLevel=1            # only C1 compiler — faster start, lower peak throughput
```

---

### Spring Boot Layered Jars

Spring Boot 2.3+ supports layered jars — optimizes Docker layer caching.

```xml
<!-- pom.xml — enable layers (default in Spring Boot 2.3+) -->
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <layers>
            <enabled>true</enabled>
        </layers>
    </configuration>
</plugin>
```

```bash
# Inspect layers
java -Djarmode=layertools -jar target/app.jar list
# dependencies          ← 3rd party libs (changes rarely)
# spring-boot-loader    ← Spring Boot loader (changes rarely)
# snapshot-dependencies ← SNAPSHOT libs (changes occasionally)
# application           ← your code (changes every build)

# Each layer becomes a separate Docker layer
# Only changed layers need to be pushed/pulled
```

---

## Registry & Image Distribution

```bash
# Docker Hub
docker login
docker tag myapp:1.0.0 username/myapp:1.0.0
docker push username/myapp:1.0.0
docker pull username/myapp:1.0.0
docker logout

# Private registry
docker login registry.example.com
docker tag myapp:1.0.0 registry.example.com/myapp:1.0.0
docker push registry.example.com/myapp:1.0.0

# AWS ECR
aws ecr get-login-password | docker login --username AWS \
  --password-stdin 123456789.dkr.ecr.ap-south-1.amazonaws.com
docker push 123456789.dkr.ecr.ap-south-1.amazonaws.com/myapp:1.0.0

# List tags
docker search myapp
curl -s https://registry.hub.docker.com/v2/repositories/library/postgres/tags/ | jq

# Image digest (immutable reference)
docker pull myapp@sha256:abc123...      # pulls exact image, not just tag
docker images --digests

# Multi-platform build (build for ARM + AMD64)
docker buildx create --use
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t myrepo/myapp:1.0.0 \
  --push .
```

---

## Security Best Practices

```dockerfile
# 1. Use specific image tags — never :latest in production
FROM eclipse-temurin:21.0.3_9-jre-alpine    # pinned version

# 2. Non-root user — always
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# 3. Read-only filesystem
docker run --read-only --tmpfs /tmp --tmpfs /app/logs myapp

# 4. Drop capabilities
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE myapp

# 5. No new privileges
docker run --security-opt no-new-privileges myapp

# 6. Minimal base image
FROM scratch                                 # empty image
FROM alpine:3.19                             # 5MB OS
FROM eclipse-temurin:21-jre-alpine          # JRE only (not JDK)
FROM gcr.io/distroless/java21               # Google distroless — no shell, no pkg manager

# 7. Don't embed secrets in image
# ❌ Bad
ENV DB_PASSWORD=supersecret
RUN echo "password=secret" > /app/config

# ✅ Use --secret (BuildKit) or environment at runtime
RUN --mount=type=secret,id=db_password \
    cat /run/secrets/db_password > /app/secret.txt

# 8. .dockerignore — exclude sensitive files
# .env, *.pem, *.key, credentials, target/

# 9. Scan for vulnerabilities
docker scout cves myapp:1.0.0               # Docker Scout (built-in)
trivy image myapp:1.0.0                     # Trivy (open source)
grype myapp:1.0.0                           # Anchore Grype
```

---

## Production Best Practices

```dockerfile
# 1. One process per container
# ✅ Each service in its own container
# ❌ Running multiple services with supervisord

# 2. Stateless containers — store state in volumes or external services
# ✅ App container is replaceable
# ❌ Container writes data to its own filesystem (lost on restart)

# 3. Health checks — always define them
HEALTHCHECK --interval=30s --timeout=5s --start-period=45s --retries=3 \
    CMD wget -qO- http://localhost:8080/actuator/health || exit 1

# 4. Resource limits — always set in production
docker run --memory=512m --cpus=1.0 myapp

# 5. Logging to stdout/stderr (let Docker capture it)
# ✅ Log to console — Docker collects it
# ❌ Log to files inside container — hard to access, fills disk

# 6. Graceful shutdown — handle SIGTERM
# Spring Boot does this by default with server.shutdown=graceful
# ENTRYPOINT in exec form (not shell form) ensures signal reaches JVM

# 7. PID 1 problem — exec form or use tini
ENTRYPOINT ["java", "-jar", "app.jar"]           # ✅ java is PID 1, gets signals
ENTRYPOINT ["tini", "--", "java", "-jar", "app.jar"]  # tini as init for zombie reaping

# 8. Image tagging strategy
myapp:1.2.3               # semantic version — immutable
myapp:1.2                 # minor version — updated on patches
myapp:latest              # ❌ mutable, avoid in production
myapp:git-abc123          # git commit SHA — traceable

# 9. Layer ordering — cache optimization
# Order: rarely changing → frequently changing
COPY pom.xml .            # dependencies
RUN mvn dependency:go-offline
COPY src/ src/            # source (changes most)
RUN mvn package
```

---

## Debugging & Troubleshooting

```bash
# Container won't start — check logs
docker logs mycontainer
docker logs --tail 50 mycontainer

# Check exit code
docker inspect mycontainer --format '{{.State.ExitCode}}'
# 0 = clean exit, 1 = error, 137 = OOM killed, 143 = SIGTERM

# OOM kill check
docker inspect mycontainer --format '{{.State.OOMKilled}}'

# Get a shell in a running container
docker exec -it mycontainer sh

# Get a shell in a stopped container — create new container from same image
docker run -it --rm --entrypoint sh myapp:1.0.0

# Debug a container that exits immediately
docker run -it --rm --entrypoint sh myapp:1.0.0    # override entrypoint

# Check resource usage
docker stats --no-stream

# Inspect network — why can't container A reach B?
docker network inspect my-network
docker exec app1 ping app2
docker exec app1 nslookup app2            # check DNS resolution
docker exec app1 wget -qO- http://app2:8080/health

# Check volumes
docker volume inspect myvolume
docker run --rm -v myvolume:/data alpine ls -la /data

# Image layer sizes — find what's making image large
docker image history myapp:1.0.0
docker image history --no-trunc myapp:1.0.0 | sort -k4 -rh

# Disk usage
docker system df                          # overview
docker system df -v                       # verbose

# Clean everything (⚠️ careful in production)
docker system prune                       # remove stopped containers, unused networks, dangling images
docker system prune -a                    # also remove unused images
docker system prune -a --volumes          # also remove volumes ⚠️
```

---

## Quick Reference Cheat Sheet

### Dockerfile best practices summary

```
FROM        → pin exact version, use alpine/jre/distroless
COPY        → use .dockerignore; copy pom before src (cache)
RUN         → chain with &&; clean up in same layer (rm -rf cache)
USER        → always non-root in final stage
ENTRYPOINT  → exec form ["java", "-jar", "app.jar"]
CMD         → overridable defaults
HEALTHCHECK → always define for production
EXPOSE      → informational; still need -p to publish
WORKDIR     → always set; avoids relative path issues
```

### docker run flags quick ref

```bash
-d                  detached
-it                 interactive + TTY
--rm                auto-remove on exit
--name              container name
-p host:container   port mapping
-e KEY=val          environment variable
--env-file .env     env file
-v vol:/path        volume
--network name      attach to network
--memory 512m       memory limit
--cpus 1.0          CPU limit
--restart policy    restart policy
```

### Compose commands quick ref

```bash
up -d               start detached
up --build          rebuild + start
down                stop + remove
down -v             stop + remove + volumes
logs -f             follow logs
exec svc bash       shell in service
ps                  service status
build               build images
pull                pull latest images
config              validate config
scale svc=3         scale replicas
```

### Common issues

| Issue | Likely cause | Fix |
|---|---|---|
| `Connection refused` | Wrong port or container not up | Check `docker ps`, check `EXPOSE` vs `-p` |
| Container exits immediately | Entrypoint fails or shell form PID issue | `docker logs`; use exec form ENTRYPOINT |
| `OOMKilled` | Container hit memory limit | Increase `--memory` or tune JVM heap |
| `No such container` | Wrong name | `docker ps -a` to find actual name |
| Slow build | Poor layer ordering | Move changing files to end of Dockerfile |
| `Permission denied` | Non-root user, wrong file ownership | `chown` in Dockerfile before `USER` |
| Can't connect between containers | Not on same network | Use user-defined bridge network |
| Old code running | Image not rebuilt | `docker compose up --build` |
| Volume data lost | Using bind mount path that doesn't exist | Create directory first or use named volume |

---

*References: Docker Docs (docs.docker.com) | Docker Compose Spec (compose-spec.io) | Play with Docker (labs.play-with-docker.com)*
