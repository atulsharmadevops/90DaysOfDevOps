# GitHub Actions – Docker Build & Push in GitHub Actions
## Task 1: Prepare
### Step 1: Use Our App/Dockerfile
- If we already have the app we Dockerized on Day 36, copy its Dockerfile into our github-actions-practice repo.
- If not, create a minimal one. For example:
    ```dockerfile
    # Minimal Dockerfile example
    FROM python:3.9-slim

    WORKDIR /app

    COPY requirements.txt .
    RUN pip install -r requirements.txt

    COPY . /app

    CMD ["python", "app.py"]
    ```
- Create `requirements.txt`
  ```code
  requests
  flask
  ```
- Create  `app.py`
  ```python
  import requests
  from flask import Flask

  app = Flask(__name__)

  @app.route("/")
  def hello():
      # Just a simple demo using requests
      r = requests.get("https://api.github.com")
      return f"Hello from Docker! GitHub API status: {r.status_code}"

  if __name__ == "__main__":
      app.run(host="0.0.0.0", port=8080)
  ```
    - This ensures we have something to build and push.
    - Place the Dockerfile at the root of our repo.
        ```bash
        git add .
        git commit -m "Add Dockerfile for CI/CD pipeline"
        git push origin main
        ```
### Step 2: Set GitHub Secrets
We need two secrets in our repo settings:
1. DOCKER_USERNAME
    - Go to our GitHub repo **--> Settings --> Secrets and variables --> Actions --> New repository secret**.
    - Name: `DOCKER_USERNAME`
    - Value: Docker Hub username.
2. DOCKER_TOKEN
    - In Docker Hub, go to **Account Settings --> Personal access tokens --> Generate new token**.
    - Copy the generated token.
    - Back in GitHub, create another secret:
        - Name: `DOCKER_TOKEN`
        - Value: paste the token.

### Step 3: Verify Secrets
- In our workflow file later, we’ll reference them as:
  ```yaml
  username: ${{ secrets.DOCKER_USERNAME }}
  password: ${{ secrets.DOCKER_TOKEN }}
  ```
- To confirm they’re set correctly, check under **Settings --> Secrets and variables --> Actions**. Both should appear in the list.

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/cbd74e821da54d9581867e5fce6a8ff18dac1cf7/2026/day-45/Screenshots/Screenshot%20(424).png)

### Step 4: Sanity Check
- Run a local build to ensure our Dockerfile works before letting GitHub Actions handle it:
```bash
docker build -t test-image .
docker run -p 8080:8080 test-image
```
If it runs locally, we’re ready for Task 2.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/cbd74e821da54d9581867e5fce6a8ff18dac1cf7/2026/day-45/Screenshots/Screenshot%20(426).png)

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/cbd74e821da54d9581867e5fce6a8ff18dac1cf7/2026/day-45/Screenshots/Screenshot%20(427).png)

## Task 2: Build the Docker Image in CI
### Step 1: Create Workflow File
Inside our repo, create the file `.github/workflows/docker-publish.yml`
```yaml
name: Docker Publish

on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      # Step 1: Checkout code
      - name: Checkout
        uses: actions/checkout@v4

      # Step 2: Build Docker image
      - name: Build Docker image
        run: |
          docker build -t my-image:latest .
```
Save the file, commit, and push:

```bash
git add .github/workflows/docker-publish.yml
git commit -m "Add Docker build workflow"
git push origin main
```
### Step 2: Verify Build Logs
- Go to our repo’s **Actions** tab.
- Select the **Docker Publish** workflow run.
- Open the **Build Docker image** step.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/cbd74e821da54d9581867e5fce6a8ff18dac1cf7/2026/day-45/Screenshots/Screenshot%20(430).png)

- Check logs: we should see Docker pulling base images, installing dependencies, and finishing with `naming to docker.io/library/my-image:latest done`.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/cbd74e821da54d9581867e5fce6a8ff18dac1cf7/2026/day-45/Screenshots/Screenshot%20(428).png)

