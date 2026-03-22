# DOCKER IMAGES AND CONTAINER LIFECYCLE

## Task 1: Docker Images
1. Pull Images
    ```bash
    docker pull nginx
    docker pull ubuntu
    docker pull alpine
    ```
2. List Images
    ```bash
    docker images
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/0f740b752428079be130bc744e67e2d7c27216fc/2026/day-30/Screenshots/Screenshot%20(77).png)

3. Compare **Ubuntu** vs **Alpine**
    - **Ubuntu: ~78.1 MB**. It’s a full-fledged base image with GNU utilities, bash, apt package manager, and a more complete userland. This makes it heavier but also more familiar for developers coming from traditional Linux environments.
        - Use Ubuntu when we want a familiar environment with more utilities pre-installed, even if it’s heavier.
    - **Alpine: ~8.44 MB**. It’s stripped down to the essentials, built on musl libc and busybox. That minimalism makes it ideal for microservices, CI/CD pipelines, and environments where size and speed matter.
        - Use Alpine when we want lean, fast containers and we're comfortable installing only what we need.
4. Inspect an Image
    ```bash
    docker inspect nginx
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/0f740b752428079be130bc744e67e2d7c27216fc/2026/day-30/Screenshots/Screenshot%20(78).png)

    Image ID & Digest
    - Unique SHA256 hash identifies the exact image version.
    - Digest (`nginx@sha256:...`) ensures immutability and reproducibility.

    Tags
    - `nginx:latest` - human-readable tag.
    - Always note that `latest` is just a tag, not necessarily the newest version.

    Created Timestamp
    - Shows when the image was built.

    Config Section
    - ExposedPorts - `80/tcp` (default HTTP port for Nginx).
    - Env Variables - `NGINX_VERSION`, `NJS_VERSION`, etc. (baked-in metadata).
    - Entrypoint - `/docker-entrypoint.sh` (script that runs first).
    - Cmd - `nginx -g 'daemon off;'` (keeps Nginx running in foreground).
    - Labels - Maintainer info.

    Architecture & OS
    - `amd64` + `linux` - tells us what platform the image is built for.

    Size
    - ~160 MB - total image size on disk.

    Storage Driver
    - `overlay2` - shows how Docker stores layers on your machine.

    RootFS Layers
    - List of SHA256 digests - each corresponds to a Dockerfile instruction.
    - Important for understanding caching and incremental builds.
5. Remove an Image
    ```bash
    docker rmi ubuntu
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/0f740b752428079be130bc744e67e2d7c27216fc/2026/day-30/Screenshots/Screenshot%20(79).png)

## Task 2: Image Layers
1. View History
    ```bash
    docker image history nginx
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/0f740b752428079be130bc744e67e2d7c27216fc/2026/day-30/Screenshots/Screenshot%20(80).png)

    Each line = a layer
    - Layers are created by Dockerfile instructions (`RUN`, `COPY`, `ENV`, `CMD`, etc.).
    - Immutable and cached - if a layer doesn’t change, Docker reuses it.

    Zero-byte layers (0B)
    - These are metadata-only changes (like `CMD`, `ENTRYPOINT`, `EXPOSE`, `ENV`, `LABEL`).
    - They don’t add files, just configuration.

    Large layers (e.g., 82 MB, 78 MB)
    - These come from `RUN` commands that install packages or copy large files.
    - Example: `RUN /bin/sh -c set -x && groupadd ...` added ~82 MB.
    - The base Debian layer (`debuerreotype`) added ~78 MB.

    COPY instructions
    - Small scripts (`docker-entrypoint.sh`, tuning scripts) are copied in as separate layers.
    - Each file added creates a new layer.

    Final CMD
    - `CMD ["nginx", "-g", "daemon off;"]` is the last instruction.
    - This doesn’t add size, but defines the default process when the container runs.

2. What are layers and why does Docker use them?<br>
**Ans.**
    - Layers are immutable building blocks.
    - Docker uses them for caching, if a layer hasn’t changed, it’s reused, speeding up builds and saving space.


## Task 3: Container Lifecycle
```bash
docker create --name test-nginx nginx
```
```bash
docker ps -a
```

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/0f740b752428079be130bc744e67e2d7c27216fc/2026/day-30/Screenshots/Screenshot%20(82).png)

- Creates a container object from the `nginx` image.
- It does not start the container — it just sets up the metadata (name, ID, filesystem, networking config).
- `docker run` = `docker create` + `docker start` in one step.
- Using `docker create` separately is useful when we want to configure or inspect a container before running it.

```bash
docker start test-nginx
```
```bash
docker ps
```

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/0f740b752428079be130bc744e67e2d7c27216fc/2026/day-30/Screenshots/Screenshot%20(83).png)

