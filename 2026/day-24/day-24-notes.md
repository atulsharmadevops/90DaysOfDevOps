# ADVANCED GIT: MERGE, REBASE, STASH & CHERRY PICK

## Task 1: Git Merge
1. Start from main
    ```
    git checkout main
    ```
2. Create new branch
    ```
    git checkout -b feature-login
    ```
3. Make some changes, then commit
    ```
    echo "Login page code" >> login.html
    git add login.html
    git commit -m "Add login page"
    ```
    ```
    echo "Login validation" >> login.html
    git add login.html
    git commit -m "Add login validation"
    ```
    
    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/9d40fbf2c69a4678dd1319509810a68731fb6141/2026/day-24/Screenshots/Screenshot%20(1).png)

4. Switch back to main
    ```
    git checkout main
    ```
5. Merge feature-login into main
    ```
    git merge feature-login
    ```
    Output:

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/9d40fbf2c69a4678dd1319509810a68731fb6141/2026/day-24/Screenshots/Screenshot%20(2).png)

    Since `main` had no new commits after branching, Git will do a **fast‑forward merge**.

6. Create new branch from main
    ```
    git checkout -b feature-signup
    ```
7. Make some changes, then commit
    ```
    echo "Signup page code" >> signup.html
    git add signup.html
    git commit -m "Add signup page"
    ```
    ```
    echo "Signup validation" >> signup.html
    git add signup.html
    git commit -m "Add signup validation"
    ```
8. Switch back to main
    ```
    git checkout main
    ```
9. Add a new commit directly on main
    ```
    echo "Update README" >> README.md
    git add README.md
    git commit -m "Update README on main"
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/9d40fbf2c69a4678dd1319509810a68731fb6141/2026/day-24/Screenshots/Screenshot%20(3).png)

10. Merge feature-signup into main
    ```
    git merge feature-signup
    ```
    Output:

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/9d40fbf2c69a4678dd1319509810a68731fb6141/2026/day-24/Screenshots/Screenshot%20(4).png)

    This time, Git cannot **fast‑forward** because `main` has diverged and our log will show a **merge commit**:
    ```bash
    git log --oneline --graph --all
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/9d40fbf2c69a4678dd1319509810a68731fb6141/2026/day-24/Screenshots/Screenshot%20(5).png)

11. What is a fast-forward merge?

    **Fast-forward merge** happens when the target branch (main) has not moved ahead since the feature branch was created. Git simply moves the pointer forward — no new commit is created.
12. When does Git create a merge commit instead?
    
    **Merge commit** is created when both branches have diverged (main has new commits). Git creates a special commit that ties histories together.
13. What is a merge conflict?

    **Merge conflict** occurs when the same line of a file is changed differently in two branches. Git cannot decide automatically, so you must resolve manually.

## Task 2: Git Rebase

**Rebase** means taking the commits from your branch and replaying them on top of another branch’s latest state, creating a cleaner, linear history instead of a merge commit. It’s often used to keep feature branches up to date with main before merging.
1. Start from main
    ```
    git checkout main
    ```
2. Create new branch
    ```
    git checkout -b feature-dashboard
    ```
3. Add a couple of commits
    ```
    echo "Dashboard layout" >> dashboard.html
    git add dashboard.html
    git commit -m "Add dashboard layout"
    ```
    ```
    echo "Dashboard charts" >> dashboard.html
    git add dashboard.html
    git commit -m "Add dashboard charts"
    ```
4. Switch back to main
    ```
    git checkout main
    ```
5. Add a new commit directly on main
    ```
    echo "Update README" >> README.md
    git add README.md
    git commit -m "Update README on main"
    ```
6. Switch back to feature-dashboard
    ```
    git checkout feature-dashboard
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/9d40fbf2c69a4678dd1319509810a68731fb6141/2026/day-24/Screenshots/Screenshot%20(7).png)

7. Rebase onto main
    ```
    git rebase main
    ```
  - Git takes the commits from `feature-dashboard` and **replays them on top of the latest `main` commit**.

  - If there are conflicts, Git will pause and ask you to resolve them before continuing (`git rebase --continue`).

  - After rebase, your branch history looks **linear** — as if you created your dashboard commits after the README update on `main`.
    
    **Compare Logs**
    ```bash
    git log --oneline --graph --all
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/9d40fbf2c69a4678dd1319509810a68731fb6141/2026/day-24/Screenshots/Screenshot%20(6).png)

    - **With merge:** You’d see a branching structure and a merge commit tying them together.

    - **With rebase:** You’ll see a straight line — no merge commit, just your feature commits stacked after main.
