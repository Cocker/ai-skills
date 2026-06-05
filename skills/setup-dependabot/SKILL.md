---
name: setup-dependabot
description: Set up .github/dependabot.yml with minimal grouped dependency updates. Use when adding or revising Dependabot configuration.
---

# Setup Dependabot

Use this skill when creating or updating `.github/dependabot.yml`.

## Defaults

- Use `schedule.interval: "weekly"`, unless specified otherwise.
- Use `open-pull-requests-limit: 5`, unless specified otherwise.
- Use `versioning-strategy: "increase"` where supported, unless specified otherwise.
- Split update groups by `minor`+`patch` versus `major`.
- Split dependency groups by `production` versus `development` when the ecosystem supports `dependency-type`.
- Set Dependabot minimum age with `cooldown`, matching existing package-manager settings such as npm/pnpm/yarn minimum release age. Do not invent a different age.
- Infer `directory` from the repo structure. In monorepos or non-root manifests, inspect lockfiles, workspaces, and package manifests instead of assuming `/`.

## Example Template

```yaml
version: 2

updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    cooldown:
      default-days: 7
    open-pull-requests-limit: 5
    versioning-strategy: "increase"
    groups:
      production-minor-patch:
        dependency-type: "production"
        update-types: ["minor", "patch"]
      production-major:
        dependency-type: "production"
        update-types: ["major"]
      development-minor-patch:
        dependency-type: "development"
        update-types: ["minor", "patch"]
      development-major:
        dependency-type: "development"
        update-types: ["major"]

  - package-ecosystem: "composer"
    directory: "/"
    schedule:
      interval: "weekly"
    cooldown:
      default-days: 7
    open-pull-requests-limit: 5
    versioning-strategy: "increase"
    groups:
      production-minor-patch:
        dependency-type: "production"
        update-types: ["minor", "patch"]
      production-major:
        dependency-type: "production"
        update-types: ["major"]
      development-minor-patch:
        dependency-type: "development"
        update-types: ["minor", "patch"]
      development-major:
        dependency-type: "development"
        update-types: ["major"]

  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 5
    groups:
      github-actions-minor-patch:
        update-types: ["minor", "patch"]
      github-actions-major:
        update-types: ["major"]
```

## Example Requests

- Add Dependabot for this repo.
- Set up minimal Dependabot config.
- Update Dependabot groups for major versus minor/patch dependencies.
