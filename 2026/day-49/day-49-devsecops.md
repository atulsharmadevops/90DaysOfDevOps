# DevSecOps: Add Security to Our CI/CD Pipeline
> Security doesn't have to be a separate process bolted on at the end. DevSecOps means adding automated security checks to the pipeline we already have — catching vulnerabilities before they ever reach production.
 
**Without DevSecOps:** We build the app → deploy it → a security team finds a vulnerability weeks later → we scramble to fix it.
 
**With DevSecOps:** We open a PR → the pipeline automatically checks for vulnerabilities → we fix it before it ever gets merged.
 
## Table of Contents
1. [Key Principles](#1-key-principles)
2. [Scan Docker Images with Trivy](#2-scan-docker-images-with-trivy)
3. [Enable GitHub Secret Scanning](#3-enable-github-secret-scanning)
4. [Scan Dependencies for Known Vulnerabilities](#4-scan-dependencies-for-known-vulnerabilities)
5. [Lock Down Workflow Permissions](#5-lock-down-workflow-permissions)
6. [The Full Secure Pipeline](#6-the-full-secure-pipeline)
7. [Bonus - Going Further](#7-bonus---going-further)
 
## 1. Key Principles
| Principle | Why It Matters |
|---|---|
| **Catch problems early** | A vulnerability found in a PR takes 5 minutes to fix. The same vulnerability in production takes days. |
| **Automate the checks** | Don't rely on someone remembering. Let the pipeline check every time, automatically. |
| **Block on critical issues** | If a scan finds a serious vulnerability, fail the pipeline — just like a failing test. |
| **Never put secrets in code** | Use GitHub Secrets. No `.env` files committed, no hardcoded API keys. |
| **Give only the access needed** | Limit workflow permissions. Your workflow doesn't need write access to everything. |

## 2. Scan Docker Images with Trivy
Trivy pulls CVE (Common Vulnerabilities and Exposures) data and scans our built Docker image for known security issues.

### Clone the Repository
```bash
git clone https://github.com/atulsharmadevops/github-actions-capstone.git
```

`.github/workflows/main-pipeline.yml`
```yaml
name: Main Branch Pipeline                                      # human‑readable label - it shows up in the Actions tab 

on:
  push:                                                         # defines when workflow runs
    branches: [ main ]                                          # it only triggers when code is pushed to 'main' branch - this makes it production pipeline

jobs:
  build-test:                                                   # job name
    uses: ./.github/workflows/reusable-build-test.yml           # calls a reusable workflow file stored in our repo
    with:                                                       # passes inputs to that reusable workflow
      python_version: "3.10"                                    # run tests using Python 3.10
      run_tests: true                                           #  ensures test suite executes

  docker-build-push:
    needs: build-test                                           # job waits until 'build-test' finishes successfully
    uses: ./.github/workflows/reusable-docker.yml               # calls our reusable docker workflow
    with:                                                       # inputs for Docker build
      image_name: atulsharmadochub/myapp                        # docker hub repo name
      tag: latest                                               # tags image as 'latest'
    secrets:                                                    # injects docker hub credentials securely from GitHub Secrets
      docker_username: ${{ secrets.DOCKER_USERNAME }}
      docker_token: ${{ secrets.DOCKER_TOKEN }}

  docker-build-push-sha:
    needs: build-test
    uses: ./.github/workflows/reusable-docker.yml
    with:
      image_name: atulsharmadochub/myapp
      tag: sha-${{ github.sha }}                                # it tags image with commit SHA (sha-<commit>) - ensures every commit has a unique, traceable image version
    secrets:
      docker_username: ${{ secrets.DOCKER_USERNAME }}
      docker_token: ${{ secrets.DOCKER_TOKEN }}

  deploy:
    runs-on: ubuntu-latest                                      # runs on a GitHub‑hosted Ubuntu runner
    needs: [docker-build-push, docker-build-push-sha]           # waits until both Docker jobs finish
    environment: production                                     # marks this job as deploying to production environment (adds protection rules if configured)
    steps:                                                      # actual commands
      - name: Deploy to production
        run: 'echo "Deploying image: ${{ needs.docker-build-push.outputs.image_url }} to production"'       # right now it just prints a message. In a real pipeline, replace this with deployment commands (e.g., kubectl apply, helm upgrade, etc.)
```
Already handles build, test, Docker build, and deploy.

### Add the Trivy Scan Step
Insert this job **after** `docker-build-push` and **before** `deploy`:
```yaml
  trivy-scan:                                                         # defines new job name
    runs-on: ubuntu-latest                                            # executes on a GitHub‑hosted Ubuntu runner
    needs: docker-build-push                                          # this job only runs after docker-build-push job finishes successfully (so the image exists)
    steps:                                                            # actual commands to run inside this job
      - name: Scan Docker image for vulnerabilities                   # human‑readable label for step
        uses: aquasecurity/trivy-action@master                        # calls Trivy GitHub Action to perform scan
        with:                                                         # inputs passed to action
          image-ref: 'atulsharmadochub/myapp:latest'                  # specifies docker image to scan (one we just built and tagged latest)
          format: 'table'                                             # outputs results in a human‑readable table format in logs
          exit-code: '1'                                              # if vulnerabilities of the specified severity are found, job fails with exit code 1 - this blocks deployment
          severity: 'CRITICAL,HIGH'                                   # only fail job if CRITICAL or HIGH vulnerabilities are detected

  deploy:
    runs-on: ubuntu-latest
    needs: [docker-build-push, docker-build-push-sha, trivy-scan]     # add trivy-scan
    environment: production
    steps:
      - name: Deploy to production
        run: 'echo "Deploying image: ${{ needs.docker-build-push.outputs.image_url }} to production"'
```
 What each option does:
| Option | Value | Effect |
|---|---|---|
| `image-ref` | `atulsharmadochub/myapp:latest` | The image to scan |
| `format` | `table` | Prints a readable CVE table in the logs |
| `exit-code` | `1` | Pipeline fails if CRITICAL or HIGH issues are found |
| `severity` | `CRITICAL,HIGH` | Only these severity levels trigger a failure |

### Commit and Verify
```bash
git add .github/workflows/main-pipeline.yml
git commit -m "Add Trivy Docker image security scan"
git push origin main
```
Open **Actions → latest run → Trivy scan step** in the logs.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/4e4ea43d169749a92825892b1a04749d5db3113d/2026/day-49/Screenshots/Screenshot%20(1377).png)

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/4e4ea43d169749a92825892b1a04749d5db3113d/2026/day-49/Screenshots/Screenshot%20(1381).png)

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/4e4ea43d169749a92825892b1a04749d5db3113d/2026/day-49/Screenshots/Screenshot%20(1383).png)

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/4e4ea43d169749a92825892b1a04749d5db3113d/2026/day-49/Screenshots/Screenshot%20(1385).png)

| Result | Outcome |
|---|---|
| Vulnerabilities found | Pipeline fails — logs show CVE table with affected packages and versions |
| Clean scan | Pipeline continues — image is pushed and deployed |
 
### What CVEs were found?
Trivy reported 12 vulnerabilities in total for atulsharmadochub/myapp:latest:

| Library/Package | CVE ID | Severity |	Notes |
| --- | --- | --- | --- |
| **libncursesw6** |	`CVE-2025-69720` |	HIGH |	Buffer overflow vulnerability in ncurses, may lead to arbitrary code execution. |
| **libsqlite3-0** |	`CVE-2026-11822` |	HIGH |	Memory corruption vulnerability in SQLite < 3.53.2. |
| **libsqlite3-0** |	`CVE-2026-11824` |	HIGH |	Heap-based buffer overflow vulnerability in SQLite < 3.53.2. |
| **Other Debian packages** |	**Multiple** |	HIGH/CRITICAL |	10 HIGH, 2 CRITICAL vulnerabilities overall. |
>Python packages (Flask, Click, Jinja2, etc.) scanned clean — no vulnerabilities detected.

### What Base Image are we using?
- **Detected OS**: Debian 13.5 (Bookworm)
- This matches the base image we’re using: `python:3.11-slim`, which is built on Debian.

## 3. Enable GitHub's Built-in Secret Scanning
GitHub scans commits for patterns that look like secrets — API keys, tokens, passwords — automatically in the background. No workflow changes needed.
 
### Enable in Repository Settings
1. Go to your repo → **Settings → Advanced Security → Secret Protection**
2. Enable **Secret Protection** — scans commits for leaked secrets after they are pushed
3. Enable **Push Protection** — blocks a push entirely if a secret is detected before it reaches the repository

    ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/4e4ea43d169749a92825892b1a04749d5db3113d/2026/day-49/Screenshots/Screenshot%20(1388).png)

### Secret Scanning vs. Push Protection
| | Secret Scanning | Push Protection |
|---|---|---|
| **When it runs** | After code is pushed | Before code is pushed |
| **What it does** | Detects secrets already in the repo | Blocks the commit if a secret is detected |
| **Human action required** | Review and rotate detected secrets | Fix the commit before pushing |
 
### What Happens if GitHub Detects a Leaked AWS Key
- GitHub flags the secret in the **Security tab** and sends an alert
- For AWS keys specifically, **GitHub notifies AWS directly** — AWS may automatically revoke the key to prevent misuse
- We must rotate the secret immediately and purge it from our Git history (`git filter-repo` or BFG Repo Cleaner)

## 4. Scan Dependencies for Known Vulnerabilities
When a PR adds a new package, `dependency-review-action` checks it against a vulnerability database before the code is merged.
 
### Update the PR Pipeline
Open `.github/workflows/pr-pipeline.yml` and add dependency vulnerability check step after our build/test steps:
```yaml
name: PR Pipeline                                                           # workflow name - human‑readable label - appears in the Actions tab

on:
  pull_request:                                                             # runs when a pull request targets 'main' branch
    branches: [ main ]                                                      # only PRs into 'main' trigger this workflow
    types: [opened, synchronize]                                            # runs when a PR is first opened or updated (synchronize = new commits pushed to PR branch)

jobs:
  # Call the reusable build-test workflow
  build-test:
    uses: ./.github/workflows/reusable-build-test.yml
    with:
      python_version: "3.10"
      run_tests: true

  # Add dependency vulnerability check
  dependency-review:                                                         # job name 
    runs-on: ubuntu-latest                                                   # executes on a GitHub‑hosted Ubuntu runner
    needs: build-test                                                        # runs only after tests pass
    steps:                                                                   # actual commands
      - name: Check dependencies for vulnerabilities
        uses: actions/dependency-review-action@v4                            # gitHub’s official action to scan new/changed dependencies in PR
        with:
          fail-on-severity: critical                                         # if any dependency with a critical CVE is introduced, the job fails - this blocks merging insecure code

  # Add a standalone job for PR summary
  pr-comment:
    runs-on: ubuntu-latest
    needs: [build-test, dependency-review]   # runs only if both tests and dependency review succeed
    steps:
      - name: Print PR summary
        run: echo "PR checks passed for branch: ${{ github.head_ref }}"      # prints a summary message with PR branch name
```
`fail-on-severity: critical` — the PR check fails if a newly introduced dependency has a critical CVE. Lower severity issues are reported in the logs but do not block the merge.
 
### Commit and Test
```bash
git add .github/workflows/pr-pipeline.yml
git commit -m "Add dependency vulnerability review to PR pipeline"
git push origin main
```

### Enable Dependency Graph
The **dependency review action** only works if GitHub’s **Dependency Graph** feature is enabled for our repository.
- Go to our repo on GitHub → **Settings**.
- Navigate to **Security & analysis**.
- Enable these options:
  - **Dependency graph**
  - **Dependabot alerts** (optional but recommended)
  - **Dependabot security updates** (optional)
Once enabled, GitHub will start indexing our `requirements.txt`, `package.json`, or other manifest files.

### Open A Pull Request
Create a New Branch
```bash
git checkout -b feature/add-dependency
```
Add a new dependency in `requirements.txt` (Python):
```txt
flask==3.0.3
requests==2.32.0
```
Commit the change to new branch:
```bash
git add requirements.txt
git commit -m "Add requests dependency"
git push origin feature/add-dependency
```
Open a **Pull Request** → Compare `feature/add-dependency` against `main`.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/4e4ea43d169749a92825892b1a04749d5db3113d/2026/day-49/Screenshots/Screenshot%20(1394).png)

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/4e4ea43d169749a92825892b1a04749d5db3113d/2026/day-49/Screenshots/Screenshot%20(1395).png)