## Task 3: Push to Docker Hub
Create a pipeline that handles login, commit SHA tagging, and pushing images: `.github/workflows/docker-push-tags.yml`
```yaml
name: Docker Push with Tags

on:
  push:
    branches:
      - main

jobs:
  docker:
    runs-on: ubuntu-latest
    steps:
      # Step 1: Checkout code
      - name: Checkout
        uses: actions/checkout@v4

      # Step 2: Log in to Docker Hub
      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_TOKEN }}

      # Step 3: Generate short commit SHA
      - name: Set short SHA
        run: echo "SHORT_SHA=${GITHUB_SHA::7}" >> $GITHUB_ENV

      # Step 4: Build and push image
      - name: Build and Push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ${{ secrets.DOCKER_USERNAME }}/github-actions-practice:latest
            ${{ secrets.DOCKER_USERNAME }}/github-actions-practice:sha-${{ env.SHORT_SHA }}
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/cbd74e821da54d9581867e5fce6a8ff18dac1cf7/2026/day-45/Screenshots/Screenshot%20(431).png)

### Verify on Docker Hub
- Confirm that both tags appear in Docker Hub repository.
  - Go to Docker Hub --> our repo

    ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/cbd74e821da54d9581867e5fce6a8ff18dac1cf7/2026/day-45/Screenshots/Screenshot%20(432).png)

  - Check for `latest` tag
  - Check for `sha-xxxxxxx` tag

    ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/cbd74e821da54d9581867e5fce6a8ff18dac1cf7/2026/day-45/Screenshots/Screenshot%20(433).png)

  - Ensure image size and build time look correct

## Task 4: Only Push on Main
### Step 1: Add Branch Condition
Create a workflow such that the push step runs only if the branch is `main`. We can use `if:` in our YAML: `.github/workflows/docker-conditional-push.yml`
```yaml
name: Docker Conditional Push

on:
  push:
    branches:
      - '**'

jobs:
  docker:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Log in to Docker Hub
        if: github.ref == 'refs/heads/main'
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_TOKEN }}

      - name: Set short SHA
        run: echo "SHORT_SHA=${GITHUB_SHA::7}" >> $GITHUB_ENV

      - name: Build Docker image
        run: docker build -t my-image:latest .

      - name: Push Docker image
        if: github.ref == 'refs/heads/main'
        run: |
          docker tag my-image:latest ${{ secrets.DOCKER_USERNAME }}/github-actions-practice:latest
          docker tag my-image:latest ${{ secrets.DOCKER_USERNAME }}/github-actions-practice:sha-${{ env.SHORT_SHA }}
          docker push ${{ secrets.DOCKER_USERNAME }}/github-actions-practice:latest
          docker push ${{ secrets.DOCKER_USERNAME }}/github-actions-practice:sha-${{ env.SHORT_SHA }}
