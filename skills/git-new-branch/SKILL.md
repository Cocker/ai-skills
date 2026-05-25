---
name: git-new-branch
description: Create a new Git branch from the right starting point.
---

# Git New Branch Skill

Use this skill when you need to create a new Git branch in this repository.

## Default Base Branch

`origin/main`, unless specified otherwise.

## Capabilities

### Fetch Latest Main Branch

```bash
git fetch --all
```

### Create Feature Branch

```bash
git switch --no-track -c <branch-name> origin/main
```

Example:

```bash
git switch --no-track -c feature/add-map-layer origin/main
```

## Example Requests

- Create a new branch and implement dashboard.
- Move current changes to a new branch.

## Guidelines

- Fetch latest changes before creating a new branch
- Branch from `origin/main`, unless specified otherwise.
