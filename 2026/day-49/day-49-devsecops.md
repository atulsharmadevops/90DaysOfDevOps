# DevSecOps: Add Security to Our CI/CD Pipeline

## What is DevSecOps?



## Task 1: Scan Your Docker Image for Vulnerabilities
### Locate your workflow file
- Go to our repo: .github/workflows/main.yml (or whatever you named your main branch pipeline).
- This file already has steps for build, test, Docker build, and deploy.
### Insert the Trivy scan step
- Right after our Docker build step, add:
    ```yaml
    - name: Scan Docker Image for Vulnerabilities
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: 'your-username/your-app:latest'
        format: 'table'
        exit-code: '1'
        severity: 'CRITICAL,HIGH'
    ```
- `trivy` pulls vulnerability data (CVEs) and scans our built image.
- `format: 'table'` --> readable output in logs.
- `exit-code: '1'` --> pipeline fails if CRITICAL/HIGH issues are found.
- If clean, pipeline continues to push and deploy.
### Push our changes
- Commit the updated workflow file.
- Push to our repo’s `main` branch.
### Check the Actions tab
- Open your repo --> Actions --> select the latest run.
- Look for the **Trivy scan step** in the logs.
- We’ll see a vulnerability table (if any issues exist).
### Verify results
- If vulnerabilities are found → pipeline fails, logs show CVEs.
- If clean → pipeline passes, Docker image is pushed and deployed.
### What CVEs (if any) were found? What base image are you using?

## Task 2: Enable GitHub's Built-in Secret Scanning
### Open your repository settings
- Go to our repo on GitHub.
- Click **Settings ---> Code security and analysis**.
### Enable Secret Scanning
- Turn on **Secret scanning**.
- This feature continuously scans commits for patterns that look like secrets (API keys, tokens, passwords).
### Enable Push Protection (if available)
- Turn on **Push protection**.
- This prevents a commit containing a secret from being pushed at all.
- If GitHub detects a secret during push, it blocks the push and shows an error message.
### No workflow changes needed, GitHub handles scanning automatically in the background.
### What is the difference between secret scanning and push protection?
- Secret scanning = runs after code is pushed, detects secrets already in the repo.
- Push protection = runs before code is pushed, blocks the commit entirely if a secret is detected.

### What happens if GitHub detects a leaked AWS key in your repo?
- GitHub flags the secret in the Security tab.
- We’ll get an alert (and possibly an email).
- For AWS keys, GitHub often notifies AWS directly, and AWS may automatically revoke the key to prevent misuse.
- We must rotate the secret immediately and remove it from our repo history.

## Task 3: Scan Dependencies for Known Vulnerabilities
### Locate your PR workflow file
- Go to `.github/workflows/pr.yml` (or whichever workflow runs on pull_request events).
- This is separate from our main pipeline.
### Add the dependency review step  
- Insert this snippet after our build/test steps:
    ```yaml
    - name: Check Dependencies for Vulnerabilities
      uses: actions/dependency-review-action@v4
      with:
        fail-on-severity: critical
    ```
- `fail-on-severity: `critical` ---> pipeline fails if a new dependency has a critical CVE.
- Lower severity issues will be reported but won’t block the PR.
### Commit and push the workflow change
- Push to our repo so the updated workflow is active.

### Test the workflow
- Open a new **Pull Request** that adds a dependency (e.g., add a new package in `requirements.txt` or `package.json`).
- This triggers the PR pipeline.
### Check the Actions tab
- Go to our repo --> Actions --> select the PR run.
- Look for the **dependency review step** in the logs.
- We’ll see a report of vulnerabilities (if any).
### Verify results
- If a critical CVE is found --> PR check fails, we must fix/remove the dependency.
- If clean --> PR passes, merge is allowed.

## Task 4: Add Permissions to Your Workflows
By default, workflows get broad permissions. Lock them down.
### Open your workflow files
- Go to `.github/workflows/` in our repo.
- Pick at least **two workflow files** (for example, `main.yml` and `pr.yml`).
### Add a permissions block`
- Place it **right after the `on:` section** at the top of the file.
- Example for a simple workflow:
    ```yaml
    on:
      push:
        branches: [ "main" ]

    permissions:
      contents: read
    ```
    - This restricts the workflow to only read repository contents.
### If our workflow needs to comment on PRs
- Use:
    ```yaml
    permissions:
      contents: read
      pull-requests: write
    ```
    - This gives just enough access to interact with PRs, nothing more.

### Commit and push changes
- Save the updated workflow files.
- Push to our repo.
- The workflows will now run with reduced permissions.
### Why is it a good practice to limit workflow permissions?
- By default, workflows have broad access (read/write to repo).
- Limiting permissions reduces the attack surface if an action is compromised.
### What could go wrong if a compromised action has write access to your repo?
- A malicious or compromised action could:
    - Push unwanted commits to your repo
    - Delete or overwrite files
    - Exfiltrate sensitive data
- With restricted permissions, the damage is contained.

## Task 5: See the Full Secure Pipeline
Look at what your pipeline does now:

PR opened
  → build & test
  → dependency vulnerability check     ← NEW (Day 49)
  → PR checks pass or fail

Merge to main
  → build & test
  → Docker build
  → Trivy image scan (fail on CRITICAL) ← NEW (Day 49)
  → Docker push (only if scan passes)
  → deploy

Always active
  → GitHub secret scanning              ← NEW (Day 49)
  → push protection for secrets         ← NEW (Day 49)
Draw this diagram in your notes. You just built a DevSecOps pipeline — security is now part of your automation, not an afterthought.

## Brownie Points (Optional — For the Curious)
Pin Actions to Commit SHAs
Tags like @v4 can be moved by the action author. For extra security, pin to the exact commit:

# Instead of this:
uses: actions/checkout@v4

# Use this:
uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11 # v4.1.1
This protects against supply chain attacks where a tag is silently changed.

Upload Scan Results to GitHub Security Tab
Add SARIF output to Trivy and upload it — your scan results will appear in the repo's Security tab:

- uses: aquasecurity/trivy-action@master
  with:
    image-ref: 'your-username/your-app:latest'
    format: 'sarif'
    output: 'trivy-results.sarif'
- uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: 'trivy-results.sarif'
Learn About OIDC (Keyless Authentication)
Instead of storing cloud credentials as long-lived secrets, GitHub Actions can use OIDC to get short-lived tokens automatically. Research: "GitHub Actions OIDC" — it's how production pipelines authenticate to AWS, GCP, and Azure without storing any keys.