Open **Actions → PR run → dependency review step** in the logs.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/4e4ea43d169749a92825892b1a04749d5db3113d/2026/day-49/Screenshots/Screenshot%20(1396).png)

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/4e4ea43d169749a92825892b1a04749d5db3113d/2026/day-49/Screenshots/Screenshot%20(1399).png)

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/4e4ea43d169749a92825892b1a04749d5db3113d/2026/day-49/Screenshots/Screenshot%20(1397).png)

| Result | Outcome |
|---|---|
| Critical CVE found | PR check fails — must remove or replace the dependency |
| Clean scan | PR passes — merge is allowed |

## 5. Add Permissions to Your Workflows
By default, GitHub Actions workflows receive **broad read/write permissions**. Restricting them reduces the blast radius if an action is ever compromised.

### Add a Permissions Block
- Go to `.github/workflows/` in our repo.
- Apply this to at least two workflow files (`main-pipeline.yml` and `pr-pipeline.yml`).
- Place a `permissions:` block directly after the `on:` section in each workflow file.

`main-pipeline.yml`
```yaml
name: Main Branch Pipeline

on:
  push:
    branches: [ main ]

permissions:          # defines what GITHUB_TOKEN can do
  contents: read      # restricts to read‑only access to repo contents (least privilege)

jobs:
  build-test:
    uses: ./.github/workflows/reusable-build-test.yml
    with:
      python_version: "3.10"
      run_tests: true

  docker-build-push:
    needs: build-test
    uses: ./.github/workflows/reusable-docker.yml
    with:
      image_name: atulsharmadochub/myapp
      tag: latest
    secrets:
      docker_username: ${{ secrets.DOCKER_USERNAME }}
      docker_token: ${{ secrets.DOCKER_TOKEN }}

  docker-build-push-sha:
    needs: build-test
    uses: ./.github/workflows/reusable-docker.yml
    with:
      image_name: atulsharmadochub/myapp
      tag: sha-${{ github.sha }}
    secrets:
      docker_username: ${{ secrets.DOCKER_USERNAME }}
      docker_token: ${{ secrets.DOCKER_TOKEN }}

  trivy-scan:
    runs-on: ubuntu-latest
    needs: docker-build-push
    steps:
      - name: Scan Docker image for vulnerabilities
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'atulsharmadochub/myapp:latest'
          format: 'table'
          exit-code: '1'
          severity: 'CRITICAL,HIGH'

  deploy:
    runs-on: ubuntu-latest
    needs: [docker-build-push, docker-build-push-sha, trivy-scan]
    environment: production
    steps:
      - name: Deploy to production
        run: echo "Deploying image: ${{ needs.docker-build-push.outputs.image_url }} to production"
```
- This restricts the workflow to only read repository contents.

