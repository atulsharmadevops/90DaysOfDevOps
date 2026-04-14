# GitHub Actions Runners: GitHub-Hosted & Self-Hosted
## Task 1: GitHub-Hosted Runners
### Workflow with 3 Jobs (Different OS)
- Create a workflow file `.github/workflows/hosted-runners.yml`:
    ```yaml
    name: GitHub-Hosted Runners Demo

    on: [push]

    jobs:
      ubuntu:
        runs-on: ubuntu-latest
        steps:
          - name: Print system info
            run: |
              echo "OS: Ubuntu"
              hostname
              whoami

      windows:
        runs-on: windows-latest
        steps:
          - name: Print system info
            run: |
              echo "OS: Windows"
              hostname
              whoami

      macos:
        runs-on: macos-latest
        steps:
          - name: Print system info
            run: |
              echo "OS: macOS"
              hostname
              whoami
    ```
    - When we push this workflow, GitHub will spin up three separate runners (Ubuntu, Windows, macOS) and execute the jobs in parallel.

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/8de6fdf83a4b49d22acfb42893a7f52e8e49ae72/2026/day-42/Screenshots/Screenshot%20(347).png)

### What is a GitHub-hosted runner?
- A GitHub-hosted runner is a virtual machine provided by GitHub to execute our workflow jobs.
- It is ephemeral: created fresh for each job, and destroyed after the job finishes.
### Who manages it?
- GitHub manages everything: provisioning, OS updates, pre-installed tools, scaling, and cleanup.
- We don’t need to maintain or secure the machine — GitHub takes care of it.

## Task 2: Explore What's Pre-installed
### Workflow Step to Print Versions
- Add this to our workflow file:
    ```yaml
    - name: Print versions
    run: |
        docker --version
        python3 --version
        node --version
        git --version
    ```
    ```yaml
    name: GitHub-Hosted Runners Demo

    on: [push]

    jobs:
      ubuntu:
        runs-on: ubuntu-latest
        steps:
          - name: Print system info
            run: |
              echo "OS: Ubuntu"
              hostname
              whoami
          - name: Print versions
            run: |
              docker --version
              python3 --version
              node --version
              git --version

      windows:
        runs-on: windows-latest
        steps:
          - name: Print system info
            run: |
              echo "OS: Windows"
              hostname
              whoami

      macos:
        runs-on: macos-latest
        steps:
          - name: Print system info
            run: |
              echo "OS: macOS"
              hostname
              whoami
    ```
    - When we run this job on ubuntu-latest, the logs will show the installed versions of Docker, Python, Node.js, and Git.

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/8de6fdf83a4b49d22acfb42893a7f52e8e49ae72/2026/day-42/Screenshots/Screenshot%20(349).png)

### Pre-installed Software on `ubuntu-latest`
According to GitHub’s official **runner images documentation**, the `ubuntu-latest` runner comes with a wide range of tools already installed, including:
- **Languages:** Python, Node.js, Ruby, PHP, Java, Go, .NET, PowerShell
- **Package managers:** npm, pip, yarn, Maven, Gradle
- **Containers & virtualization:** Docker, Podman, Buildah
- **Version control:** Git, Subversion
- **Build tools:** CMake, Make, GCC, Clang
- **Cloud CLIs:** AWS CLI, Azure CLI, Google Cloud SDK
- **Other utilities:** curl, wget, jq, zip/unzip, tar, rsync

This list is regularly updated by GitHub to ensure compatibility with modern workflows.


### Why does it matter that runners come with tools pre-installed?
- **Saves setup time:** We don’t need to install Docker, Python, Node, or Git manually in every workflow.
- **Consistency:** Ensures all jobs run on a predictable environment with known versions.
- **Reliability:** Reduces risk of build failures due to missing dependencies.
- **Speed:** Faster CI/CD pipelines since installation steps are skipped.
- **Standardization:** Makes workflows portable and easier to share across teams.

## Task 3: Set Up a Self-Hosted Runner
### Step 1: Navigate to Runner Settings
- Go to our GitHub repository.
- Click **Settings --> Actions --> Runners --> New self-hosted runner**.

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/8de6fdf83a4b49d22acfb42893a7f52e8e49ae72/2026/day-42/Screenshots/Screenshot%20(351).png)

- Select **Linux** as the operating system.

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/8de6fdf83a4b49d22acfb42893a7f52e8e49ae72/2026/day-42/Screenshots/Screenshot%20(353).png)

GitHub will generate a setup script for us — copy it.
### Step 2: Download and Configure Runner
On our Linux machine:

Download
```bash
# Create a folder
$ mkdir actions-runner && cd actions-runner

#Download the latest runner package
$ curl -o actions-runner-linux-x64-2.333.1.tar.gz -L https://github.com/actions/runner/releases/download/v2.333.1/actions-runner-linux-x64-2.333.1.tar.gz

# Optional: Validate the hash
$ echo "18f8f68ed1892854ff2ab1bab4fcaa2f5abeedc98093b6cb13638991725cab74  actions-runner-linux-x64-2.333.1.tar.gz" | shasum -a 256 -c

# Extract the installer
$ tar xzf ./actions-runner-linux-x64-2.333.1.tar.gz
```
Configure
```bash
# Create the runner and start the configuration experience
$ ./config.sh --url https://github.com/AtulSharmaGeit/github-actions-practice --token BQ5Q452R2BVQOMN63HNM4LDJ3TD5C
```

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/8de6fdf83a4b49d22acfb42893a7f52e8e49ae72/2026/day-42/Screenshots/Screenshot%20(354).png)

