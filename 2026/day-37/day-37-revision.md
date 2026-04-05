# Day 37 - Docker Revision & Self-Assessment

---

## Self-Assessment Checklist

### ✅ Run a container from Docker Hub (interactive + detached)

```bash
# Detached — runs in the background, returns container ID
docker run -d -p 8080:80 --name mynginx nginx

# Interactive — drops you into a shell inside the container
docker run -it ubuntu bash
```

**Key flags:** `-d` = detached, `-it` = interactive + pseudo-TTY, `-p host:container` = port mapping, `--name` = assign a name instead of a random one.

---

### ✅ List, stop, remove containers and images

```bash
# Containers
docker ps           # running only
docker ps -a        # all (including stopped)
docker stop mynginx
docker rm mynginx
docker rm -f mynginx   # force-remove without stopping first

# Images
docker images          # or: docker image ls
docker rmi nginx:alpine
docker image prune -a  # remove all unused images
```

---

### ✅ Explain image layers and how caching works

A Docker image is a **stack of read-only layers**. Each instruction in a Dockerfile (`RUN`, `COPY`, `ADD`) creates one new layer on top of the previous. When you run a container, Docker adds a thin **writable layer** on top - all changes during the container's life live there and are discarded when the container is removed.

**Caching:** Docker caches each layer by its content hash. When you rebuild:
- If nothing changed for a layer, Docker reuses the cache - instant.
- The moment any layer changes, **all subsequent layers are invalidated** and rebuilt from scratch.

**Practical implication — order matters:**

```dockerfile
# BAD — code changes bust the dependency cache every time
COPY . .
RUN npm install

# GOOD — dependencies only reinstall when package.json changes
COPY package*.json ./
RUN npm install
COPY . .
```

---

### ✅ Write a Dockerfile from scratch

```dockerfile
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install --production

COPY . .

EXPOSE 3000

CMD ["node", "server.js"]
```

