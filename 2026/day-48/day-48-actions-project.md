# GitHub Actions Project: End-to-End CI/CD Pipeline
## Task 1: Set Up the Project Repo
### Create the repo on GitHub: `github-actions-capstone`
### Create a simple Flask app - `app.py`:
```python
from flask import Flask

app = Flask(__name__)

@app.route("/health")
def health():
    return {"status": "ok"}, 200

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```
### Add requirements.txt:
```code
Flask==3.0.3    #AttributeError: module 'werkzeug' has no attribute '__version__'
pytest
```
**Create a Virtual Environment**
```bash
python3 -m venv venv
```
- This creates a folder `venv/` with its own Python + pip.

**Activate the Virtual Environment**
```bash
source venv/bin/activate
```
- We should see `(venv)` at the start of our shell prompt.

**Install Requirements**
```bash
pip install -r requirements.txt
```
- Now it will install Flask (and any other dependencies) inside the virtual environment without touching system Python.

**Run Our App**
```bash
python app.py
```
Then test:
```bash
curl http://localhost:5000/health
```
Expected:
```json
{"status":"ok"}
```
**Deactivate When Done**
```bash
deactivate
```
### Add `Dockerfile`:
```dockerfile
# Use lightweight Python image
FROM python:3.10-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy app code
COPY app.py .

# Expose port
EXPOSE 5000

# Run the app
CMD ["python", "app.py"]
```
**Build and run the container**
```bash
docker build -t myapp:latest .
docker run -d -p 5000:5000 --name myapp myapp:latest
sleep 5
curl http://localhost:5000/health
```
**Expected output**
```json
{"status":"ok"}
```
**Then clean up**
```bash
docker stop myapp && docker rm myapp
```
**Add a minimal test file so the workflow has something to run**
  ```python
  # tests/test_app.py
  import json
  import sys
  import os

  # Add repo root to sys.path
  sys.path.insert(0, os.path.abspath(os.path.join(os.path.dirname(__file__), "..")))

  from app import app

  def test_health_endpoint():
      client = app.test_client()
      response = client.get("/health")
      data = json.loads(response.data)
      assert response.status_code == 200
      assert data["status"] == "ok"
  ```
  - Commit this under `tests/` and push. Now `pytest` will pass.

## Task 2: Reusable Workflow — Build & Test
**Create `.github/workflows/reusable-build-test.yml`:**
```yaml
name: Reusable Build & Test

on:
  workflow_call:                        #makes this workflow reusable - other workflows can call it
    inputs:                             #parameters passed in by caller
      python_version:                   #required string (e.g., "3.10")
        required: true
        type: string
      run_tests:                        #optional boolean, defaults to true
        required: false
        type: boolean
        default: true

jobs:
  build-test:
    runs-on: ubuntu-latest
    outputs:                                                      #exposes job outputs to caller workflow
      test_result: ${{ steps.set-result.outputs.test_result }}    #it maps job’s test_result to step set-result’s output.
    steps:
      - name: Checkout code
        uses: actions/checkout@v3                                 #checks out our repo code so subsequent steps can access it

      - name: Set up Python
        if: ${{ inputs.python_version }}                          #conditionally runs if python_version input is provided
        uses: actions/setup-python@v4                             #uses official action to install specified Python version
        with:
          python-version: ${{ inputs.python_version }}

      - name: Install dependencies
        run: pip install -r requirements.txt                      #installs dependencies listed in requirements.txt

      - name: Run tests
        if: ${{ inputs.run_tests }}                               #runs only if run_tests is true
        run: |
          set -e                                                  #ensures script exits immediately if any command fails
          pytest                                                  #it automatically discovers test files (named test_*.py or *_test.py) --> runs functions inside them --> reports results (pass/fail --> exits with a status code that CI/CD systems can use

      - name: Set result output
        id: set-result                                            #defines a step with id: set-result so its outputs can be referenced
        run: |
          if [ "${{ inputs.run_tests }}" = "true" ]; then
            echo "test_result=passed" >> $GITHUB_OUTPUT
          else
            echo "test_result=skipped" >> $GITHUB_OUTPUT
          fi                                                      #this value is then exposed as job output (${{ steps.set-result.outputs.test_result }})
```
- This workflow does NOT deploy — it only builds and tests.

