# 🐳 Docker Cheat Sheet

---

## Container Commands

| Command | Description |
|--------|-------------|
| `docker run -d -p 8080:80 --name myapp nginx` | Run container detached, map ports, assign name |
| `docker run -it ubuntu bash` | Run container interactively with a shell |
| `docker ps` | List running containers |
| `docker ps -a` | List all containers (including stopped) |
| `docker stop myapp` | Gracefully stop a running container |
| `docker stop $(docker ps -q)` | Stop all running containers |
| `docker rm myapp` | Remove a stopped container |
| `docker rm -f myapp` | Force remove a running container |
| `docker exec -it myapp bash` | Open shell inside a running container |
| `docker exec myapp env` | Run a one-off command inside a container |
| `docker logs myapp` | Show container logs |
| `docker logs -f myapp` | Stream container logs (follow mode) |
| `docker logs --tail 100 myapp` | Show last 100 log lines |
| `docker inspect myapp` | Show full container metadata as JSON |
| `docker stats` | Live resource usage for all containers |
| `docker cp myapp:/etc/nginx/nginx.conf ./` | Copy file from container to host |

---

## Image Commands

| Command | Description |
|--------|-------------|
| `docker build -t myapp:1.0 .` | Build image from Dockerfile in current directory |
| `docker build -f Dockerfile.prod -t myapp:prod .` | Build using a specific Dockerfile |
| `docker pull nginx:alpine` | Pull image from Docker Hub |
| `docker push myrepo/myapp:1.0` | Push image to a registry |
| `docker tag myapp:latest myrepo/myapp:1.0` | Tag an image for a registry |
| `docker images` | List all local images |
| `docker image ls` | Same as above (modern syntax) |
| `docker rmi myapp:1.0` | Remove a local image |
| `docker image inspect nginx` | Show image metadata and layers |
| `docker history nginx` | Show layer history of an image |
| `docker save myapp:1.0 (pipe) gzip > myapp.tar.gz` | Export image to a tar archive |
| `docker load < myapp.tar.gz` | Import image from a tar archive |

---

## Volume Commands

| Command | Description |
|--------|-------------|
| `docker volume create mydata` | Create a named volume |
| `docker run -v mydata:/app/data nginx` | Mount named volume into a container |
| `docker run -v $(pwd):/app nginx` | Mount current directory (bind mount) |
| `docker volume ls` | List all volumes |
| `docker volume inspect mydata` | Show volume details and mount path |
| `docker volume rm mydata` | Remove a volume |
| `docker volume prune` | Remove all unused volumes |

---

## Network Commands

| Command | Description |
|--------|-------------|
| `docker network create mynet` | Create a custom bridge network |
| `docker network create --driver overlay mynet` | Create an overlay network (Swarm) |
| `docker run --network mynet nginx` | Run container on a specific network |
| `docker network ls` | List all networks |
| `docker network inspect mynet` | Show network details and connected containers |
| `docker network connect mynet myapp` | Connect a running container to a network |
| `docker network disconnect mynet myapp` | Disconnect container from a network |
| `docker network rm mynet` | Remove a network |

---

## Compose Commands

| Command | Description |
|--------|-------------|
| `docker compose up -d` | Start all services in detached mode |
| `docker compose up --build` | Rebuild images before starting |
| `docker compose down` | Stop and remove containers and networks |
| `docker compose down -v` | Also remove named volumes |
| `docker compose ps` | List service containers and their status |
| `docker compose logs -f` | Stream logs from all services |
| `docker compose logs -f web` | Stream logs from a specific service |
| `docker compose exec web bash` | Open shell inside a running service |
| `docker compose build` | Build or rebuild service images |
| `docker compose pull` | Pull latest images for all services |
| `docker compose restart web` | Restart a specific service |
| `docker compose run --rm web python manage.py migrate` | Run a one-off command in a service |
| `docker compose config` | Validate and print the resolved compose config |

---

## Cleanup Commands

| Command | Description |
|--------|-------------|
| `docker container prune` | Remove all stopped containers |
| `docker image prune` | Remove dangling (untagged) images |
| `docker image prune -a` | Remove all unused images |
| `docker volume prune` | Remove all unused volumes |
| `docker network prune` | Remove all unused networks |
| `docker system prune` | Remove stopped containers, unused networks, dangling images |
| `docker system prune -a --volumes` | Full nuke — removes everything unused |
| `docker system df` | Show disk usage by images, containers, volumes |

---

## Dockerfile Instructions

| Instruction | Description |
|------------|-------------|
| `FROM node:20-alpine` | Base image — always the first instruction |
| `RUN apt-get update && apt-get install -y curl` | Execute commands and commit result as a new layer |
| `COPY ./src /app/src` | Copy files from build context into the image |
| `ADD archive.tar.gz /app/` | Like COPY, but auto-extracts archives and supports URLs |
| `WORKDIR /app` | Set working directory for subsequent instructions |
| `ENV NODE_ENV=production` | Set environment variables baked into the image |
| `ARG VERSION=1.0` | Build-time variable passed via `--build-arg` |
| `EXPOSE 3000` | Document the port the container listens on (informational) |
| `VOLUME ["/app/data"]` | Declare a mount point for external volumes |
| `USER node` | Switch to a non-root user for security |
| `CMD ["node", "server.js"]` | Default command — overridable at `docker run` |
| `ENTRYPOINT ["docker-entrypoint.sh"]` | Fixed entrypoint — `CMD` becomes its default args |
| `HEALTHCHECK CMD curl -f http://localhost/ \|\| exit 1` | Define a health check for the container |
| `ONBUILD COPY . /app` | Trigger instruction when this image is used as a base |

---

> **Tip:** `CMD` sets defaults you can override; `ENTRYPOINT` locks the executable.  
> Use both together: `ENTRYPOINT` for the binary, `CMD` for default flags.