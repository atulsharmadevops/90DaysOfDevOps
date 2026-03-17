# GITHUB CLI: MANAGE GITHUB FROM TERMINAL

## Task 1: Install and Authenticate
1. Install GitHub CLI
    ```bash
    sudo apt update
    sudo apt install gh
    ```
    After installation, check the version:
    ```bash
    gh --version
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/da799ad587e79366c4cf0cca146256be60e89b17/2026/day-26/Screenshots/Screenshot%20(22).png)

2. Authenticate with GitHub
    ```bash
    gh auth login
    ```
    - GitHub.com  or GitHub Enterprise → select GitHub.com.
    - Preferred protocol → HTTPS or SSH.
    - Authentication method → options include:
        - Web-based login (browser OAuth flow) → opens a browser window to sign in.
        - Paste a Personal Access Token (PAT) → if you already have one.
        - SSH key authentication → if you’ve configured SSH keys with GitHub.

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/da799ad587e79366c4cf0cca146256be60e89b17/2026/day-26/Screenshots/Screenshot%20(23).png)

3. Verify Login & Active Account
    ```bash
    gh auth status
    ```

    ![immage alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/da799ad587e79366c4cf0cca146256be60e89b17/2026/day-26/Screenshots/Screenshot%20(24).png)

    - This will show:
        - Which GitHub account is logged in.
        - Whether HTTPS or SSH is being used.
        - The active hostname (usually github.com).

## Task 2: Working with Repositories
1. Create a New Repo (Public with README)
    ```bash
    gh repo create my-test-repo --public --confirm --add-readme
    ```
    - `--public` - makes it public
    - `--confirm` - skips interactive prompts
    - `--add-readme` - initializes with a README file

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/da799ad587e79366c4cf0cca146256be60e89b17/2026/day-26/Screenshots/Screenshot%20(26).png)

2. Clone a Repo Using `gh`
    ```bash
    gh repo clone AtulSharmaGeit/my-test-repo
    ```

    ![immage alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/da799ad587e79366c4cf0cca146256be60e89b17/2026/day-26/Screenshots/Screenshot%20(28).png)

3. View Repo Details
    ```bash
    gh repo view AtulSharmaGeit/my-test-repo
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/da799ad587e79366c4cf0cca146256be60e89b17/2026/day-26/Screenshots/Screenshot%20(29).png)

    - Shows description, default branch, visibility, etc.
    - Add `--web` to open in browser.
    - Add `--`json` for machine-readable output.
4. List All Repositories
    ```bash
    gh repo list AtulSharmaGeit
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/da799ad587e79366c4cf0cca146256be60e89b17/2026/day-26/Screenshots/Screenshot%20(30).png)

5. Open Repo in Browser
    ```bash
    gh repo view AtulSharmaGeit/my-test-repo --web
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/da799ad587e79366c4cf0cca146256be60e89b17/2026/day-26/Screenshots/Screenshot%20(31).png)

6. Delete the Test Repo
    ```bash
    gh repo delete my-test-repo
    ```
    - CLI will ask for confirmation.
    - Add --yes to skip the prompt (use cautiously).

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/da799ad587e79366c4cf0cca146256be60e89b17/2026/day-26/Screenshots/Screenshot%20(32).png)

## Task 3: Issues<br>
**GitHub Issues are a built‑in tool for tracking tasks, bugs, feature requests, and project work. Each issue acts like a lightweight ticket that can be assigned, labeled, discussed, and linked to code changes, helping teams stay organized and collaborate effectively.**

1. Create an Issue
    ```bash
    gh issue create --title "Bug in login flow" --body "Steps to reproduce: 1. Go to login page 2. Enter credentials 3. Error appears" --label bug --repo AtulSharmaGeit/devops-git-practice
    ```
    - `--title` - short summary of the issue
    - `--body` - detailed description
    - `--label` - assign a label (e.g., bug, enhancement, documentation)
    - We can also add `--repo owner/repo` if we’re not inside the repo directory.

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/da799ad587e79366c4cf0cca146256be60e89b17/2026/day-26/Screenshots/Screenshot%20(33).png)

2. List All Open Issues
    ```bash
    gh issue list --repo AtulSharmaGeit/devops-git-practice
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/da799ad587e79366c4cf0cca146256be60e89b17/2026/day-26/Screenshots/Screenshot%20(34).png)


    - Shows issue number, title, labels, and status.
    - Add `--state all` to see both open and closed issues.
    - Add `--json` for machine-readable output (useful in scripts).

