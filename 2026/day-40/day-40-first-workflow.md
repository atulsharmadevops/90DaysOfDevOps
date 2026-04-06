# First GitHub Actions Workflow

## Task 1: Set Up
1. Create a new public repository
    ```bash
    gh repo create github-actions-practice --public
    ```

2. Clone it locally
    ```bash
    git clone https://github.com/<your-username>/github-actions-practice.git

    cd github-actions-practice
    ```
3. Create the folder structure
    ```bash
    mkdir -p .github/workflows
    ```
    - `.github/` is a special directory recognized by GitHub.
    - `workflows/` is where all your pipeline `.yml` files live.

4. Verify structure
    ```bash
    tree .github
    ```
    Expected output:

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/404031dc9635c2fdfc71c825e062824e3ee7ced8/2026/day-40/Screenshots/Screenshot%20(291).png)

## Task 2: Hello Workflow
1. Create the Workflow File

    Inside our repo, create the file `.github/workflows/hello.yml` with the following content:
    ```yaml
    name: Hello Workflow

    on:
      push:   # Trigger on every push

    jobs:
      greet:
        runs-on: ubuntu-latest   # Runner environment
        steps:
          - name: Checkout code
            uses: actions/checkout@v4

          - name: Say hello
            run: echo "Hello from GitHub Actions!"
    ```
2. Push and Run

    - Save the file.

    - Stage and commit:
        ```bash
        git add .github/workflows/hello.yml

        git commit -m "Add hello workflow"

        git push origin main
        ```
    - Go to our repository on **GitHub → Actions** tab.

    - We’ll see a new workflow run triggered by our push.

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/404031dc9635c2fdfc71c825e062824e3ee7ced8/2026/day-40/Screenshots/Screenshot%20(292).png)

3. Verify
    - The workflow should show a **green checkmark** if successful.
    - Click into the run → open the **greet job**.
    - We’ll see two steps:
        - Checkout code (using `actions/checkout@v4`)
        - Say hello (prints `Hello from GitHub Actions!`)

      ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/404031dc9635c2fdfc71c825e062824e3ee7ced8/2026/day-40/Screenshots/Screenshot%20(293).png)

## Task 3: Understand the Anatomy
 
| Key | What it does |
|-----|-------------|
| `on:` | Defines the **trigger** — what event causes this workflow to run. `push` means it runs every time code is pushed to any branch. You can also trigger on `pull_request`, `schedule`, `workflow_dispatch`, etc. |
| `jobs:` | Contains all the **jobs** in the workflow. A workflow can have one or many jobs. Each job runs independently (on its own runner) unless you declare dependencies with `needs:`. |
| `runs-on:` | Tells GitHub **which type of machine (runner)** to use for this job. `ubuntu-latest` means the latest Ubuntu Linux runner provided by GitHub. |
| `steps:` | The **ordered list of actions** within a job. Each step runs sequentially on the same runner. If one step fails, subsequent steps are skipped (by default). |
| `uses:` | Runs a **pre-built Action** from the GitHub Marketplace or a repository. Example: `actions/checkout@v4` checks out your repository code onto the runner so later steps can use it. |
| `run:` | Executes a raw **shell command** on the runner. Anything you'd type in a terminal goes here. Multi-line commands use the `|` (pipe/block scalar) syntax. |
| `name:` (on a step) | A **human-readable label** for the step. It shows up in the Actions UI, making it easy to identify which step did what when reading logs. |
 
## Task 4: Add More Steps
1. Updated `.github/workflows/hello.yml`
    ```yaml
    name: Hello Workflow

    on:
      push:   # Trigger on every push

    jobs:
      greet:
        runs-on: ubuntu-latest
        steps:
          - name: Checkout code
            uses: actions/checkout@v4

          - name: Say hello
            run: echo "Hello from GitHub Actions!"

          - name: Print date and time
            run: date

          - name: Print branch name
            run: 'echo "Branch: ${{ github.ref_name }}"'

          - name: List repo files
            run: ls -la

          - name: Print runner OS
            run: uname -a
    ```

2. Push and Watch
- Save the file.
- Commit and push:
    ```bash
    git add .github/workflows/hello.yml
    git commit -m "Add extra steps to hello workflow"
    git push origin main
    ```
- Go to the Actions tab in your repo.
- Watch the new run — you’ll see each step executed in sequence.

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/404031dc9635c2fdfc71c825e062824e3ee7ced8/2026/day-40/Screenshots/Screenshot%20(299).png)

3. Verification
- Green checkmark → pipeline succeeded.
- Inside the job logs, you’ll see:
    - The greeting message.
    - Current date/time.
    - Branch name (from `${{ github.ref_name }}`).
    - List of repo files.
    - Runner OS details.

  ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/404031dc9635c2fdfc71c825e062824e3ee7ced8/2026/day-40/Screenshots/Screenshot%20(300).png)

## Task 5: Break It On Purpose
1. Update Workflow to Fail

    Add a failing step to `.github/workflows/hello.yml`:
    ```yaml
      - name: Print branch name
        run: echo "Branch: ${{ github.ref_name }}"
    ```

2. Push and Observe
    - Commit and push:
        ```bash
        git add .github/workflows/hello.yml
        git commit -m "Add failing step"
        git push origin main
        ```
    - Go to the Actions tab.
    - We’ll see the workflow run turn red instead of green.

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/404031dc9635c2fdfc71c825e062824e3ee7ced8/2026/day-40/Screenshots/Screenshot%20(295).png)

    - Click into the run ---> open the greet job ---> the failing step will have a ❌ mark.

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/404031dc9635c2fdfc71c825e062824e3ee7ced8/2026/day-40/Screenshots/Screenshot%20(297).png)
    
3. Fix It
    - Remove or correct the failing command.
    - Push again.
    - The pipeline should return to green.

4. What does a failed pipeline look like?<br>
**Ans.**
    - The overall workflow shows a red X instead of a green check.
    - The failed step is highlighted with ❌.
    - Steps after the failure are skipped (unless we configure continue-on-error).

5. How do you read the error?<br>
**Ans.**
    - Click into the failed job.
    - Expand the failing step.
    - The logs show the exact command that failed and the error message.
    - Example: command not found or Process completed with exit code 1.