```bash
# Last step, run it!
$ ./run.sh
```

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/8de6fdf83a4b49d22acfb42893a7f52e8e49ae72/2026/day-42/Screenshots/Screenshot%20(359).png)

Using our self-hosted runner
```bash
# Use this YAML in your workflow file for each job
runs-on: self-hosted
```
### Step 4: Verify in GitHub
- Go back to **Settings --> Actions --> Runners**.
- We should see our runner listed with a **green dot** and status **Idle**.

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/8de6fdf83a4b49d22acfb42893a7f52e8e49ae72/2026/day-42/Screenshots/Screenshot%20(356).png)

- That means it’s ready to pick up jobs.

### What is Self-Hosted Runner?
- A self-hosted runner is a machine we manage yourself (local or cloud).
- We are responsible for installing dependencies, keeping it secure, and maintaining uptime.
- Once registered, it appears in our repo’s runner list and can be targeted in workflows with runs-on: self-hosted.

## Task 4: Use Your Self-Hosted Runner
### Workflow File: `.github/workflows/self-hosted.yml`
```yaml
name: Self-Hosted Runner Demo

on: [push]

jobs:
  demo:   # <-- this is the job identifier
    runs-on: self-hosted
    steps:
      - name: Print hostname
        run: hostname

      - name: Print working directory
        run: pwd

      - name: Create a file
        run: echo "Hello from my self-hosted runner!" > runner-test.txt

      - name: Verify file exists
        run: ls -l runner-test.txt
```
- **Hostname** - Prints the actual hostname of our machine/VM (not GitHub’s infra).
- **Working directory** - Shows the runner’s working directory (usually inside `/home/atulsharma/actions-runner/_work/github-actions-practice/github-actions-practice`).
- **File creation** - Creates `runner-test.txt` on our machine.
- **Verification step** - Lists the file to confirm it exists.

### Verification
- Push this workflow to our repo.
- Watch the job logs in GitHub Actions - they should show our machine’s hostname and working directory.

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/8de6fdf83a4b49d22acfb42893a7f52e8e49ae72/2026/day-42/Screenshots/Screenshot%20(360).png)

- Log in to our machine/VM and check the runner’s working directory. We should see `runner-test.txt` created there.

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/8de6fdf83a4b49d22acfb42893a7f52e8e49ae72/2026/day-42/Screenshots/Screenshot%20(362).png)

### What Happened?
- Running a workflow on a **self-hosted runner** proves that jobs can execute directly on our hardware.
- We can interact with files, directories, and installed tools specific to our environment.
- This is useful for custom setups (e.g., GPU builds, private networks, or specialized dependencies).

## Task 5: Labels
### Step 1: Add a Label to Your Runner
- When configuring our self-hosted runner, We can add labels. For example:
    ```bash
    ./config.sh --url https://github.com/AtulSharmaGeit/github-actions-practice --token BQ5Q452R2BVQOMN63HNM4LDJ3TD5C \
    --labels my-linux-runner
    ```
    - This registers our runner with the label `my-linux-runner`.
### Step 2: Update Workflow to Use Labels
- Modify our workflow to target both `self-hosted` and our custom label:
    ```yaml
    name: Self-Hosted Runner with Label

    on: [push]

    jobs:
      demo:
        runs-on: [self-hosted, my-linux-runner]
        steps:
          - name: Print hostname
            run: hostname

          - name: Print working directory
            run: pwd

          - name: Create file
            run: echo "Hello from labeled runner!" > runner-labeled.txt

          - name: Verify file exists
            run: ls -l runner-labeled.txt
    ```
    - When we push this workflow, GitHub will assign the job to our runner that matches both `self-hosted` and `my-linux-runner`.
### Step 3: Verify
- Trigger the workflow with a push.
- Check the Actions logs - it should run on our machine.
- On our machine, verify that `runner-labeled.txt` exists in the runner’s working directory.

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/8de6fdf83a4b49d22acfb42893a7f52e8e49ae72/2026/day-42/Screenshots/Screenshot%20(364).png)

### Why are labels useful when you have multiple self-hosted runners?
- Labels let us **target specific runners** (e.g., Linux vs Windows, GPU-enabled vs CPU-only).
- They help organize runners by **capabilities** (e.g., `docker`, `gpu`, `arm64`).
- They prevent jobs from accidentally running on the wrong machine.
- They make scaling easier when we have a fleet of self-hosted runners.

## Task 6: GitHub-Hosted vs Self-Hosted
| Feature | GitHub-Hosted | Self-Hosted |
|---|---|---|
| **Who manages it?** | GitHub (fully managed) | You (your team/infra) |
| **Cost** | Free tier + paid minutes beyond quota | Free to run; you pay for the hardware/VM |
| **Pre-installed tools** | Extensive (Docker, Node, Python, Git, etc.) | Only what you install yourself |
| **Good for** | Open source, standard CI/CD, quick setup | Long jobs, custom hardware, private networks, GPU workloads |
| **Security concern** | Isolated, fresh VM per job — very safe | Persistent machine; malicious PRs could access your environment |
| **Startup time** | ~30–60 seconds (VM provisioning) | Near-instant (runner already running) |
| **Customization** | Limited to what GitHub provides | Full control — install anything |
| **Persistence** | No — clean slate every run | Yes — files persist between runs (can be good or bad) |
 

 
## Key Takeaways
 
- **GitHub-hosted runners** are plug-and-play, ephemeral VMs managed entirely by GitHub — ideal for most projects.
- **Self-hosted runners** give you full control — your hardware, your network, your tools — but require setup and maintenance.
- Use **labels** to route jobs to specific runners when you have a fleet.
- For public repos using self-hosted runners, always be cautious — restrict which workflows can trigger on self-hosted runners to prevent code execution attacks.