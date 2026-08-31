# Git & GitHub Cheat Sheet

A practical reference for the Git commands and GitHub workflow used in this repository.

> **Mental model:** Git is the version-control system. GitHub is a platform that hosts Git repositories and provides collaboration features such as pull requests, reviews, issues, and Actions.

## 1. Basic Git Workflow

```text
Working Directory
      |
      | git add
      v
Staging Area
      |
      | git commit
      v
Local Repository
      |
      | git push
      v
Remote Repository (GitHub)
```

Typical workflow:

```bash
git status
git add <file>
git commit -m "Describe the change"
git push
```

## 2. Repository Setup

```bash
git init
git clone <repository-url>
```

`git init` creates a new Git repository. `git clone` creates a local copy of an existing repository.

## 3. Check Status

```bash
git status
```

Shows the current branch, modified files, staged files, untracked files, and whether the local branch is ahead/behind its remote.

## 4. Stage Changes

```bash
git add <file>
git add file1.md file2.md
git add .
```

`git add` moves changes into the staging area. It does not create a commit.

## 5. Commit Changes

```bash
git commit -m "Describe the change"
```

A commit creates a snapshot of staged changes in the local Git repository.

Good examples:

```text
Add Docker development environment
Add Ansible inventory
Implement disk usage collection
Document project architecture
Fix SSH connectivity
```

## 6. View Changes

```bash
git diff
git diff --staged
git show <commit>
```

- `git diff` — unstaged changes
- `git diff --staged` — staged changes
- `git show` — details of a commit

## 7. View History

```bash
git log
git log --oneline
git log --oneline --all --graph
```

The last command is particularly useful for understanding branch history.

## 8. Branches

A branch is a movable pointer to a line of development.

```text
main
 |
 A---B
             C---D  develop
```

Commands:

```bash
git branch
git branch <branch-name>
git switch <branch-name>
git switch -c <branch-name>
git branch -d <branch-name>
```

Example:

```bash
git switch -c feature/docker-environment
```

## 9. Local vs Remote Branches

Local branches exist in your local Git repository.

Remote-tracking branches represent the state Git knows about on a remote.

```text
Local:
develop

Remote-tracking:
origin/develop

GitHub:
develop
```

Commands:

```bash
git branch
git branch -r
git branch -a
```

## 10. Remotes

A remote is another repository location known to Git.

The conventional name for the primary remote is `origin`.

```bash
git remote -v
git remote add origin <repository-url>
```

Example:

```text
origin  git@github.com:username/project.git (fetch)
origin  git@github.com:username/project.git (push)
```

## 11. Push

`git push` sends local commits to a remote repository.

```bash
git push
```

For the first push of a new branch:

```bash
git push -u origin <branch-name>
```

Example:

```bash
git push -u origin feature/docker-environment
```

The `-u` sets the upstream relationship, so later `git push` is normally sufficient.

Explicit push:

```bash
git push origin main
```

## 12. Fetch vs Pull

### Fetch

```bash
git fetch
```

Downloads information and commits from the remote without integrating them into the current branch.

### Pull

```bash
git pull
```

Normally fetches remote changes and integrates them into the current branch.

A useful simplified model:

```text
git pull
    =
git fetch
    +
integrate changes
```

Do not confuse `git pull` with a GitHub Pull Request.

## 13. Merge

A merge combines the histories of two branches.

Example:

```text
main

A---B
           C---D  develop
```

Merge `develop` into `main`:

```bash
git switch main
git merge develop
```

Depending on the history, Git may perform a fast-forward merge or create a merge commit.

## 14. Pull Requests

A **Pull Request (PR)** is a GitHub collaboration mechanism for proposing that changes from one branch be merged into another branch.

Typical workflow:

```text
feature branch
      |
    commit
      |
     push
      |
GitHub branch
      |
Pull Request
      |
review / checks
      |
    merge
      |
develop or main
```

Important:

> A Pull Request is a **GitHub concept**, not a Git command.

Do not confuse:

```text
git pull
```

with:

```text
Pull Request
```

They are completely different.

## 15. GitHub Flow

A simplified GitHub Flow workflow:

```text
main
 |
 | create branch
 v
feature branch
 |
 | changes
 | commit
 | push
 v
GitHub feature branch
 |
 | Pull Request
 v
Review / automated checks
 |
 | merge
 v
main
```

For this project we are using a slightly more structured variation:

```text
main
  ^
  |
develop
  ^
  |
feature/*
```

Feature branches can be merged into `develop`.

After a meaningful, tested milestone, `develop` can be merged into `main`.

