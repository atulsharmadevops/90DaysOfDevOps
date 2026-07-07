# 🐳 Ultimate Docker Cheat Sheet

A comprehensive, production-ready reference guide for Docker, Docker Compose, and Dockerfiles.

## 📌 Table of Contents
* [Container Management](#-container-management)
* [Image Management](#-image-management)
* [Volume Management](#-volume-management)
* [Network Management](#-network-management)
* [Docker Compose](#-docker-compose)
* [System Cleanup](#-system-cleanup)
* [Dockerfile Reference](#-dockerfile-reference)

---

## 📦 Container Management

| Command | Description |
| :--- | :--- |
| `docker run -d -p 8080:80 --name myapp nginx` | Run container in background (detached), map ports, assign name |
| `docker run -it ubuntu bash` | Run a container interactively with a TTY terminal and shell |
| `docker ps` | List all currently running containers |
| `docker ps -a` | List all containers (including stopped/exited) |
| `docker stop myapp` | Gracefully stop a running container (`SIGTERM`) |
| `docker stop $(docker ps -q)` | Stop all active running containers |
| `docker rm myapp` | Remove a stopped container |
| `docker rm -f myapp` | Force-remove a running container (`SIGKILL`) |
| `docker exec -it myapp bash` | Open an interactive shell inside a running container |
| `docker exec myapp env` | Run a one-off command inside an active container |
| `docker logs myapp` | Fetch and view container logs |
| `docker logs -f myapp` | Stream live container logs (follow mode) |
| `docker logs --tail 100 myapp` | Display only the last 100 log lines |
| `docker inspect myapp` | Return low-level configuration metadata as JSON |
| `docker stats` | Stream live CPU, memory, and network resource usage metrics |
| `docker cp myapp:/etc/nginx/nginx.conf ./` | Copy files/folders between container and local host filesystem |

---

## 🖼️ Image Management

| Command | Description |
| :--- | :--- |
| `docker build -t myapp:1.0 .` | Build an image from a Dockerfile in the current directory |
| `docker build -f Dockerfile.prod -t myapp:prod .` | Build an image using a specifically named Dockerfile |
| `docker pull nginx:alpine` | Download an image from Docker Hub (or configured registry) |
| `docker push myrepo/myapp:1.0` | Upload a tagged image to a remote registry |
| `docker tag myapp:latest myrepo/myapp:1.0` | Create a target tag that references a source image |
| `docker images` \| `docker image ls` | List all locally cached Docker images |
| `docker rmi myapp:1.0` | Delete a local image |
| `docker image inspect nginx` | Display detailed configurations and layer metadata of an image |
| `docker history nginx` | Show the history and intermediate layers of an image |
| `docker save myapp:1.0 \| gzip > myapp.tar.gz` | Compress and export an image into a tarball archive |
| `docker load < myapp.tar.gz` | Load an image from a compressed tarball archive |

---

## 💾 Volume Management

| Command | Description |
| :--- | :--- |
| `docker volume create mydata` | Provision a persistent named volume |
| `docker run -v mydata:/app/data nginx` | Mount a named volume into a container path |
| `docker run -v $(pwd):/app nginx` | Bind-mount the current working directory into a container |
| `docker volume ls` | List all local volumes managed by Docker |
| `docker volume inspect mydata` | Show specific volume data, including its host storage path |
| `docker volume rm mydata` | Delete a specific unused volume |
| `docker volume prune` | Wipe out all unused local volumes |

---

## 🌐 Network Management

| Command | Description |
| :--- | :--- |
| `docker network create mynet` | Create a custom user-defined bridge network |
| `docker network create --driver overlay mynet` | Spin up an overlay network for multi-host cluster routing (Swarm) |
| `docker run --network mynet nginx` | Launch a container attached to a specific network |
| `docker network ls` | List all active networks |
| `docker network inspect mynet` | Review settings and view all containers connected to the network |
| `docker network connect mynet myapp` | Hot-connect a running container to an existing network |
| `docker network disconnect mynet myapp` | Safe disconnect a container from a network |
| `docker network rm mynet` | Delete a network |

---

## 🛠️ Docker Compose

| Command | Description |
| :--- | :--- |
| `docker compose up -d` | Build, (re)create, start, and attach to services in the background |
| `docker compose up --build` | Force a rebuild of service images before spinning them up |
| `docker compose down` | Stop containers and remove containers, networks, and images created by `up` |
| `docker compose down -v` | Tear down the stack and also destroy declared named volumes |
| `docker compose ps` | List status of all containers managed by the current compose file |
| `docker compose logs -f` | Tail combined engine logs across all services simultaneously |
| `docker compose logs -f web` | View streaming logs for a specific service target |
| `docker compose exec web bash` | Open a shell inside a running service container |
| `docker compose build` | Build or rebuild services defined in the configuration file |
| `docker compose pull` | Fetch the latest remote image tags for your services |
| `docker compose restart web` | Restart a specific service without tearing down the configuration |
| `docker compose run --rm web python manage.py migrate` | Run a one-off command inside a service and remove container after exit |
| `docker compose config` | Validate, parse, and render the resolved Compose configuration |

---

## 🧹 System Cleanup

| Command | Description |
| :--- | :--- |
| `docker container prune` | Purge all stopped containers |
| `docker image prune` | Remove dangling (untagged) images only |
| `docker image prune -a` | Remove all images not explicitly associated with a running container |
| `docker volume prune` | Delete all unused volumes |
| `docker network prune` | Delete all unused networks |
| `docker system prune` | Clear out stopped containers, unused networks, and dangling images |
| `docker system prune -a --volumes` | **The Nuclear Option:** Erases *everything* unused (containers, images, volumes, networks) |
| `docker system df` | Show detailed breakdowns of space consumed by Docker objects |

---

## 📄 Dockerfile Reference

| Instruction | Description |
| :--- | :--- |
| `FROM node:20-alpine` | Set the base image to build upon. *Must be the first non-comment entry.* |
| `RUN apt-get update && apt-get install -y curl` | Run shell commands in a new layer and commit results. |
| `COPY ./src /app/src` | Copy files/directories from the local host build context into the image layer. |
| `ADD archive.tar.gz /app/` | Like `COPY`, but automatically unpacks local tar archives and fetches remote URLs. |
| `WORKDIR /app` | Set the active working directory context for all subsequent instructions. |
| `ENV NODE_ENV=production` | Define environment variables that persist inside the runtime container environment. |
| `ARG VERSION=1.0` | Declare variables that users can pass at build-time using `--build-arg`. |
| `EXPOSE 3000` | Documentation directive exposing internal ports (purely informational). |
| `VOLUME ["/app/data"]` | Create a managed mount point for mapping externally persistent storage. |
| `USER node` | Switch execution privileges to a non-root user account for optimal container security. |
| `CMD ["node", "server.js"]` | Provide default commands or arguments for an executing container. *Overridable.* |
| `ENTRYPOINT ["docker-entrypoint.sh"]` | Configure a container to run as an executable. *Difficult to override.* |
| `HEALTHCHECK CMD curl -f http://localhost/ \|\| exit 1` | Set up a diagnostic rule to verify if the internal application process is healthy. |
| `ONBUILD COPY . /app` | Register a trigger execution routine when this image is later consumed as a `FROM` base. |

---

> 💡 **Best Practice Tip:** Use `ENTRYPOINT` for the core executable of your container, and use `CMD` to pass default flags or arguments to that executable. If a user overrides arguments at runtime via `docker run`, it will seamlessly substitute your `CMD` defaults while preserving your core `ENTRYPOINT`.