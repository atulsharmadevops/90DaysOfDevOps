# MULTI-STAGE BUILDS & DOCKER HUB

##  Task 1: The Problem with Large Images
1. Write a Simple App<br>
    Create a folder for our project:
    ```bash
    mkdir hello-node
    cd hello-node
    ```
    Inside, create `hello-node/app.js`:
    ```javascript
    // app.js
    console.log("Hello World from Node.js!");
    ```
    Add a `hello-node/package.json` (minimal):
    ```json
    {
    "name": "hello-node",
    "version": "1.0.0",
    "main": "app.js"
    }
    ```
2. Create a Single-Stage Dockerfile<br>
    Create a file named `hello-node/Dockerfile.single`:

    ```dockerfile
    FROM node:18 #Base image includes Node.js runtime, npm, & full OS environment.

    WORKDIR /app #Working directory, all Subsequent commands execute relative to this directory.
    COPY package*.json ./ #Copies `package.json` into the container’s `/app` directory, so that dependency installation can be cached if the app code changes later.
    RUN npm install --production || true #Install only production dependencies (ignores dev dependencies). `|| true` ensures build doesn’t fail if there are warnings or minor issues.
    COPY . . #Copies rest of our application files into container’s `/app` directory including `app.js`

    CMD ["node", "app.js"] #Default command when the container starts. It runs our Node.js app by executing `node app.js`.
    ```
    
3. Build the Image
    ```bash
    docker build -t hello-node:single -f Dockerfile.single .
    ```
4. Run the Container<br>
    Verify it works:
    ```bash
    docker run --rm hello-node:single
    ```
    We should see:

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/958fac05b7a5b1c7d3168b9272bc6fc663332a14/2026/day-35/Screenshots/Screenshot%20(256).png)

5. Check the Image Size<br>
    List images:
    ```bash
    docker images hello-node:single
    ```
    We’ll see something like:
    
    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/958fac05b7a5b1c7d3168b9272bc6fc663332a14/2026/day-35/Screenshots/Screenshot%20(257).png)

    - Record the size (e.g., ~1.09GB).
    - This is our baseline to compare against the optimized multi-stage build later.

## Task 2: Multi-Stage Build
1. Create a Multi-Stage Dockerfile<br>
Make a new file called `Dockerfile.multistage`:
    ```dockerfile
    # Stage 1: Builder
    FROM node:18 AS builder #Use full Node.js 18 image, `AS builder` names this stage to reference it later.

    WORKDIR /app
    COPY package*.json ./
    RUN npm install --production
    COPY . .

    # Stage 2: Final minimal image
    FROM node:18-alpine #Switch to lightweight Alpine-based Node.js image

    WORKDIR /app
    COPY --from=builder /app . #Copies everything from builder stage into this final image, only built app and its dependencies are included - no compilers or caches.

    # Add non-root user for security
    RUN adduser -D nodeuser #Creates a non-root user (nodeuser) using Alpine’s adduser.
    USER nodeuser #Switch to that user so app doesn’t run with root privileges.

    CMD ["node", "app.js"]
    ```

    - `RUN adduser -D nodeuser` - creates a dedicated non‑root user (`nodeuser`) in the image. By default, containers run as `root`, which is risky because if the app is compromised, the attacker gets root privileges inside the container.

    - `USER nodeuser` - switches the execution context so our app runs as that non‑root user. This limits what the process can do, reducing the attack surface and aligning with production security standards.

2. Build the Multi-Stage Image
    ```bash
    docker build -t hello-node:multistage -f Dockerfile.multistage .
    ```
3. Run the Container<br>
    Verify it works:
    ```bash
    docker run --rm hello-node:multistage
    ```
    Output:

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/958fac05b7a5b1c7d3168b9272bc6fc663332a14/2026/day-35/Screenshots/Screenshot%20(259).png)

4. Check the Image Size<br>
    List images:
    ```bash
    docker images hello-node:multistage
    ```
    We’ll see something like:

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/958fac05b7a5b1c7d3168b9272bc6fc663332a14/2026/day-35/Screenshots/Screenshot%20(260).png)

5. Compare Sizes
    - Single-stage image (Task 1): ~1.09GB
    - Multi-stage image (Task 2): ~127MB

6. Why is the multi-stage image so much smaller?<br>
**Ans.** The multi‑stage image is smaller because the final stage only includes the lightweight runtime (like `node:18-alpine`) and our app files, while all build tools, caches, and extra OS packages from the builder stage are discarded. This separation strips away unnecessary layers, reducing size and improving security.

    We use multi-stage builds to strip away build-time dependencies, leaving only what’s needed to run the app. That’s why the image is smaller, faster, and more secure.

## Task 3: Push to Docker Hub
1. Create a Docker Hub Account
    - Go to [hub.docker.com]()
    - Sign up for a free account.
    - Choose a username — this will be part of our image name.
2. Log In from our Terminal
    ```bash
    docker login
    ```
    - Enter Docker Hub username and password.
    - If successful, we’ll see:
    
    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/958fac05b7a5b1c7d3168b9272bc6fc663332a14/2026/day-35/Screenshots/Screenshot%20(261).png)