8. What does rebase actually do to your commits?<br>
    **Ans.** It takes commits from your branch and re-applies them on top of another branch (like replaying history).
9. How is the history different from a merge?<br>
    **Ans.** Merge preserves both histories with a merge commit; rebase rewrites history to look linear.
10. Why should you never rebase commits that have been pushed and shared with others?<br>
    **Ans.** Because rewriting history breaks others’ clones and causes confusion.
11. When would you use rebase vs merge?<br>
    **Ans.** Use cases:<br>
     Merge: when you want to preserve history of collaboration.<br>
    Rebase: when you want a clean, linear history for readability.

## Task 3: Squash Commit vs Merge Commit
1. Start from main
    ```
    git checkout main
    ```
2. Create new branch
    ```
    git checkout -b feature-profile
    ```
3. Add several small commits
    ```
    echo "Profile page" >> profile.html
    git add profile.html
    git commit -m "Add profile page"
    ```
    ```
    echo "Fix typo" >> profile.html
    git add profile.html
    git commit -m "Fix typo in profile"
    ```
    ```
    echo "Formatting changes" >> profile.html
    git add profile.html
    git commit -m "Update formatting"
    ```
    ```
    echo "Minor CSS tweak" >> profile.html
    git add profile.html
    git commit -m "CSS tweak for profile"
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/9d40fbf2c69a4678dd1319509810a68731fb6141/2026/day-24/Screenshots/Screenshot%20(8).png)

4. Switch back to main
    ```
    git checkout main
    ```
5. Squash merge
    ```
    git merge --squash feature-profile
    ```
6. Commit the squashed changes
    ```
    git commit -m "Add profile feature (squashed)"
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/9d40fbf2c69a4678dd1319509810a68731fb6141/2026/day-24/Screenshots/Screenshot%20(10).png)

    Observation:
    - All 4–5 commits from `feature-profile` are combined into **one single commit** on `main`.
    - `git log` will show only **one new commit** added to `main`.

7. Create new branch
    ```
    git checkout -b feature-settings
    ```
8. Add a few commits
    ```
    echo "Settings page" >> settings.html
    git add settings.html
    git commit -m "Add settings page"
    ```
    ```
    echo "Settings validation" >> settings.html
    git add settings.html
    git commit -m "Add settings validation"
    ```
    ```
    echo "Settings CSS" >> settings.html
    git add settings.html
    git commit -m "Add CSS for settings"
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/9d40fbf2c69a4678dd1319509810a68731fb6141/2026/day-24/Screenshots/Screenshot%20(12).png)

9. Switch back to main
    ```
    git checkout main
    ```
10. Regular merge
    ```
    git merge feature-settings
    ```
    Observation:
    - Git preserves all commits from `feature-settings`.
    - `git log` will show **all 3 commits** plus a **merge commit** tying them together.
    - History looks branched and merged, not squashed.
11. Comparison
    ```
    git log --oneline --graph --all
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/9d40fbf2c69a4678dd1319509810a68731fb6141/2026/day-24/Screenshots/Screenshot%20(11).png)

12. What does squash merging do?<br> 
    **Ans.** It combines all commits from a branch into a single commit before merging into main.
13. When would you use squash merge vs regular merge?<br>
    **Ans.** Use squash for small or messy branches to keep history clean, and regular merge when you want to preserve detailed commit history. 
   
14. What is the trade-off of squashing?<br>
    **Ans.** We gain a tidy, readable history but lose the granular commit details that can help with debugging or tracking contributions.


## Task 4: Git Stash
1. Start on main branch
    ```
    git checkout main
    ```
2. Make changes but do not commit
    ```
    echo "Temporary change" >> login.html
    ```
