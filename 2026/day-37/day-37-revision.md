# 🐳 Day 37: Docker Revision & Self-Assessment

A comprehensive review of Docker fundamentals, multi-container orchestration, and production best practices.

## 📌 Table of Contents
* [Self-Assessment Checklist](#-self-assessment-checklist)
  * [1. Core Container Operations](#1-core-container-operations)
  * [2. Image Layers & Caching](#2-image-layers--caching)
  * [3. Writing Dockerfiles (`CMD` vs `ENTRYPOINT`)](#3-writing-dockerfiles-cmd-vs-entrypoint)
  * [4. Persistent Storage (Volumes vs Bind Mounts)](#4-persistent-storage-volumes-vs-bind-mounts)
  * [5. Networking & Multi-Container Systems](#5-networking--multi-container-systems)
  * [6. Advanced Compose & Optimization](#6-advanced-compose-&-optimization)
* [🧠 Quick-Fire Concept Answers](#-quick-fire-concept-answers)

---

## 📋 Self-Assessment Checklist

### 1. Core Container Operations

#### ✅ Run a container from Docker Hub (interactive + detached)
```bash
# Detached — runs in the background, returns container ID
docker run -d -p 8080:80 --name mynginx nginx

# Interactive — drops you into an interactive TTY shell inside the container
docker run -it ubuntu bash
```
* **Key flags:** `-d` (detached), `-it` (interactive + pseudo-TTY), `-p host:container` (port mapping), `--name` (explicit naming).

#### ✅ List, stop, remove containers and images
```bash
# Container Management
docker ps             # List running containers
docker ps -a          # List all containers (including stopped)
docker stop mynginx   # Gracefully stop container (SIGTERM)
docker rm mynginx     # Remove a stopped container
docker rm -f mynginx  # Force-remove a running container (SIGKILL)

# Image Management
docker images            # List local images (or: docker image ls)
docker rmi nginx:alpine  # Remove a local image
docker image prune -a    # Purge all unused cached images
```

---

### 2. Image Layers & Caching

#### ✅ Explain image layers and how caching works
A Docker image is an immutable **stack of read-only layers**. Each instruction in a Dockerfile (`RUN`, `COPY`, `ADD`) commits a new layer on top of the previous layer. When a container spins up, Docker places a thin **writable container layer** on top. All runtime data mutations live in this top layer and evaporate when the container is deleted.

**Caching Mechanism:** Docker tracks layers by content hashes. During builds:
1. If a layer's contents and instructions match perfectly, Docker skips building and reuses the cached layer.
2. The instant a layer's cache is busted (e.g., source code changes), **all subsequent downstream layers are completely invalidated** and must rebuild from scratch.

* **Optimization Workflow:** Keep volatile source modifications as low down in the Dockerfile as possible.
```dockerfile
# ❌ BAD: Any tiny code change forces npm install to re-run
COPY . .
RUN npm install

#  GOOD: Node modules only reinstall when dependency files change
COPY package*.json ./
RUN npm install --production
COPY . .
```

---

### 3. Writing Dockerfiles (`CMD` vs `ENTRYPOINT`)

#### ✅ Write a basic Dockerfile from scratch
```dockerfile
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install --production

COPY . .

EXPOSE 3000

CMD ["node", "server.js"]
```
* `FROM`: Selects the base image layer (*always required first*).
* `WORKDIR`: Standardizes the target working path context; provisions it if missing.
* `COPY`: Explicitly transfers files from the local build context to the image environment.
* `RUN`: Fires shell executions during building and commits a fresh image layer.
* `EXPOSE`: Purely informational documentation mapping intended runtime ports.
* `CMD`: Dictates the default process run command when the container initializes.

#### ✅ Explain CMD vs ENTRYPOINT
Both parameters set container startup processes, but they offer completely different structural flexibility.

| Directive | Core Purpose | Overridable at Runtime? | Common Use Case |
| :--- | :--- | :--- | :--- |
| **`CMD`** | Sets fallback commands / arguments | **Yes** (any trailing text to `docker run` overwrites `CMD`) | Default flags or secondary shells |
| **`ENTRYPOINT`** | Locks down the immutable core executable | **No** (runtime arguments append to it instead) | Fixed application binaries |

```dockerfile
# Robust Multi-layered Architecture
ENTRYPOINT ["python", "app.py"]
CMD ["--port", "8000"]
```
* Executing `docker run myapp` -> runs `python app.py --port 8000`
* Executing `docker run myapp --port 9000` -> overrides `CMD` to run `python app.py --port 9000`

#### ✅ Build and tag a custom image
```bash
# Build image locally with tag
docker build -t myapp:1.0 .

# Retag an existing image target for Docker Hub upload
docker tag myapp:1.0 atulsharmadochub/myapp:1.0
docker tag myapp:1.0 atulsharmadochub/myapp:latest
```
* **Format Structure:** `registry/username/repository:tag` (omitting registry defaults directly to Docker Hub).

---

### 4. Persistent Storage (Volumes vs Bind Mounts)

#### ✅ Create and use named volumes
```bash
# Pre-provision volume
docker volume create mydata

# Mount named volume dynamically into a new execution thread
docker run -d -v mydata:/app/data --name myapp postgres

# Inspect the system directory where data resides on host machine
docker volume inspect mydata

# Clean up structural artifacts
docker volume rm mydata
```
* **Named Volumes** are fully isolated, sandboxed, and managed directly by the Docker Engine filesystem. Data explicitly outlives the container execution lifespan.

#### ✅ Use bind mounts
```bash
# Real-time synchronization of local project files (hot-reloading dev workflow)
docker run -v $(pwd):/app -p 3000:3000 node:20 node server.js

# Hardened read-only execution layer mount
docker run -v $(pwd)/config:/app/config:ro myapp
```
* **Bind Mounts** tether an absolute directory pathway directly from your host workstation inside the container environment. Changes instantly cross-propagate on both sides.

---

### 5. Networking & Multi-Container Systems

#### ✅ Create custom networks and connect containers
```bash
# Initialize custom isolated network bridging space
docker network create mynet

# Launch container isolated on custom network space
docker run -d --network mynet --name db postgres
docker run -d --network mynet --name web nginx

# Hot-attach a running container container into existing network paths
docker network connect mynet myapp
```
* **Service Discovery:** Containers attached to an identical custom user-defined network space resolve endpoints directly using **container names as hostnames** via Docker's internal DNS layer. Default `bridge` networks lack name resolution.

#### ✅ Write a docker-compose.yml for a multi-container app
```yaml
services:
  web:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgres://user:pass@db:5432/mydb
    depends_on:
      db:
        condition: service_healthy
    volumes:
      - ./src:/app/src

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: mydb
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  pgdata:
```

---

### 6. Advanced Compose & Optimization

#### ✅ Use environment variables and .env files in Compose
```ini
# .env file configuration example (Never check this file into source control!)
POSTGRES_PASSWORD=supersecret
APP_PORT=3000
```
```yaml
# docker-compose.yml structural embedding
services:
  web:
    image: myapp
    ports:
      - "${APP_PORT}:3000"
    env_file:
      - .env                  # Evaluates and injects all vars into container env
    environment:
      - NODE_ENV=production   # Inline override declaration (takes highest precedence)
```
* **Evaluation Precedence:** Shell Environment Variables -> Compose `environment:` Block -> `.env` configuration file.

#### ✅ Write a multi-stage Dockerfile
```dockerfile
# --- Stage 1: Continuous Integration / Compilation Stage ---
FROM node:20 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build          # Builds production bundle out into /app/dist

# --- Stage 2: Clean Production Runtime Stage ---
FROM node:20-alpine AS production
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
EXPOSE 3000
CMD ["node", "dist/server.js"]
```
* Multi-stage structures compile assets using extensive development tools, then copy only the resulting production assets into a clean, lightweight image. This cuts disk space and minimizes potential vulnerability attack vectors.

#### ✅ Use healthchecks and depends_on
```yaml
services:
  api:
    build: .
    depends_on:
      db:
        condition: service_healthy   # Defers api creation until db is fully listening

  db:
    image: postgres:16-alpine
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s              # Initialization grace period
```

---

## 🧠 Quick-Fire Concept Answers

#### 1. What is the difference between an image and a container?
An **image** is a static, read-only blueprint containing the application binaries, system snapshots, and filesystem layers sitting on a storage disk. A **container** is a live, dynamic runtime instantiation of that image running inside its own isolated process and kernel namespace. Think of an image as the *Class blueprint*, and a container as the *instantiated Object*.

#### 2. What happens to data inside a container when you remove it?
Any data written outside an explicitly defined volume or bind mount point is immediately **permanently lost**. Containers are completely ephemeral; the top-level writable container filesystem layer is destroyed instantly upon executing `docker rm`.

#### 3. How do two containers on the same custom network communicate?
They communicate over TCP/UDP channels using their **assigned container names as domain hostnames** (e.g., `http://db:5432`). Docker intercepts these requests through an embedded automatic DNS server layer to handle resolution dynamically.

#### 4. What does `docker compose down -v` do differently from `docker compose down`?
Standard `docker compose down` stops container operations and cleans up shared network bridges but preserves persistent volume stores. Appending the **`-v` flag completely purges all declared named storage volumes** alongside the containers—resetting the entire environment back to an absolute clean slate.

#### 5. Why are multi-stage builds useful?
They completely decouple compilation tools from runtime environments. By abandoning heavy developer tools, compilers, package managers, and source text in temporary intermediate layers, the final shipped image contains only production artifacts. This dramatically compresses image footprints (e.g., shrinking a node build from 1.5 GB down to less than 150 MB) and cuts down the application security vulnerability surface area.

#### 6. What is the difference between `COPY` and `ADD`?
`COPY` is an explicit, singular instruction that duplicates files or folders from the local system build directory context over to the image's filesystem. `ADD` handles basic file replication but includes complex magic capabilities: it automatically unpacks compressed local archive formats (like `.tar.gz`) directly on arrival and can fetch files directly from external remote URL addresses. **Best Practice:** Default to `COPY` for predictability; use `RUN curl` or `RUN wget` instead of `ADD` with URLs to prevent cache-busting behavior.

#### 7. What does `-p 8080:80` mean?
It establishes a port forwarding rule mapping **Port 8080 on the outer host OS interface** directly to **Port 80 sitting inside the container context**. External traffic routing to `localhost:8080` is routed seamlessly directly to the application layer binding container port 80.

#### 8. How do you check how much disk space Docker is using?
Run `docker system df` to see a structured breakdown of disk consumption across containers, cached layers, images, and volume blocks. To clear out inactive objects and reclaim maximum host memory space safely, run:
```bash
docker system prune -a --volumes
```