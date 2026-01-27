---
longform:
  format: single
  title: Git
title: Git
---
# Comprehensive Version Control Guide: Git & GitHub

## 1. Introduction to Version Control

**Version Control** is a system that tracks changes to files over time, enabling collaboration, history tracking, and code management. It allows developers to work simultaneously, merge changes, and revert to previous versions when needed.

## 2. Centralized vs Distributed VCS

**Centralized VCS (CVS, SVN)**:
- Single central server stores all versions
- Requires network connection for most operations
- Single point of failure
- Examples: Subversion, Perforce

**Distributed VCS (Git, Mercurial)**:
- Every developer has complete repository copy
- Can work offline
- No single point of failure
- Examples: Git, Mercurial

## 3. Getting Started with Git

**Installation**:
- Download from git-scm.com
- Available for Windows, macOS, Linux

**Configuration**:
Set global configuration with:
- git config --global user.name "Your Name"
- git config --global user.email "email@example.com"

**GitHub Account Setup**:
1. Create account at github.com
2. Set up SSH keys for authentication
3. Configure profile and preferences

## 4. Core Git Commands

**Repository Management**:
- git init - Initialize new repository
- git clone [url] - Clone existing repository
- git status - Check repository status

**Basic Workflow**:
1. Make changes to files
2. git add [files] - Stage changes
3. git commit -m "message" - Commit changes
4. git push - Send to remote repository
5. git pull - Get updates from remote

**History & Inspection**:
- git log - View commit history
- git diff - See changes
- git show [commit] - View specific commit

## 5. Branching & Merging

**Branch Operations**:
- git branch - List/create branches
- git checkout [branch] - Switch branches
- git checkout -b [branch] - Create and switch
- git merge [branch] - Merge branches
- git branch -d [branch] - Delete branch

**Remote Branches**:
- git fetch - Download remote changes
- git pull --rebase - Update with rebase
- git push origin [branch] - Push branch

#### Git Merge vs Git Rebase

##### **Git Merge**

**Purpose**: Combines changes from different branches while preserving the commit history.

**How it works**:

- Creates a new "merge commit" that ties together the histories
- Preserves the exact timeline of when commits were made
- Non-destructive operation

**Common workflow**:
```bash
# Merge feature branch into main
git checkout main
git merge feature-branch
```

**Visual result**:
```bash
      A---B---C feature
     /         \
D---E---F---G---H main (with merge commit)
```

**When to use**:

- When you want to preserve the complete history
- When working on public/shared branches
- When multiple people are collaborating on the same branch

**Pros**:

- Simple and safe
- Preserves original context
- Non-destructive

**Cons**:

- Can create a cluttered history with many merge commits
- Harder to read linear history

##### **Git Rebase**

**Purpose**: Re-applies commits from one branch onto another, creating a linear history.

**How it works**:

- Takes commits from your branch and "replays" them on top of another branch
- Creates new commits (with new hashes)
- Results in a linear, cleaner history

**Common workflow**:
```bash
# Rebase feature branch onto main
git checkout feature-branch
git rebase main

# Then fast-forward merge
git checkout main
git merge feature-branch
```

**Visual result** (before rebase):
```
      A---B---C feature
     /
D---E---F---G main
```

**After rebase**:
```
              A'--B'--C' feature
             /
D---E---F---G main
```

**When to use**:

- When you want a clean, linear project history
- Before merging to clean up your branch commits
- For local branches (not yet shared)

**Pros**:

- Clean, linear history
- Easier to navigate and understand
- No unnecessary merge commits

**Cons**:

- Rewrites history (can cause issues if already pushed)
- More complex conflict resolution
- Can be dangerous on shared branches

##### **Git Cherry-Pick**

Cherry-pick applies **specific commits** from one branch to another without merging entire branches.

###### Quick Setup & Example

```bash
# Setup
mkdir demo && cd demo
git init
echo "# Project" > README.md
git add . && git commit -m "Initial commit"

# Create feature branch with two commits
git checkout -b feature
echo "func1()" > feature1.js
git add . && git commit -m "Add feature1"
echo "func2()" > feature2.js
git add . && git commit -m "Add feature2"

# Get commit hashes
git log --oneline
# Output: 
# abc1234 Add feature2
# def5678 Add feature1
# xyz9876 Initial commit

# Cherry-pick only feature1 to main
git checkout main
git cherry-pick def5678
```

###### Visual Diagram

###### Before Cherry-Pick:
```
      main:     O─────O─────O
                ↑
              Initial
                
      feature:  O─────A─────B
                        ↑    ↑
                    feature1 feature2
                    (def5678)(abc1234)
```

###### After Cherry-Pick:
```
      main:     O─────O─────O─────A'
                ↑                 ↑
              Initial         Cherry-picked
                                feature1
                                (same changes as A)
                
      feature:  O─────A─────B
                (unchanged)
```

###### Key Points
- **A'** = New commit with same changes as **A**, but different hash
- Only selected commit(s) are copied
- Original branch remains unchanged
- Useful for selective bug fixes/features across branches

## 6. Advanced Git Operations

**Stashing**:
- git stash - Save uncommitted changes
- git stash pop - Restore stashed changes
- git stash list - View stashes

**Rebasing**:
- git rebase [branch] - Reapply commits
- git rebase -i - Interactive rebase

**Undoing Changes**:
- git revert [commit] - Create undo commit
- git reset --soft/hard - Move HEAD pointer
- git checkout -- [file] - Discard changes

**Tagging**:
- git tag [name] - Create lightweight tag
- git tag -a [name] -m "msg" - Annotated tag
- git push --tags - Share tags

## 7. GitHub Workflows

**Forking Workflow**:
1. Fork repository on GitHub
2. Clone your fork locally
3. Create feature branch
4. Make changes and commit
5. Push to your fork
6. Create Pull Request

**Pull Request Process**:
- Create PR from fork to original
- Add description and reviewers
- Address review comments
- Merge via squash/merge/rebase

## 8. Git Ignore Patterns

Create `.gitignore` file to exclude:
- Node modules: node_modules/
- Build outputs: dist/, build/
- Environment files: .env
- IDE files: .vscode/, .idea/
- OS files: .DS_Store, Thumbs.db