## Task 3: Reusable Workflow — Docker Build & Push
**First**, go to **Docker Hub --> Settings --> Personal access tokens --> Generate new token**.

Then in our GitHub repo go to **Settings --> Secrets and variables --> Actions --> add `DOCKER_USERNAME` and `DOCKER_TOKEN`**.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b3f3e42b1d8ca0b597460fefa53c7c5185b4d8e9/2026/day-48/Screenshots/Screenshot%20(563).png)

Create `.github/workflows/reusable-docker.yml`:
```yaml
name: Reusable Docker Build & Push

on:
  workflow_call:                                                                             #makes this workflow reusable - other workflows can call it
    inputs:
      image_name:                                                                            #docker image name (e.g., atulsharmadochub/myapp)
        required: true
        type: string
      tag:                                                                                   #tag (e.g., latest or sha-12345)
        required: true
        type: string
    secrets:
      docker_username:
        required: true
      docker_token:
        required: true

jobs:
  docker-build-push:
    runs-on: ubuntu-latest
    outputs:                                                                                   #exposes 'image_url' to caller workflow, wired to step 'set-output'
      image_url: ${{ steps.set-output.outputs.image_url }}
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Log in to Docker Hub                                                             #ensures runner can push images to our account
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.docker_username }}
          password: ${{ secrets.docker_token }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v4
        with:
          context: .                                                                            #build from current directory
          push: true                                                                            #push image to Docker Hub
          tags: ${{ inputs.image_name }}:${{ inputs.tag }}                                      #applies tag (e.g., atulsharmadochub/myapp:latest).

      - name: Set image URL output                                                              #this job maps this to its own output so caller workflow can use it
        id: set-output                                                                          #defines a step with id: set-output
        run: echo "image_url=${{ inputs.image_name }}:${{ inputs.tag }}" >> $GITHUB_OUTPUT      #this makes steps.set-output.outputs.image_url available

```
## Task 4: PR Pipeline
### Workflow File: `.github/workflows/pr-pipeline.yml`
```yaml
name: PR Pipeline

on:
  pull_request:                                                          #runs when a pull request event occurs
    branches: [ main ]                                                   #only triggers if PR targets main branch
    types: [opened, synchronize]                                         #runs when PR is opened or PR is synchronized (updated with new commits)

jobs:
  # Call the reusable build-test workflow
  build-test:                                                            #this job validates PR code by building and testing it
    uses: ./.github/workflows/reusable-build-test.yml                    #calls our reusable-build-test workflow
    with:
      python_version: "3.10"
      run_tests: true                                                    #runs test suite

  # Add a standalone job for PR summary
  pr-comment:
    runs-on: ubuntu-latest
    needs: build-test                                                     #waits until build-test job finishes successfully
    steps:
      - name: Print PR summary
        run: echo "PR checks passed for branch: ${{ github.head_ref }}"   #(github.head_ref) - the source branch of the PR 
```
### Verification
- Push our workflows to the `main` branch
  ```bash
  git add reusable-build-test.yml reusable-docker.yml pr-pipeline.yml
  git commit -m "Added reusable build-test, docker, and PR pipeline workflows"
  git push origin main
  ```
- Go to our repo’s **Actions tab**. We should see the **PR pipeline** listed.

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b3f3e42b1d8ca0b597460fefa53c7c5185b4d8e9/2026/day-48/Screenshots/Screenshot%20(552).png)

- Trigger the PR pipeline
  ```bash
  git checkout -b feature/test-pr
  echo "Enforcing code quality checks before merging into main." >> README.md
  git add README.md
  git commit -m "Testing PR trigger"
  git push origin feature/test-pr
  ```
- Open a **pull request** into `main`. The **PR pipeline** will run automatically.

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b3f3e42b1d8ca0b597460fefa53c7c5185b4d8e9/2026/day-48/Screenshots/Screenshot%20(545).png)

- GitHub Actions will run:
    - Build & test job.
    - PR summary job.

      ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b3f3e42b1d8ca0b597460fefa53c7c5185b4d8e9/2026/day-48/Screenshots/Screenshot%20(547).png)

- We should see output like:
  
  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b3f3e42b1d8ca0b597460fefa53c7c5185b4d8e9/2026/day-48/Screenshots/Screenshot%20(550).png)

