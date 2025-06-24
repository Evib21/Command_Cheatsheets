# Essential Git Commands (80/20 Rule)

| No. | Command                     | Description                                             | Common Use Case                                   |
| --- | --------------------------- | ------------------------------------------------------- | ------------------------------------------------- |
| 1   | `git clone <repo-url>`      | Clone a remote repo to your local machine               | Start working on a project                        |
| 2   | `git status`                | Show changes in your working directory and staging area | Check what’s modified, staged, or untracked       |
| 3   | `git add <file>`            | Stage a specific file for commit                        | Prepare file(s) for commit                        |
| 4   | `git add .`                 | Stage all changes in the current directory              | Quickly stage all changes                         |
| 5   | `git commit -m "msg"`       | Commit staged changes with a message                    | Save a snapshot with a description                |
| 6   | `git push`                  | Push commits to the remote repository                   | Sync local commits to GitHub/GitLab               |
| 7   | `git pull`                  | Fetch and merge changes from the remote repository      | Update local branch with remote changes           |
| 8   | `git fetch`                 | Fetch changes without merging                           | See remote changes before deciding to merge       |
| 9   | `git log`                   | View commit history                                     | Inspect commit timeline                           |
| 10  | `git diff`                  | See changes not yet staged                              | Review unstaged modifications                     |
| 11  | `git branch`                | List all local branches                                 | See all local branches                            |
| 12  | `git branch <branch>`       | Create a new branch                                     | Start a feature or isolate work                   |
| 13  | `git checkout <branch>`     | Switch to another branch                                | Move between branches (older versions of Git)     |
| 14  | `git switch <branch>`       | Modern way to switch branches                           | Preferred for switching in newer Git versions     |
| 15  | `git merge <branch>`        | Merge another branch into the current one               | Combine feature branch into main branch           |
| 16  | `git stash`                 | Temporarily save changes without committing             | Save work before switching branches               |
| 17  | `git stash pop`             | Reapply the last stashed changes                        | Restore previously saved work                     |
| 18  | `git reset --hard <commit>` | Reset to a specific commit, discarding changes          | Undo changes & reset branch to a known good state |
| 19  | `git restore <file>`        | Discard changes in a file (Git 2.23+)                   | Undo uncommitted changes to a file                |
| 20  | `git rm <file>`             | Remove a tracked file                                   | Delete a file from both working directory and Git |
| 21  | `git init`                  | Initialize a new Git repository                         | Start tracking changes in a new project           |
| 22  | `git remote -v`             | View configured remote repositories                     | Check remote URL linked to local repo             |
| 23  | `git log --oneline`         | Show concise commit history                             | Quick overview of recent commits                  |
| 24  | `git reflog`                | Show history of HEAD and branch movements               | Recover lost commits or undo actions              |
| 25  | `git config`                | Configure user name, email, and preferences             | Initial setup or updating Git settings            |

---