3. View a Specific Issue
    ```bash
    gh issue view 12
    ```
    - Replace `12` with the issue number.
    - Shows full details: title, body, labels, assignees, comments.
    - Add `--web` to open it in browser.

4. Close an Issue
    ```bash
    gh issue close 1 --repo AtulSharmaGeit/devops-git-practice --comment "Fixed in PR #34"
    ```
    - Closes issue number `1`.
    - Add `--comment "Fixed in PR #34"` to leave a closing note.

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/da799ad587e79366c4cf0cca146256be60e89b17/2026/day-26/Screenshots/Screenshot%20(35).png)

5. How could we use `gh` issue in a script or automation?<br>
**Ans.**
    - **Auto‑create issues from CI failures** - If a build or test fails, you can automatically create an issue:
        ```bash
        gh issue create --title "CI Failure: Build #123" \
        --body "Build failed on commit abc123. Logs attached." \
        --label ci-failure
        ```   
        - This ensures every failure is tracked without manual effort.

    - **Generate reports of open issues** - We can list issues in JSON format and parse them with `jq`:
        ```bash
        gh issue list --state open --json number,title,labels \
        | jq '.[] | {id: .number, title: .title, labels: .labels}'
        ```
        - This can be scheduled to produce daily reports or dashboards.

    - **Close stale issues automatically** - Combine with shell scripting to close issues older than X days:
        ```bash
        for issue in $(gh issue list --state open --json number,createdAt \
        | jq -r '.[] | select(.createdAt < "2026-02-01T00:00:00Z") | .number'); do
        gh issue close $issue --comment "Closing stale issue"
        done
        ```
    -  **Trigger workflows based on issues** - We can use `gh issue view` inside scripts to check labels or status, then trigger different actions (e.g., run a test suite if an issue is labeled `bug`).

## Task 4: Pull Requests
1. Create a Branch, Make a Change, Push It
    - Create and switch to a new branch
    ```
    git checkout -b feature-branch
    ```
    - Make changes to your files
    ```
    echo "New feature line" >> README.md
    ```
    - Stage and commit
    ```
    git add README.md
    git commit -m "Add new feature line"
    ```
    - Push branch to GitHub
    ```
    git push -u origin feature-branch
    ```
2. Create a Pull Request
    ```bash
    gh pr create --fill
    ```
    - `--fill` - auto‑fills PR title and body from commit messages.
    
    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/da799ad587e79366c4cf0cca146256be60e89b17/2026/day-26/Screenshots/Screenshot%20(36).png)

    We can also specify:
    ```bash
    gh pr create --title "Add new feature" --body "This PR adds..."
    ```
3. List All Open PRs
    ```bash
    gh pr list
    ```
    - Shows PR number, title, author, and status.
    - Add `--json` for machine‑readable output.

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/da799ad587e79366c4cf0cca146256be60e89b17/2026/day-26/Screenshots/Screenshot%20(37).png)

4. View PR Details
    ```bash
    gh pr view 2
    ```
    - Replace 5 with your PR number.
    - Shows description, status, reviewers, checks.
    - Add `--web` to open in browser.
    - Add `--json` to parse in scripts.

5. Merge Your PR
    ```bash
    gh pr merge 2 --merge
    ```
    - Merge methods supported:
        - `--merge` - standard merge commit
        - `--squash` - squash commits into one
        - `--rebase` - rebase and merge

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/da799ad587e79366c4cf0cca146256be60e89b17/2026/day-26/Screenshots/Screenshot%20(38).png)

    Example:
    ```bash
    gh pr merge 2 --squash --delete-branch
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/da799ad587e79366c4cf0cca146256be60e89b17/2026/day-26/Screenshots/Screenshot%20(39).png)

    - Squashes commits and deletes the branch after merge.

6. What merge methods does `gh pr merge` support?<br>
**Ans.** When we merge a pull request with GitHub CLI, we can choose one of three methods:
    - `--merge` - Creates a standard merge commit (preserves commit history).
    - `--squash` - Squashes all commits into a single commit on the base branch.
    - `--rebase` - Rebases commits from the PR branch onto the base branch (linear history).
    
    We can also add flags like `--delete-branch` to automatically remove the feature branch after merging.

7. How would we review someone else's PR using `gh`?<br>
**Ans.**
- Check out the PR locally
    ```bash
    gh pr checkout <number>
    ```
    - This fetches the PR branch so you can test the changes.
- Leave a review
    ```bash
    gh pr review <number> --approve
    gh pr review <number> --request-changes --body "Please fix the login bug."
    gh pr review <number> --comment --body "Looks good, but consider updating docs."
    ```
    - `--approve` - approves the PR.
    - `--request-changes` - asks for modifications.
    - `--`comment` - leaves feedback without changing approval status.

