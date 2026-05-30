---
name: manage-pnpm-workspace
description: "Manage and upgrade hardened pnpm 11 workspaces and JavaScript/TypeScript monorepos. Use when creating, migrating, auditing, or maintaining pnpm workspaces, including monorepo layout and dependency management."
---

# Manage pnpm Workspace

Use this skill for both initial pnpm setup and ongoing workspace maintenance. Treat a single application repo as a pnpm workspace root too; project-level pnpm settings belong in `pnpm-workspace.yaml`.

## Inspect First

Before editing, collect the workspace shape and package-manager policy:

- Read root `package.json`, `pnpm-workspace.yaml`, `pnpm-lock.yaml`, `.npmrc`, `turbo.json`, `nx.json`, `lerna.json`, and `rush.json`.
- Read package manifests under likely workspace roots: `apps/*`, `packages/*`, `tools/*`, `examples/*`, `services/*`, and any existing `packages` globs.
- Identify whether the repo is a single-app workspace, app-only monorepo, package-publishing monorepo, or mixed stack repo.
- Build the internal package graph from `workspace:*`, `workspace:^`, `workspace:~`, and local path dependencies.
- Do not remove other package-manager files or lockfiles unless the user explicitly asked for a migration.

## pnpm 11 Config

- Put project pnpm settings in `pnpm-workspace.yaml`, not `.npmrc`, except auth and registry tokens.
- Set an exact root `packageManager`, for example `"pnpm@11.2.2"`, using the repo's existing pnpm version when already present.
- If the repo uses an older pnpm major, suggest updating to pnpm 11 before adding new hardening settings.
- Prefer `pmOnFail` over older `managePackageManagerVersions`, `packageManagerStrict`, and `packageManagerStrictVersion` settings.
- Prefer `allowBuilds` over older `onlyBuiltDependencies`, `ignoredBuiltDependencies`, `neverBuiltDependencies`, and `ignoreDepScripts` settings.
- Keep root `private: true` for app repos and unpublished monorepo roots.
- Add or preserve `engines.node` when the runtime floor is known.
- Keep dependency specs exact for hardened app repos. Convert `^`, `~`, and `latest` only when the task is hardening-focused and the lockfile preserves resolved versions.

## Hardened Baseline

Start from this baseline for a single-app workspace:

```yaml
minimumReleaseAge: 10080
minimumReleaseAgeStrict: true
minimumReleaseAgeIgnoreMissingTime: false
trustPolicy: no-downgrade
trustLockfile: false
blockExoticSubdeps: true

engineStrict: true
pmOnFail: error
savePrefix: ""

verifyStoreIntegrity: true
strictStorePkgContentCheck: true
strictDepBuilds: true
dangerouslyAllowAllBuilds: false
allowBuilds: {}
```

For monorepos, add these workspace-specific settings to the baseline:

```yaml
linkWorkspacePackages: false
saveWorkspaceProtocol: rolling
sharedWorkspaceLockfile: true
disallowWorkspaceCycles: true
failIfNoMatch: true

packages:
  - "apps/*"
  - "packages/*"
```

Adjust the baseline deliberately:

- Add `minimumReleaseAgeExclude` only for pinned package/version exceptions that must install immediately and have explicit user confirmation.
- Add `trustPolicyExclude` only for reviewed package/version exceptions that have explicit user confirmation.
- Leave `trustLockfile: false` for public or contributor-edited repos.
- Do not set `dangerouslyAllowAllBuilds: true`.
- Use `allowBuilds` as an explicit allow/deny map. Keep it empty until pnpm reports packages that need review, and ask the user before adding each package.

## Monorepo Layout