- `FROM` - sets the base image (always first)
- `WORKDIR` - sets working directory; creates it if it doesn't exist
- `COPY` - brings files from your build context into the image
- `RUN` - executes a command and commits the result as a new layer
- `EXPOSE` - documents the port (informational; doesn't publish it)
- `CMD` - the default command to run when the container starts

---

### ✅ Explain CMD vs ENTRYPOINT

Both define what runs when a container starts - the difference is **rigidity**.

| | `CMD` | `ENTRYPOINT` |
|---|---|---|
| Purpose | Default command/args | Locked-in executable |
| Overridable? | Yes — anything after `docker run image <here>` replaces it | No — args append to it |
| Common use | Default flags or commands | The binary itself |

```dockerfile
# ENTRYPOINT locks the executable; CMD provides overridable defaults
ENTRYPOINT ["python", "app.py"]
CMD ["--port", "8000"]

# docker run myapp                    → python app.py --port 8000
# docker run myapp --port 9000        → python app.py --port 9000
# docker run myapp bash               → python app.py bash  (probably wrong!)
```

Use `CMD` alone for simple containers. Use `ENTRYPOINT` + `CMD` together when the container should always run one specific program.

---

### ✅ Build and tag a custom image

```bash
# Build with a tag
docker build -t myapp:1.0 .

# Tag an existing image (e.g. for Docker Hub)
docker tag myapp:1.0 atulsharmadochub/myapp:1.0

# Tag as latest too
docker tag myapp:1.0 atulsharmadochub/myapp:latest
```

Format: `registry/username/repo:tag` - when registry is omitted, Docker Hub is assumed.

---

### ✅ Create and use named volumes

```bash
# Create explicitly
docker volume create mydata

# Or let Docker create it on first use
docker run -d -v mydata:/app/data --name myapp postgres

# Inspect where it lives on the host
docker volume inspect mydata

# Clean up
docker volume rm mydata
```

Named volumes **persist beyond the container's life** - data survives `docker rm`. Ideal for databases, uploads, anything stateful.

---

### ✅ Use bind mounts

```bash
# Mount current directory into the container (great for local dev)
docker run -v $(pwd):/app -p 3000:3000 node:20 node server.js

# Read-only bind mount
docker run -v $(pwd)/config:/app/config:ro myapp
```

Bind mounts link a **host path** directly into the container. Changes on either side are reflected immediately - perfect for hot-reload dev workflows. Unlike named volumes, the host path must exist.

---

### ✅ Create custom networks and connect containers

```bash
# Create a network
docker network create mynet

# Run containers on that network
docker run -d --network mynet --name db postgres
docker run -d --network mynet --name web nginx

# Connect an already-running container
docker network connect mynet myapp

# Verify
docker network inspect mynet
```

Containers on the same custom network can reach each other **by container name** as the hostname - no IPs needed. The default `bridge` network doesn't support this; always use a named network for multi-container setups.

---

### ✅ Write a docker-compose.yml for a multi-container app

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

### ✅ Use environment variables and .env files in Compose

```bash
# .env file (auto-loaded by Compose — never commit this)
POSTGRES_PASSWORD=supersecret
APP_PORT=3000
```

```yaml
# docker-compose.yml
services:
  web:
    image: myapp
    ports:
      - "${APP_PORT}:3000"
    env_file:
      - .env                  # injects all vars from file into container
    environment:
      - NODE_ENV=production   # inline vars take precedence
```

**Priority (highest → lowest):** shell env → `environment:` block → `.env` file.  
Use `.env` for secrets/config, `environment:` for non-sensitive defaults, and always add `.env` to `.gitignore`.

---

### ✅ Write a multi-stage Dockerfile

```dockerfile
# Stage 1 — builder (large, has dev tools)
FROM node:20 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build          # produces /app/dist

# Stage 2 — production (lean, only what runs)
FROM node:20-alpine AS production
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

The final image contains **only Stage 2** - the builder tools, source files, and dev dependencies are discarded. This is the standard way to ship small, secure production images.

---

### ✅ Push an image to Docker Hub

```bash
# 1. Log in (prompts for password)
docker login

# 2. Tag with your Docker Hub username
docker tag myapp:1.0 yourusername/myapp:1.0

# 3. Push
docker push yourusername/myapp:1.0

# 4. Pull anywhere
docker pull yourusername/myapp:1.0
```

---

### ✅ Use healthchecks and depends_on

```dockerfile
# In Dockerfile
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1
```

```yaml
# In docker-compose.yml
services:
  api:
    build: .
    depends_on:
      db:
        condition: service_healthy   # waits until db passes healthcheck

  db:
    image: postgres:16-alpine
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s              # grace period on first start
```

`depends_on` without `condition` only waits for the container to **start**, not be **ready**. Always pair it with a healthcheck for databases and slow-starting services.

---

## Quick-Fire Answers

**1. What is the difference between an image and a container?**

An **image** is a read-only blueprint - a snapshot of a filesystem and metadata baked into layers. It sits on disk and doesn't run anything.

A **container** is a live, running instance of an image - it has its own writable layer, its own process space, and its own network interface. The image is the class; the container is the object.

---

**2. What happens to data inside a container when you remove it?**

It's gone.

Containers have a writable layer that is destroyed with `docker rm`. Any data written inside the container (logs, uploads, database files) is lost unless you explicitly persisted it to a **named volume** or **bind mount** before removal.

---

**3. How do two containers on the same custom network communicate?**

They use each other's **container name as the hostname**.

Docker's embedded DNS resolver handles the lookup automatically. If a container named `db` is on the `mynet` network, another container on the same network can reach it at `db:5432` with no extra configuration. This only works on user-defined networks - the default `bridge` network does not support name resolution.

---

**4. What does `docker compose down -v` do differently from `docker compose down`?**

`docker compose down` stops and removes containers and the networks Compose created, but **leaves named volumes intact**.

`docker compose down -v` does all of that and **also deletes named volumes** declared in the compose file. Use `-v` when you want a clean slate (e.g. resetting a dev database). Avoid it in production.

---

**5. Why are multi-stage builds useful?**

They let us use a fat image with all our build tools (compilers, bundlers, test runners) for the build stage, then copy only the compiled output into a minimal runtime image.

The final shipped image has no build toolchain, no source code, no dev dependencies - smaller attack surface, smaller size, faster pulls. A Node app might go from 1.2 GB (with build tools) to under 150 MB.

---

**6. What is the difference between `COPY` and `ADD`?**

`COPY` does exactly one thing: copies files from the build context into the image.

`ADD` does that plus two extras - it auto-extracts `.tar` archives and can fetch from remote URLs.

In practice, **always use `COPY`** unless we specifically need the extraction behaviour. `ADD` with a URL bypasses the layer cache and introduces unpredictability; use `RUN curl` instead for remote files.

---

**7. What does `-p 8080:80` mean?**

It maps **port 8080 on the host** to **port 80 inside the container**.

Format is always `host:container`. Traffic hitting `localhost:8080` on our machine is forwarded to port 80 in the container. The container itself still thinks it's listening on 80 - only the host-side mapping changes.

---

**8. How do you check how much disk space Docker is using?**

```bash
docker system df
```

This breaks down usage by images, containers, volumes, and build cache — with a "reclaimable" column showing what's safe to delete.

For a full cleanup of everything unused:

```bash
docker system prune -a --volumes
```

---