3. Try switching to another branch
    ```
    git checkout feature-login
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/08c71ab37531885e27629ab59b6bf58787fb7c02/2026/day-24/Screenshots/Screenshot%20(179).png)

    **Git will complain:** "Your local changes would be overwritten by checkout"

4. Save work-in-progress with stash
    ```
    git stash
    ```
    or with a message

    ```
    git stash push -m "WIP: temp changes"
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/08c71ab37531885e27629ab59b6bf58787fb7c02/2026/day-24/Screenshots/Screenshot%20(180).png)

5. Switch to another branch safely
    ```
    git checkout feature-login
    ```
6. Do some commits here...
    ```
    echo "Feature login work" >> login.html
    git add login.html
    git commit -m "Work on login feature"
    ```
7. Switch back to main
    ```
    git checkout main
    ```
8. Apply stashed changes
    ```
    git stash pop
    ```
    **Applies** the stash and removes it from the stash list

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/08c71ab37531885e27629ab59b6bf58787fb7c02/2026/day-24/Screenshots/Screenshot%20(181).png)

9. Stash multiple times
    ```
    echo "Another change" >> login.html
    git stash push -m "Second WIP change"
    ```
    ```
    echo "Third change" >> login.html
    git stash push -m "Third WIP change"
    ```
10. List all stashes
    ```
    git stash list
    ```
    
    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/08c71ab37531885e27629ab59b6bf58787fb7c02/2026/day-24/Screenshots/Screenshot%20(185).png)

11. Apply a specific stash without removing it
    ```
    git stash apply stash@{1}
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/08c71ab37531885e27629ab59b6bf58787fb7c02/2026/day-24/Screenshots/Screenshot%20(187).png)

12. What is the difference between `git stash pop` and `git stash apply`?<br>
    **Ans.** `git stash pop` applies the stashed changes and removes them from the stash list, while `git stash apply` applies the changes but keeps the stash entry for reuse.
13. When would you use stash in a real-world workflow?<br>
    **Ans.** You’d use stash when you’re mid‑work on a feature but need to quickly switch branches (e.g., to fix a production bug) without committing unfinished changes.

## Task 5: Cherry Picking
1. Start from main
    ```
    git checkout main
    ```
2. Create new branch
    ```
    git checkout -b feature-hotfix
    ```
3. Add three commits
    ```
    echo "Hotfix step 1" >> hotfix.txt
    git add hotfix.txt
    git commit -m "Hotfix: step 1"
    ```
    ```
    echo "Hotfix step 2" >> hotfix.txt
    git add hotfix.txt
    git commit -m "Hotfix: step 2"
    ```
    ```
    echo "Hotfix step 3" >> hotfix.txt
    git add hotfix.txt
    git commit -m "Hotfix: step 3"
    ```
4. Switch back to main
    ```
    git checkout main
    ```
5. Find commit hashes in feature-hotfix
    ```
    git log --oneline feature-hotfix
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/08c71ab37531885e27629ab59b6bf58787fb7c02/2026/day-24/Screenshots/Screenshot%20(188).png)

    Copy the commit hash of "Hotfix: step 2"

6. Cherry-pick only the second commit
    ```
    git cherry-pick <commit-hash>
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/08c71ab37531885e27629ab59b6bf58787fb7c02/2026/day-24/Screenshots/Screenshot%20(194).png)

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/08c71ab37531885e27629ab59b6bf58787fb7c02/2026/day-24/Screenshots/Screenshot%20(189).png)

7. Verification
    ```
    git log --oneline --graph --all
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/08c71ab37531885e27629ab59b6bf58787fb7c02/2026/day-24/Screenshots/Screenshot%20(193).png)

8. What does cherry-pick do?  
**Ans.** It takes a single commit from one branch and applies it onto another branch, without merging the entire branch history.

9. When would you use cherry-pick in a real project?  
**Ans.** You’d use it to quickly apply a specific fix or feature commit (like a hotfix) to another branch, such as bringing a bug fix from a feature branch into main without merging everything.

10. What can go wrong with cherry-picking?  
**Ans.** It can introduce duplicate commits or conflicts if the branch is later merged normally, and rewriting history this way can complicate collaboration if not managed carefully.