- View details before reviewing
    ```bash
    gh pr view <number>
    ```
    - This shows description, status, reviewers, and checks.

## Task 5: GitHub Actions & Workflows
1. List the workflow runs on any public repo that uses GitHub Actions
    ```bash
    gh run list
    ```
    - Shows recent workflow runs (ID, status, conclusion, event type).

    We can also target a specific repo:
    ```bash
    gh run list --repo owner/repo-name
    ```

2. View the status of a specific workflow run
    ```bash
    gh run view <run-id>
    ```
    - Replace `<run-id>` with the ID from the list above.
    - Shows detailed info: job steps, logs, conclusion, duration.
    - Add `--web` to open the run in your browser.
    - Add `--json` for machine‑readable output (great for scripting).

3. How could `gh run` and `gh workflow` be useful in a CI/CD pipeline?<br>
**Ans.**
- Using `gh run` in CI/CD
    - Monitor builds/tests directly from terminal
        ```bash
        gh run list --repo owner/repo
        ```
        Quickly see which runs are in progress, failed, or succeeded.

    - View detailed logs for debugging
        ```bash
        gh run view <run-id>
        ```
        Inspect job steps, errors, and conclusions without opening the browser.

    - Automate reruns
        ```bash
        gh run rerun <run-id>
        ```
        Useful for automatically retrying flaky jobs in scripts.

- Using `gh workflow` in CI/CD
    - List workflows in a repo
        ```bash
        gh workflow list
        ```
        Helps identify which pipelines exist (e.g., build, deploy, test).

    - Enable/disable workflows
        ```bash
        gh workflow disable <workflow-id>
        gh workflow enable <workflow-id>
        ```
        Control which pipelines run, useful for maintenance windows.

    - Trigger workflows manually
        ```bash
        gh workflow run <workflow-id>
        ```
        Start a pipeline on demand (e.g., redeploy production).

##  Task 6: Useful `gh` Tricks
1. `gh api` — Raw GitHub API Calls<br>
    Make direct API requests without leaving your terminal:
    ```bash
    gh api /repos/owner/repo/issues --method GET --paginate
    ```
    - Useful for scripting when you need data not exposed by standard `gh` commands.
    - Combine with `jq` for parsing JSON output.

2. `gh gist` — Create & Manage Gists<br>
    Create a new gist from a file:
    ```bash
    gh gist create script.sh --public
    ```
    List your gists:
    ```bash
    gh gist list
    ```
    View a gist:
    ```bash
    gh gist view <gist-id>
    ```
    Great for sharing snippets or cheat sheets.

3. `gh release` — Manage Releases<br>
    Create a release:
    ```bash
    gh release create v1.0.0 --notes "Initial release"
    ```
    List releases:
    ```bash
    gh release list
    ```
    Upload assets to a release:

    ```bash
    gh release upload v1.0.0 build.zip
    ```
    Perfect for packaging and distributing builds in CI/CD.

4. `gh alias` — Shortcuts for Frequent Commands<br>
    Set an alias:
    ```bash
    gh alias set prc "pr create --fill"
    ```
    Now you can run:
    ```bash
    gh prc
    ```
    Saves time when you repeat commands often.

5. `gh search repos` — Search GitHub Repos<br>
    Search repos by topic:
    ```bash
    gh search repos --topic devops --limit 10
    ```
    Search repos by language:
    ```bash
    gh search repos --language python --limit 5
    ```
    Useful for discovering projects or inspiration directly from terminal.