`pr-pipeline.yml`
  ```yaml
  name: PR Pipeline

  on:
    pull_request:
      branches: [ main ]
      types: [opened, synchronize]

  permissions:              # defines what GITHUB_TOKEN can do
    contents: read          # grants read‑only access to repository contents
    pull-requests: write    # allows the workflow to comment or update PRs (needed if we later add a step to post a comment back to PR)

  jobs:
    build-test:
      uses: ./.github/workflows/reusable-build-test.yml
      with:
        python_version: "3.10"
        run_tests: true

    dependency-review:
      runs-on: ubuntu-latest
      needs: build-test
      steps:
        - name: Check dependencies for vulnerabilities
          uses: actions/dependency-review-action@v4
          with:
            fail-on-severity: critical

    pr-comment:
      runs-on: ubuntu-latest
      needs: [build-test, dependency-review]
      steps:
        - name: Print PR summary
          run: echo "PR checks passed for branch: ${{ github.head_ref }}"
  ```
  - This gives just enough access to interact with PRs, nothing more.

### Commit and Push Changes
```bash
git add .main-pipeline.yml pr-pipeline.yml
git commit -m "Add permissions blocks to workflows for least privilege"
git push origin main
```
### Verify in GitHub
- Go to our repo → **Actions** tab.
- Trigger a run (**Push** or **PR**).
- Open the workflow logs → we’ll see the jobs running with the **reduced permissions** we set.

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/4e4ea43d169749a92825892b1a04749d5db3113d/2026/day-49/Screenshots/Screenshot%20(1407).png)