- Confirm that **no Docker login/build/push steps** are executed.

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b3f3e42b1d8ca0b597460fefa53c7c5185b4d8e9/2026/day-48/Screenshots/Screenshot%20(551).png)

## Task 5: Main Branch Pipeline
### Workflow File: `.github/workflows/main-pipeline.yml`
```yaml
name: Main Branch Pipeline

on:
  push:                                                                                               #this ensures only production‑ready code triggers pipeline
    branches: [ main ]

jobs:
  # Job 1: Build & Test
  build-test:
    uses: ./.github/workflows/reusable-build-test.yml
    with:
      python_version: "3.10"
      run_tests: true

  # Job 2: Docker Build & Push (depends on build-test)
  docker-build-push:
    needs: build-test                                                                                  #runs only after build-test passes
    uses: ./.github/workflows/reusable-docker.yml                                                      #calls our reusable-docker.yml workflow
    with:
      image_name: atulsharmadochub/myapp
      tag: latest                                                                                      #builds and pushes image tagged latest to Docker Hub
    secrets:
      docker_username: ${{ secrets.DOCKER_USERNAME }}
      docker_token: ${{ secrets.DOCKER_TOKEN }}

  # Job 2b: Push with short SHA tag
  docker-build-push-sha:
    needs: build-test                                                                                   #runs only after build-test passes
    uses: ./.github/workflows/reusable-docker.yml                                                       #calls our reusable-docker.yml workflow
    with:
      image_name: atulsharmadochub/myapp
      tag: sha-${{ github.sha }}                                                                        #builds and pushes same image but tagged with commit SHA - this gives us a unique version per commit for traceability
    secrets:
      docker_username: ${{ secrets.DOCKER_USERNAME }}
      docker_token: ${{ secrets.DOCKER_TOKEN }}

  # Job 3: Deploy (depends on Docker jobs)
  deploy:                                                                                               #echoes image URL from docker-build-push job
    runs-on: ubuntu-latest
    needs: [docker-build-push, docker-build-push-sha]                                                   #runs only after both Docker jobs succeed
    environment: production                                                                             #marks job as targeting production environment
    steps:
      - name: Deploy to production
        run: echo "Deploying image: ${{ needs.docker-build-push.outputs.image_url }} to production"     #right now it’s just a placeholder — in a real pipeline, we’d replace this with deployment commands (e.g., kubectl apply, docker service update, etc.)
```

### Verification
- Merge a PR into `main`.
  - Merge commit --> creates a new commit on `main` --> triggers `push`.
- GitHub Actions will run:
  - Build & test job.
  - Docker build & push jobs (latest + SHA).
  - Deploy job (after Docker jobs succeed).

    ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b3f3e42b1d8ca0b597460fefa53c7c5185b4d8e9/2026/day-48/Screenshots/Screenshot%20(565).png)

- We should see logs like:

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b3f3e42b1d8ca0b597460fefa53c7c5185b4d8e9/2026/day-48/Screenshots/Screenshot%20(566).png)

## Task 6: Scheduled Health Check
### Workflow File: `.github/workflows/health-check.yml`
```yaml
name: Scheduled Health Check

on:
  schedule:
    - cron: '0 */12 * * *'                                                 # every 12 hours
  workflow_dispatch:                                                       # manual trigger

jobs:
  health-check:
    runs-on: ubuntu-latest
    steps:
      - name: Pull latest Docker image
        run: docker pull atulsharmadochub/myapp:latest

      - name: Run container
        run: docker run -d -p 5000:5000 --name myapp atulsharmadochub/myapp:latest

      - name: Wait for app to start
        run: sleep 10

      - name: Curl health endpoint
        id: curl
        run: |
          if curl -s http://localhost:5000/health | grep -q "ok"; then     #hits /health endpoint on running container
            echo "status=PASSED" >> $GITHUB_OUTPUT                         #if response contains "ok", sets status=PASSED otherwise status=FAILED
          else
            echo "status=FAILED" >> $GITHUB_OUTPUT                         #writes result to $GITHUB_OUTPUT so later steps can use it
          fi

      - name: Stop and remove container
        run: |
          docker stop myapp
          docker rm myapp

      - name: Write summary                                                #writes a Markdown summary to GitHub Actions job summary which is visible in Actions run details
        run: |
          echo "## Health Check Report" >> $GITHUB_STEP_SUMMARY
          echo "- Image: atulsharmadochub/myapp:latest" >> $GITHUB_STEP_SUMMARY
          echo "- Status: ${{ steps.curl.outputs.status }}" >> $GITHUB_STEP_SUMMARY
          echo "- Time: $(date)" >> $GITHUB_STEP_SUMMARY
```
### Verification
- Trigger manually (`workflow_dispatch`) to test.
- Check the run summary ---> we should see a markdown report like:

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b3f3e42b1d8ca0b597460fefa53c7c5185b4d8e9/2026/day-48/Screenshots/Screenshot%20(559).png)