## 16. Our Project Workflow

For `ansible-server-usage-monitor`:

```text
main
  |
  | stable milestone
  ^
develop
  ^
feature/docker-environment
feature/ansible-inventory
feature/data-monitoring
feature/reporting
```

Example:

```bash
git switch develop
git switch -c feature/docker-environment
```

Work on the feature:

```bash
git status
git add .
git commit -m "Add Docker server environment"
git push -u origin feature/docker-environment
```

Then create a Pull Request on GitHub.

After review/testing:

```text
feature/docker-environment
          ↓
       develop
```

Once a larger milestone is complete and tested:

```text
develop
   ↓
Pull Request
   ↓
main
```

## 17. Undo / Restore Basics

Unstage a file:

```bash
git restore --staged <file>
```

Discard unstaged changes to a file:

```bash
git restore <file>
```

> Be careful: `git restore <file>` discards uncommitted changes to that file.

Always inspect first:

```bash
git status
git diff
```

## 18. Rename and Delete

Rename:

```bash
git mv old-name.md new-name.md
```

Delete:

```bash
git rm <file>
```

Then commit:

```bash
git commit -m "Rename documentation file"
```

## 19. Useful Inspection Commands

```bash
git branch --show-current
git branch -a
git remote -v
git ls-remote --heads origin
git log --oneline --all --graph
git status
```

## 20. Common Daily Workflow

```bash
git status
```

Make changes, then inspect:

```bash
git diff
```

Stage:

```bash
git add <file>
```

Check staged changes:

```bash
git diff --staged
```

Commit:

```bash
git commit -m "Describe the change"
```

Push:

```bash
git push
```

## 21. Git vs GitHub

### Git

Git is the distributed version-control system.

Git handles:

- Commits
- Branches
- Merging
- History
- Staging
- Local repositories
- Remotes

Common commands:

```bash
git add
git commit
git branch
git switch
git merge
git log
git diff
git fetch
git pull
git push
```

### GitHub

GitHub is a platform built around Git.

GitHub provides:

- Remote repositories
- Pull Requests
- Code review
- Issues
- Projects
- GitHub Actions
- Repository permissions
- Branch protection
- Collaboration

Mental model:

```text
Git
 |
 +-- Version control
 +-- Local repository
 +-- Branches
 +-- Commits
 +-- Merge
 +-- Remote communication
 |
 v
GitHub
 |
 +-- Hosts Git repositories
 +-- Pull Requests
 +-- Reviews
 +-- Issues
 +-- Actions
 +-- Collaboration
```

## 22. Important Terminology

| Term | Meaning |
|---|---|
| Repository | Project directory tracked by Git, including its Git history |
| Working tree | Files currently checked out and being worked on |
| Staging area | Changes selected for the next commit |
| Commit | Snapshot of staged changes |
| Branch | Movable pointer to a line of development |
| Remote | Another repository location known to Git |
| `origin` | Conventional name for the primary remote |
| Upstream branch | Remote-tracking branch associated with a local branch |
| Fetch | Download remote information without integrating it into the current branch |
| Pull | Fetch remote changes and integrate them into the current branch |
| Push | Send local commits to a remote repository |
| Merge | Combine the histories of two branches |
| Pull Request | GitHub mechanism for proposing and reviewing a branch merge |

## 23. Commands to Memorize First

```bash
git status
git add .
git commit -m "message"
git log --oneline
git diff
git branch
git switch <branch>
git switch -c <new-branch>
git remote -v
git fetch
git pull
git push
git push -u origin <branch>
git merge <branch>
```

## 24. Quick Mental Model

```text
                 GIT
                  |
        +---------+---------+
        |                   |
      LOCAL               REMOTE
        |                   |
    working tree        GitHub repo
        |
     staging
        |
     commit
        |
     branch
        |
      push --------------->
        |
      <---------------- fetch
        |
      pull
```

Collaborative development:

```text
feature branch
      |
   commits
      |
     push
      |
GitHub branch
      |
Pull Request
      |
review / checks
      |
    merge
      |
   develop
      |
stable milestone
      |
    main
```

## Official References

- Git reference: https://git-scm.com/docs
- Git official cheat sheet: https://git-scm.com/cheat-sheet
- Pro Git book: https://git-scm.com/book/en/v2
- GitHub Git cheat sheet: https://docs.github.com/en/get-started/git-basics/git-cheatsheet
- GitHub Flow: https://docs.github.com/en/get-started/using-github/github-flow
- GitHub Git documentation: https://docs.github.com/en/get-started/using-git