### How to Verify Reduced Permissions
Try an action that requires write access
- For example, in `main-pipeline.yml`, if we add a step that tries to push a commit or open an issue, it will fail because the workflow no longer has write permissions.
- In `pr-pipeline.yml`, posting a PR comment will succeed (because we allowed `pull-requests: write`), but trying to push code would fail.

### Why is it a good practice to limit workflow permissions?
- By default, workflows have broad access (read/write to repo).
- Limiting permissions reduces the attack surface if an action is compromised.

### What could go wrong if a compromised action has write access to your repo?
- A malicious or compromised action could:
    - Push unwanted commits to your repo
    - Delete or overwrite files
    - Exfiltrate sensitive data
- With restricted permissions, the damage is contained.

## 6. The Full Secure Pipeline
After today, your pipeline looks like this:
```
PR opened
  → build & test
  → dependency vulnerability check     ← added Day 49
  → PR checks pass or fail
 
Merge to main
  → build & test
  → Docker build
  → Trivy image scan (fail on CRITICAL) ← added Day 49
  → Docker push (only if scan passes)
  → deploy
 
Always active (no workflow needed)
  → GitHub secret scanning              ← enabled Day 49
  → push protection for secrets         ← enabled Day 49
```
Security is now part of your automation — not an afterthought, not a separate process. Every PR and every merge is checked automatically.

