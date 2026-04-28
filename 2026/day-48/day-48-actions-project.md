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
flask==2.2.5
```
### Add `Dockerfile`:
File: `Dockerfile`

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
### Add Basic Test Script
File: `test.sh`
```bash
#!/bin/bash
set -e

# Start container
docker build -t myapp:latest .
docker run -d -p 5000:5000 --name myapp myapp:latest

# Wait for app to start
sleep 5

# Test health endpoint
STATUS=$(curl -s http://localhost:5000/health | grep -o "ok")

if [ "$STATUS" == "ok" ]; then
  echo "Health check PASSED"
else
  echo "Health check FAILED"
  exit 1
fi

# Cleanup
docker stop myapp
docker rm myapp
```
Make it executable:
```bash
chmod +x test.sh
```
### Add a `README.md` with a brief project description and a placeholder for badges.

## Task 2: Reusable Workflow — Build & Test
Create `.github/workflows/reusable-build-test.yml`:
```yaml
name: Reusable Build & Test

on:
  workflow_call:
    inputs:
      python_version:
        required: true
        type: string
      run_tests:
        required: false
        type: boolean
        default: true

jobs:
  build-test:
    runs-on: ubuntu-latest
    outputs:
      test_result: ${{ steps.set-result.outputs.test_result }}
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set up Python
        if: ${{ inputs.python_version }}
        uses: actions/setup-python@v4
        with:
          python-version: ${{ inputs.python_version }}

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Run tests
        if: ${{ inputs.run_tests }}
        run: |
          set -e
          pytest || echo "Tests failed" && exit 1

      - name: Set result output
        id: set-result
        run: |
          if [ "${{ inputs.run_tests }}" = "true" ]; then
            echo "test_result=passed" >> $GITHUB_OUTPUT
          else
            echo "test_result=skipped" >> $GITHUB_OUTPUT
          fi
```
This workflow does NOT deploy — it only builds and tests.

## Task 3: Reusable Workflow — Docker Build & Push
**First**, go to Docker Hub → Account Settings → Security → create an access token. Then in your GitHub repo go to Settings → Secrets → add `DOCKER_USERNAME` and `DOCKER_TOKEN`.

Create `.github/workflows/reusable-docker.yml`:
```yaml
name: Reusable Docker Build & Push

on:
  workflow_call:
    inputs:
      image_name:
        required: true
        type: string
      tag:
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
    outputs:
      image_url: ${{ steps.set-output.outputs.image_url }}
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Log in to Docker Hub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.docker_username }}
          password: ${{ secrets.docker_token }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: ${{ inputs.image_name }}:${{ inputs.tag }}

      - name: Set image URL output
        id: set-output
        run: echo "image_url=${{ inputs.image_name }}:${{ inputs.tag }}" >> $GITHUB_OUTPUT

```
## Task 4: PR Pipeline
### Workflow File: `.github/workflows/pr-pipeline.yml`
```yaml
name: PR Pipeline

on:
  pull_request:
    branches: [ main ]
    types: [opened, synchronize]

jobs:
  # Call the reusable build-test workflow
  build-test:
    uses: ./.github/workflows/reusable-build-test.yml
    with:
      python_version: "3.10"
      run_tests: true

  # Add a standalone job for PR summary
  pr-comment:
    runs-on: ubuntu-latest
    needs: build-test
    steps:
      - name: Print PR summary
        run: echo "PR checks passed for branch: ${{ github.head_ref }}"
```
- **Trigger:** Runs only on PRs targeting `main` when opened or updated.
- **Job 1:** Calls your reusable build-test workflow (from Task 2).
- **Job 2:** Prints a summary message after tests complete.
- **No Docker build/push:** This workflow intentionally skips Docker steps to keep PR checks lightweight.

### Verification
- Open a PR against `main`.
- GitHub Actions will run:
    - Build & test job.
    - PR summary job.
- We should see output like:
    ```Code
    PR checks passed for branch: feature-xyz
    ```
- Confirm that **no Docker login/build/push steps** are executed.

## Task 5: Main Branch Pipeline
### Workflow File: `.github/workflows/main-pipeline.yml`
```yaml
name: Main Branch Pipeline

on:
  push:
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
    needs: build-test
    uses: ./.github/workflows/reusable-docker.yml
    with:
      image_name: mydockerhubuser/myapp
      tag: latest
    secrets:
      docker_username: ${{ secrets.DOCKER_USERNAME }}
      docker_token: ${{ secrets.DOCKER_TOKEN }}

  # Job 2b: Push with short SHA tag
  docker-build-push-sha:
    needs: build-test
    uses: ./.github/workflows/reusable-docker.yml
    with:
      image_name: mydockerhubuser/myapp
      tag: sha-${{ github.sha }}
    secrets:
      docker_username: ${{ secrets.DOCKER_USERNAME }}
      docker_token: ${{ secrets.DOCKER_TOKEN }}

  # Job 3: Deploy (depends on Docker jobs)
  deploy:
    runs-on: ubuntu-latest
    needs: [docker-build-push, docker-build-push-sha]
    environment: production
    steps:
      - name: Deploy to production
        run: echo "Deploying image: ${{ needs.docker-build-push.outputs.image_url }} to production"
```
- **Trigger:** Runs only on `push` to `main`.
- **Job 1:** Calls our reusable build-test workflow.
- **Job 2:** Calls reusable Docker workflow twice:
  - Once with `latest` tag.
  - Once with `sha-<commit>` tag (we can shorten SHA with `cut -c1-7` if desired).
- **Job 3:** Deploy job prints deployment message and uses `environment: production`.
  - If we set up environment protection rules in GitHub --> Settings --> Environments --> Production, this job will require manual approval.

### Verification
- Merge a PR into `main`.
- GitHub Actions will run:
  - Build & test job.
  - Docker build & push jobs (latest + SHA).
  - Deploy job (after Docker jobs succeed).
- We should see logs like:
  ```Code
  Deploying image: mydockerhubuser/myapp:latest to production
  ```
## Task 6: Scheduled Health Check
### Workflow File: .github/workflows/health-check.yml
```yaml
name: Scheduled Health Check

on:
  schedule:
    - cron: '0 */12 * * *'   # every 12 hours
  workflow_dispatch:          # manual trigger

jobs:
  health-check:
    runs-on: ubuntu-latest
    steps:
      - name: Pull latest Docker image
        run: docker pull mydockerhubuser/myapp:latest

      - name: Run container
        run: docker run -d -p 5000:5000 --name myapp mydockerhubuser/myapp:latest

      - name: Wait for app to start
        run: sleep 5

      - name: Curl health endpoint
        id: curl
        run: |
          if curl -s http://localhost:5000/health | grep -q "ok"; then
            echo "status=PASSED" >> $GITHUB_OUTPUT
          else
            echo "status=FAILED" >> $GITHUB_OUTPUT
          fi

      - name: Stop and remove container
        run: |
          docker stop myapp
          docker rm myapp

      - name: Write summary
        run: |
          echo "## Health Check Report" >> $GITHUB_STEP_SUMMARY
          echo "- Image: mydockerhubuser/myapp:latest" >> $GITHUB_STEP_SUMMARY
          echo "- Status: ${{ steps.curl.outputs.status }}" >> $GITHUB_STEP_SUMMARY
          echo "- Time: $(date)" >> $GITHUB_STEP_SUMMARY
```
- **Trigger:** Runs every 12 hours via cron, plus manual dispatch.
- **Steps:**
  - Pull latest Docker image.
  - Run container in detached mode.
  - Wait 5 seconds.
  - Curl `/health` endpoint --> check response.
  - Stop & remove container.
  - Write a clean summary to `$GITHUB_STEP_SUMMARY`.
- **Summary:** Appears in the Actions run summary page with a neat report.

### Verification
- Trigger manually (`workflow_dispatch`) to test.
- Check the run summary ---> we should see a markdown report like:
  ```Code
  ## Health Check Report
  - Image: mydockerhubuser/myapp:latest
  - Status: PASSED
  - Time: Fri Apr 17 12:32 IST 2026
  ```
## Task 7: Add Badges & Documentation
Add status badges for all your workflows to the repo README.md
Add a pipeline architecture diagram in your notes — draw (or describe) the flow:
PR opened → build & test → PR checks pass
Merge to main → build & test → Docker build & push → deploy
Every 12 hours → health check
Fill in your notes: What would you add next? (Slack notifications? Multi-environment? Rollback?)

## Brownie Points: Add Security to Your Pipeline
### Workflow Update: Add Security Scan
Modify `.github/workflows/main-pipeline.yml` after the Docker build job:

```yaml
jobs:
  # ... existing jobs ...

  docker-build-push:
    needs: build-test
    uses: ./.github/workflows/reusable-docker.yml
    with:
      image_name: mydockerhubuser/myapp
      tag: latest
    secrets:
      docker_username: ${{ secrets.DOCKER_USERNAME }}
      docker_token: ${{ secrets.DOCKER_TOKEN }}

  security-scan:
    needs: docker-build-push
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@0.20.0
        with:
          image-ref: mydockerhubuser/myapp:latest
          format: 'table'
          exit-code: '1'
          severity: 'CRITICAL'
          output: trivy-report.txt

      - name: Upload Trivy scan report
        uses: actions/upload-artifact@v3
        with:
          name: trivy-report
          path: trivy-report.txt

  deploy:
    runs-on: ubuntu-latest
    needs: [docker-build-push, security-scan]
    environment: production
    steps:
      - name: Deploy to production
        run: echo "Deploying image: ${{ needs.docker-build-push.outputs.image_url }} to production"
```
- **Trivy Action:** `aquasecurity/trivy-action` scans the Docker image.
- **Fail on CRITICAL:** `exit-code: 1` ensures the pipeline fails if CRITICAL vulnerabilities are found.
- **Report Upload:** `actions/upload-artifact` saves the scan report for review.
- **Deploy Job Dependency:** The deploy job waits for both Docker build and security scan to succeed.

### Verification
- Merge a PR to `main`.
- Pipeline runs: build --> Docker push --> Trivy scan --> deploy.
- If CRITICAL CVEs are found:
  - Pipeline fails at the security scan step.
  - Report is available in the workflow artifacts.
- If clean:
  - Deploy job runs as usual.