3. Tag Our Image Properly<br>
    Suppose our multi-stage image is `hello-node:multistage`.<br>
    Tag it with Docker Hub username and a version:
    ```bash
    docker tag hello-node:multistage atulsharmadochub/hello-node:1.0
    ```
    - Format: `username/repo:tag`
    - Example: `atulsharmadochub/hello-node:1.0`
    - `latest` is the default tag if none is specified.
4. Push the Image to Docker Hub
    ```bash
    docker push atulsharmadochub/hello-node:1.0
    ```
    - This uploads our image layers to Docker Hub.
    - We’ll see progress bars as each layer is pushed.

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/958fac05b7a5b1c7d3168b9272bc6fc663332a14/2026/day-35/Screenshots/Screenshot%20(262).png)

5. Verify by Pulling on Another Machine<br>
    Option A: Remove locally and re-pull
    ```bash
    docker rmi atulsharmadochub/hello-node:1.0
    ```
    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/958fac05b7a5b1c7d3168b9272bc6fc663332a14/2026/day-35/Screenshots/Screenshot%20(263).png)

    ```bash
    docker pull atulsharmadochub/hello-node:1.0
    ```
    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/958fac05b7a5b1c7d3168b9272bc6fc663332a14/2026/day-35/Screenshots/Screenshot%20(264).png)

    Option B: Pull on a different machine
    ```bash
    docker pull atulsharmadochub/hello-node:1.0
    docker run --rm atulsharmadochub/hello-node:1.0
    ```
    We should see:
    ```Code
    Hello World from Node.js!
    ```

## Task 4: Docker Hub Repository
1. Check Our Pushed Image
    - Log in to [Docker Hub](https://hub.docker.com).
    - Navigate to our profile - **Repositories**.
    - We should see the repo we pushed earlier (e.g., `atulsharmadochub/hello-node`).

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/958fac05b7a5b1c7d3168b9272bc6fc663332a14/2026/day-35/Screenshots/Screenshot%20(265).png)

2. Add a Description
    - Click on the repository.
    - In the **Settings** or **Overview** tab, add a description like:<br>
    “Optimized Node.js Hello World app using multi-stage builds. Demonstrates reduced image size.”

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/958fac05b7a5b1c7d3168b9272bc6fc663332a14/2026/day-35/Screenshots/Screenshot%20(267).png)

3. Explore the Tags Tab
    - Go to the Tags section of our repository.
    - We’ll see tags such as:
        - `1.0` (the version we pushed)
        - `latest` (default if we didn’t specify a tag)
    - Tags are how we version images — we can push multiple versions (`1.1`, `2.0`) without overwriting older ones.

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/e71ac5aa01c3f8df4ca9f73984f7564a0e67ba22/2026/day-35/Screenshots/Screenshot%20(271).png)

4. Pull a Specific Tag vs Latest<br>
    Pull a specific tag:
    ```bash
    docker pull atulsharmadochub/hello-node:1.0
    ```
    - Always fetches that exact version.
    
    Pull `latest`:
    ```bash
    docker pull atulsharmadochub/hello-node:latest
    ```
    - Fetches whatever image is currently tagged as `latest`. This may change over time.

- Key Difference:
    - `:1.0` is immutable — we always get the same image.
    - `:latest` is mutable — it depends on what we last pushed as latest.

## Task 5: Image Best Practices
1. Use a Minimal Base Image<br>
    Instead of `node:18` (Debian-based, ~950MB), switch to `node:18-alpine` (~60MB).
    This alone cuts hundreds of MB.
    ```dockerfile
    FROM node:18-alpine
    ```
2. Don’t Run as Root<br>
    Add a non-root user for security:
    ```dockerfile
    RUN adduser -D nodeuser
    USER nodeuser
    ```
    - This prevents privilege escalation risks inside containers.

3. Combine RUN Commands<br>
    Each `RUN` creates a new layer. Combine them to reduce layers:
    ```dockerfile
    RUN apk add --no-cache curl && \
        adduser -D nodeuser
    ```
4. Use Specific Tags<br>
    Avoid `latest` — it changes over time. Pin your base image:
    ```dockerfile
    FROM node:18-alpine
    ```
    - This ensures reproducibility.

5. Example Optimized Dockerfile
    ```dockerfile
    # Dockerfile.bestpractice
    FROM node:18-alpine

    WORKDIR /app
    COPY package*.json ./

    RUN npm install --production
        
    RUN adduser -D nodeuser

    COPY . .
    USER nodeuser

    CMD ["node", "app.js"]
    ```
6. Build and Check Size
    ```bash
    docker build -t hello-node:bestpractice -f Dockerfile.bestpractice .
    docker images hello-node:bestpractice
    ```
    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/958fac05b7a5b1c7d3168b9272bc6fc663332a14/2026/day-35/Screenshots/Screenshot%20(268).png)

    - We’ll likely see:

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/958fac05b7a5b1c7d3168b9272bc6fc663332a14/2026/day-35/Screenshots/Screenshot%20(270).png)

    - **Before (single-stage, Debian)**: ~1.09GB
    - **After (multi-stage, alpine, non-root, fewer layers)**: ~127MB