## 7. Bonus - Going Further
### Pin Actions to Commit SHAs
Tags like `@v4` can be silently moved by the action author to point at different — potentially malicious — code. Pin to a specific commit SHA instead:
```yaml
# Less secure — tag can be moved
uses: actions/checkout@v4
 
# More secure — pinned to an exact commit
uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11  # v4.1.1
```
This prevents supply chain attacks where a compromised action tag is silently updated with malicious code. Pinning actions is considered best practice in production pipelines.
 
### Upload Trivy Results to the GitHub Security Tab
Output results in SARIF format and upload them to make findings visible in GitHub's native Security dashboard alongside Dependabot alerts:
```yaml
- name: Scan and output SARIF
  uses: aquasecurity/trivy-action@57a97c7e7821a5776cebc9bb87c984fa69cba8f1
  with:
    image-ref: 'atulsharmadochub/myapp:latest'
    format: 'sarif'
    output: 'trivy-results.sarif'

- name: Upload SARIF to GitHub Security tab
  uses: github/codeql-action/upload-sarif@dd903d2e4f5405488e5ef1422510ee31c8b32357
  with:
    sarif_file: 'trivy-results.sarif'
```

```yaml
name: Main Branch Pipeline

on:
  push:
    branches: [ main ]

permissions:                                                # restricts default GITHUB_TOKEN to least privilege
  contents: read                                            # read‑only repo access
  packages: write                                           # needed to push Docker images
  security-events: write                                    # required to upload SARIF results to GitHub Security tab

jobs:
  build-test:
    uses: ./.github/workflows/reusable-build-test.yml
    with:
      python_version: "3.10"
      run_tests: true

  docker-build-push:
    needs: build-test
    uses: ./.github/workflows/reusable-docker.yml
    with:
      image_name: atulsharmadochub/myapp
      tag: latest
    secrets:
      docker_username: ${{ secrets.DOCKER_USERNAME }}
      docker_token: ${{ secrets.DOCKER_TOKEN }}

  docker-build-push-sha:
    needs: build-test
    uses: ./.github/workflows/reusable-docker.yml
    with:
      image_name: atulsharmadochub/myapp
      tag: sha-${{ github.sha }}
    secrets:
      docker_username: ${{ secrets.DOCKER_USERNAME }}
      docker_token: ${{ secrets.DOCKER_TOKEN }}

  trivy-scan:
    runs-on: ubuntu-latest
    needs: docker-build-push
    steps:
      - name: Scan and output SARIF
        uses: aquasecurity/trivy-action@57a97c7e7821a5776cebc9bb87c984fa69cba8f1   # calls official Trivy GitHub Action - pinned it to a full commit SHA instead of a moving tag like @master - this is a supply chain hardening practice - it guarantees we’re running exact code we audited
        with:
          image-ref: 'atulsharmadochub/myapp:latest'
          format: 'sarif'                                                          # outputs results in SARIF format (Static Analysis Results Interchange Format) - SARIF is a standardized JSON schema for security findings - GitHub understands SARIF and can display results in the Security tab
          output: 'trivy-results.sarif'                                            # saves scan results to a file named trivy-results.sarif in workspace - this file is then consumed by next step

      - name: Upload SARIF to GitHub Security tab
        uses: github/codeql-action/upload-sarif@dd903d2e4f5405488e5ef1422510ee31c8b32357   # calls GitHub’s official action to upload SARIF files - pinned to a full commit SHA for immutability
        with:
          sarif_file: 'trivy-results.sarif'                                        # points to SARIF file generated by Trivy in previous step

  deploy:
    runs-on: ubuntu-latest
    needs: [docker-build-push, docker-build-push-sha, trivy-scan]
    environment: production
    steps:
      - name: Deploy to production
        run: echo "Deploying image: ${{ needs.docker-build-push.outputs.image_url }} to production"
```