```
### Step 2: Test the Condition
- Push a commit to a feature branch:
  ```bash
  git checkout -b feature/test-branch
  git push origin feature/test-branch
  ```
- Go to **Actions** tab:
  - Workflow runs.
  
    ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/cbd74e821da54d9581867e5fce6a8ff18dac1cf7/2026/day-45/Screenshots/Screenshot%20(435).png)

  - Build step executes.
  - Push steps are **skipped** (we’ll see “Job skipped due to condition”).

    ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/cbd74e821da54d9581867e5fce6a8ff18dac1cf7/2026/day-45/Screenshots/Screenshot%20(436).png)

- Push to `main`:
  - Workflow runs.

    ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/cbd74e821da54d9581867e5fce6a8ff18dac1cf7/2026/day-45/Screenshots/Screenshot%20(441).png)

  - Both build and push steps execute.

    ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/cbd74e821da54d9581867e5fce6a8ff18dac1cf7/2026/day-45/Screenshots/Screenshot%20(442).png)

  - Images appear in Docker Hub.

    ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/cbd74e821da54d9581867e5fce6a8ff18dac1cf7/2026/day-45/Screenshots/Screenshot%20(443).png)

    ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/cbd74e821da54d9581867e5fce6a8ff18dac1cf7/2026/day-45/Screenshots/Screenshot%20(444).png)

## Task 5: Add a Status Badge
### Step 1: Get Badge URL
- Go to our GitHub repo --> **Actions** tab.
- Select our workflow (`docker-publish.yml`).
- Click the **... menu** (top right) --> **Create status badge**.

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/cbd74e821da54d9581867e5fce6a8ff18dac1cf7/2026/day-45/Screenshots/Screenshot%20(446).png)

- Copy the Markdown snippet. It looks like:

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/cbd74e821da54d9581867e5fce6a8ff18dac1cf7/2026/day-45/Screenshots/Screenshot%20(447).png)

### Step 2: Add to README
- Open our `README.md`.
- Paste the badge snippet at the top (or in a “Build Status” section).<br>
Example:
  ```markdown
  # GitHub Actions Practice

  [![Docker Publish](https://github.com/atulsharmadevops/github-actions-practice/actions/workflows/docker-publish.yml/badge.svg)](https://github.com/atulsharmadevops/github-actions-practice/actions/workflows/docker-publish.yml)

  A practice repository to learn GitHub Actions, Docker builds, and CI/CD pipelines.
  ```
### Step 3: Commit & Push
```bash
git add README.md
git commit -m "Add CI/CD status badge"
git push origin main
```
### Step 4: Verify
- Go back to our repo homepage.
- The badge should appear in the README.

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/cbd74e821da54d9581867e5fce6a8ff18dac1cf7/2026/day-45/Screenshots/Screenshot%20(448).png)

  - When our workflow succeeds, the badge shows **green**. If it fails, it shows **red**.

## Task 6: Pull and Run It
### Step 1: Pull the Image
- On our local machine:
  ```bash
  docker pull atulsharmadochub/github-actions-practice:latest
  ```
  - Replace `<DOCKER_USERNAME>` with Docker Hub username.
- We can also test a commit‑specific tag:
  ```bash
  docker pull atulsharmadochub/github-actions-practice:sha-5abb21d
  ```
    ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/cbd74e821da54d9581867e5fce6a8ff18dac1cf7/2026/day-45/Screenshots/Screenshot%20(449).png)

### Step 2: Run the Container
- Run the image and expose the app’s port:
  ```bash
  docker run -p 8080:8080 atulsharmadochub/github-actions-practice:latest
  ```
  - `-p 8080:8080` maps container port 8080 to our local machine’s port 8080.

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/cbd74e821da54d9581867e5fce6a8ff18dac1cf7/2026/day-45/Screenshots/Screenshot%20(454).png)

### Step 3: Confirm It Works
- Open our browser at `http://localhost:8080`.
- We should see our app running exactly as it did locally.

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/cbd74e821da54d9581867e5fce6a8ff18dac1cf7/2026/day-45/Screenshots/Screenshot%20(427).png)

### What is the full journey from git push to a running container?<br>
**Ans.**<br>
**Journey from Git Push → Running Container:**
1. Developer pushes code to GitHub (`git push origin main`).
2. GitHub Actions triggers the `docker-publish.yml` workflow.
3. Workflow checks out code, logs into Docker Hub, builds Docker image.
4. Image tagged as `latest` and `sha-<commit>` and pushed to Docker Hub.
5. Status badge in README shows workflow success.
6. On local/cloud machine, `docker pull` retrieves the image.
7. `docker run` starts the container.
8. Application is live and accessible - pipeline complete.

This closes the loop: **every push to `main` --> new Docker image --> pull and run anywhere**. That’s production‑grade CI/CD.