- Discover actual package roots from files, not assumptions. Common globs are `apps/*`, `packages/*`, `tools/*`, `examples/*`, and `services/*`.
- Prefer a small number of top-level workspace roots. Do not add broad globs like `**/*`.
- Name internal packages consistently, usually with a repo scope such as `@org/web`, `@org/db`, `@org/config`, and `@org/cli`.
- Use a shared config package for reusable TypeScript, lint, or formatting config when multiple packages need it.
- Mark unpublished internal packages as `private: true`.
- Use `workspace:*` for internal dependencies unless the repo publishes packages and needs semver-specific workspace ranges.
- Keep package-local scripts in each package and expose common entrypoints from the root as script aliases that use `pnpm -F`, for example `"web:build": "pnpm -F @org/web build"`.
- Use dependency-aware filters when the task needs the target package plus its workspace dependencies, for example `"web:build": "pnpm -F @org/web... build"`.
- Avoid cross-package imports through source-relative paths. Depend on the package and use its `exports` map.

## Dependency Policy

- Pin dependency versions when hardening an existing app repo.
- Convert local path dependencies to `workspace:*` when the target package is part of the workspace.
- Keep dependency ownership local: app-only runtime deps belong in the app package, shared runtime deps belong in packages that import them, and repo tooling belongs at the root only when it is actually run from the root.
- Review `minimumReleaseAgeExclude`, `trustPolicyExclude`, and `allowBuilds` during dependency updates; remove exceptions that are no longer needed, and require explicit user confirmation before adding new exceptions or build-script approvals.

## Common Tasks

### Add a Workspace Package

- Create the package under an existing workspace root when possible.
- Add `name`, `version`, and `private: true` for unpublished packages. Prefer `type: module` for greenfield or ESM workspaces; preserve CommonJS in existing CJS repos unless the task is an ESM migration.
- Add an `exports` map for packages consumed by other workspace packages.
- Add package-local scripts such as `build`, `typecheck`, `test`, or `lint` only when they are meaningful.
- Add root script aliases only for common workflows the user will run directly, using `pnpm -F <package> <script>`.
- Validate with `pnpm -r list --depth -1` and a filtered command for the new package.

### Harden an Existing Repo

- Add or merge `pnpm-workspace.yaml` with the hardened baseline.
- Set exact `packageManager` in root `package.json`.
- Pin direct dependency specs using lockfile resolved versions when available.
- Run lockfile-only install first to surface config and build-script issues without changing source behavior.
- Review every requested build-script approval and get explicit user confirmation before adding it to `allowBuilds`.

### Migrate to pnpm

- Treat `package-lock.json`, `yarn.lock`, `bun.lock`, or `bun.lockb` as migration evidence.
- Ask before deleting old lockfiles.
- Preserve scripts and package names.
- Convert internal links to `workspace:*` after confirming the target package is in `packages` globs.
- Run `pnpm install --lockfile-only`, then the smallest repo-specific checks.

## Migration Checks

- If `.npmrc` contains non-auth pnpm settings, move them to `pnpm-workspace.yaml`.
- Migrate older pnpm hardening keys deliberately.
- Migrate `packageManagerStrictVersion: true` to `pmOnFail: error`.
- Migrate `managePackageManagerVersions: false` to `pmOnFail: ignore`, unless strict version matching is desired.
- Migrate `onlyBuiltDependencies`, `ignoredBuiltDependencies`, `neverBuiltDependencies`, and `ignoreDepScripts` to `allowBuilds`.
- If direct dependency specs are ranged but the lockfile has exact resolved versions, pin direct dependencies to those exact versions before regenerating the lockfile.

## Verification

When changing workspace or dependency config, run the smallest command set that proves the workspace still works:

```bash
pnpm install --lockfile-only
pnpm -r list --depth -1
pnpm -r exec node --version
```

Then run repo-specific checks. For monorepos, prefer filtered commands that match changed packages before running everything:

```bash
pnpm -F @org/web build
pnpm -F @org/web test
pnpm -F @org/db typecheck
```

For pure audits, avoid mutating the lockfile. Use `pnpm install --frozen-lockfile` when validating committed lockfile consistency.