## Task 7: Add Badges & Documentation
- Add status badges for all our workflows to the repo `README.md`

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b3f3e42b1d8ca0b597460fefa53c7c5185b4d8e9/2026/day-48/Screenshots/Screenshot%20(567).png)

- Add a pipeline architecture diagram:
  ```Code
  PR opened
    ↓
  Build & Test
    ↓
  PR checks pass
    ↓
  Merge to main
    ↓
  Build & Test
    ↓
  Docker Build & Push
    ↓
  Deploy
    ↓
  ───────────────
  Every 12 hours
    ↓
  Health Check
  ```
- What would we can add next? (Slack notifications? Multi-environment? Rollback?)<br>
**Ans.**
  - **Slack notifications**  
  Send alerts when builds fail or deploys succeed. This keeps your team in the loop without checking the Actions tab.
  - **Multi‑environment support**  
  Separate pipelines for dev → staging → prod. Add approvals or manual gates before promoting to production.
  - **Rollback strategy**  
  If a deploy fails, automatically revert to the last stable Docker image. This ensures uptime and reliability.
  - **Security scans**  
  Run Trivy or Snyk on your Docker images to catch vulnerabilities before pushing to production.
  - **Coverage reports**  
  Publish test coverage results and add a badge to your README. This shows code quality at a glance.
  - **Artifact retention**  
  Save build artifacts (logs, binaries) for debugging and auditing.
## Brownie Points: Add Security to Our Pipeline
### Workflow Update: Add Security Scan
Modify `.github/workflows/main-pipeline.yml` after the Docker build job:

```yaml
jobs:
  # ... existing jobs ...

  docker-build-push:
    needs: build-test
    uses: ./.github/workflows/reusable-docker.yml
    with:
      image_name: atulsharmadochub/myapp
      tag: latest
    secrets:
      docker_username: ${{ secrets.DOCKER_USERNAME }}
      docker_token: ${{ secrets.DOCKER_TOKEN }}

  security-scan:
    needs: docker-build-push                                # runs only after Docker image has been built and pushed
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@v0.36.0
        with:
          image-ref: atulsharmadochub/myapp:latest          #scans pushed Docker image
          format: 'table'                                   #outputs results in a human‑readable table
          exit-code: '1'                                    #if CRITICAL vulnerabilities are found, the job fails
          severity: 'CRITICAL'                              #only reports critical issues
          output: trivy-report.txt                          #saves report to a file

      - name: Upload Trivy scan report
        uses: actions/upload-artifact@v4                    #attaches report to workflow run so we can download it later
        with:
          name: trivy-report
          path: trivy-report.txt

  deploy:
    runs-on: ubuntu-latest
    needs: [docker-build-push, security-scan]                #deploy only happens if both Docker build and security scan succeed
    environment: production
    steps:
      - name: Deploy to production
        run: echo "Deploying image: ${{ needs.docker-build-push.outputs.image_url }} to production"
```
### Verification
- Push to `main` branch:
  ```bash
  git add main-pipeline.yml
  git commit -m "Added Security Scan to Main Pipeline"
  git push origin main
  ```
- Pipeline runs: build --> Docker push --> Trivy scan --> deploy.

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b3f3e42b1d8ca0b597460fefa53c7c5185b4d8e9/2026/day-48/Screenshots/Screenshot%20(561).png)

- If CRITICAL CVEs are found:
  - Pipeline fails at the security scan step.
  - Report is available in the workflow artifacts.
- If clean:
  - Deploy job runs as usual.