- It starts an existing container that was previously created with docker create.
- The container moves from Created ---> Running state.
- Unlike `docker run`, it does not create a new container - it only starts the one we already defined.

```bash
docker pause test-nginx
```
```bash
docker ps
```

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/0f740b752428079be130bc744e67e2d7c27216fc/2026/day-30/Screenshots/Screenshot%20(85).png)

- It **suspends all processes** inside the container by sending a `SIGSTOP`.
- The **container remains running** in Docker’s view, but its processes are frozen.
- Useful when you want to temporarily halt activity without stopping or killing the container.

```bash
docker unpause test-nginx
```
```bash
docker ps
```

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/0f740b752428079be130bc744e67e2d7c27216fc/2026/day-30/Screenshots/Screenshot%20(86).png)

- It resumes all processes inside a paused container.
- The container moves from Paused - Running state.
- Processes pick up exactly where they left off - nothing is lost.

```bash
docker stop test-nginx
```
```bash
docker ps -a
```

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/0f740b752428079be130bc744e67e2d7c27216fc/2026/day-30/Screenshots/Screenshot%20(87).png)

- Sends a SIGTERM signal to the main process inside the container.
- Gives the process a chance to shut down gracefully.
- If it doesn’t exit within the timeout (default 10s), Docker sends SIGKILL to force termination.
- The container moves from Running ---> Exited state.

```bash
docker restart test-nginx
```
```bash
docker ps
```

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/0f740b752428079be130bc744e67e2d7c27216fc/2026/day-30/Screenshots/Screenshot%20(88).png)

- It’s a shortcut that does stop + start in one command.
- The container moves from Exited ---> Running again.
- Useful when we want to quickly refresh a container without removing it.

```bash
docker kill test-nginx
```
```bash
docker ps -a
```

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/1742516a6f7c7696b259d2d3116d065fd772dda9/2026/day-30/Screenshots/Screenshot%20(90).png)

- Immediately sends a SIGKILL signal to the container’s main process.
- Unlike `docker stop`, it does not allow a graceful shutdown.
- The container moves from Running ---> Exited state instantly. 

```bash
docker rm test-nginx
```
```bash
docker ps -a
```

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/1742516a6f7c7696b259d2d3116d065fd772dda9/2026/day-30/Screenshots/Screenshot%20(91).png)

- Removes the container object from Docker.
- Deletes its metadata, filesystem layer (unless volumes are attached), and name reference.
- The container must be in Exited state before removal (or you can force with `-f`).
- After removal, the container no longer appears in ``docker ps -a.


## Task 4: Working with Running Containers
1. Run in Detached Mode
    ```bash
    docker run -d --name webserver nginx
    ```
2. Logs
    ```bash
    docker logs webserver
    docker logs -f webserver   # follow mode
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/1742516a6f7c7696b259d2d3116d065fd772dda9/2026/day-30/Screenshots/Screenshot%20(94).png)

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/1742516a6f7c7696b259d2d3116d065fd772dda9/2026/day-30/Screenshots/Screenshot%20(93).png)

3. Exec into Container
    ```bash
    docker exec -it webserver /bin/bash
    ```
    (or /bin/sh for Alpine)

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/1742516a6f7c7696b259d2d3116d065fd772dda9/2026/day-30/Screenshots/Screenshot%20(95).png)

4. Run Single Command
    ```bash
    docker exec webserver ls /usr/share/nginx/html
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/1742516a6f7c7696b259d2d3116d065fd772dda9/2026/day-30/Screenshots/Screenshot%20(96).png)

5. Inspect Container
    ```bash
    docker inspect webserver
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/1742516a6f7c7696b259d2d3116d065fd772dda9/2026/day-30/Screenshots/Screenshot%20(97).png)

## Task 5: Cleanup
1. Stop All Containers
    ```bash
    docker stop $(docker ps -q)
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/1742516a6f7c7696b259d2d3116d065fd772dda9/2026/day-30/Screenshots/Screenshot%20(98).png)

2. Remove All Stopped Containers
    ```bash
    docker rm $(docker ps -aq)
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/1742516a6f7c7696b259d2d3116d065fd772dda9/2026/day-30/Screenshots/Screenshot%20(99).png)

3. Remove Unused Images
    ```bash
    docker image prune -a
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/1742516a6f7c7696b259d2d3116d065fd772dda9/2026/day-30/Screenshots/Screenshot%20(100).png)

4. Check Disk Usage
    ```bash
    docker system df
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/1742516a6f7c7696b259d2d3116d065fd772dda9/2026/day-30/Screenshots/Screenshot%20(101).png)