>Pinned Trivy and CodeQL actions to full commit SHAs, eliminating resolution errors. Configured Trivy to output SARIF and upload results to GitHub Security tab, integrating vulnerability findings into native code scanning alerts.

### Commit and Verify
```bash
git add .github/workflows/main-pipeline.yml
git commit -m "Harden main pipeline: add permissions, pin SHAs, upload Trivy SARIF"
git push origin main
```
- GitHub Actions will immediately pick up the new workflow definition.
- On our next push to `main`, the pipeline will run with:
  - **Reduced permissions** (`contents: read`, `packages: write`, `security-events: write`).
  - **Pinned SHAs** for actions (supply chain hardening).
  - **Trivy SARIF upload** → results will appear in **Security → Code scanning alerts** tab.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/4e4ea43d169749a92825892b1a04749d5db3113d/2026/day-49/Screenshots/Screenshot%20(1402).png)

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/4e4ea43d169749a92825892b1a04749d5db3113d/2026/day-49/Screenshots/Screenshot%20(1404).png)

### Use OIDC for Keyless Cloud Authentication
Instead of storing long-lived cloud credentials in GitHub Secrets, use OIDC (OpenID Connect). GitHub Actions can request short-lived tokens from AWS, GCP, or Azure automatically — no permanent keys stored anywhere.
| | Long-lived Credentials | OIDC |
|---|---|---|
| **Storage** | GitHub Secrets (permanent) | No storage needed |
| **Rotation** | Manual | Automatic — tokens expire after each run |
| **Blast radius if leaked** | Full access until manually revoked | Token already expired |
| **Setup complexity** | Simple | Requires IAM role/trust policy configuration |
 
OIDC is the modern standard for cloud authentication in production CI/CD pipelines.