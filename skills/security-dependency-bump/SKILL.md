---
name: security-dependency-bump
description: Investigate and patch vulnerable Node.js packages in npm, yarn, or pnpm repos. Use when the user asks to bump a package for security reasons, review a vulnerable dependency, or determine whether a package is direct or transitive.
---

# Security Dependency Bump

Handle security-driven package remediation for Node repositories. First explain the dependency and the safest fix, then wait for explicit approval before editing files.

## Scope

- Node-only. Support npm, yarn, and pnpm.
- Prefer the smallest safe change that removes the vulnerable path.
- For analysis, use package.json plus the owning lockfile rather than node_modules.
- Do not treat an existing overrides or resolutions pattern in package.json as a reason to skip considering a direct dependency bump or direct-parent upgrade first.
- Do not guess a safe version when none is known.
- Do not default to blanket npm audit fix, yarn audit --fix, or similar bulk commands.

## Inputs To Gather

Ask for these before patching:

- vulnerable package name
- advisory, CVE, or audit output if the user has it
- vulnerable version or affected range if known
- target safe version or range if already known
- affected workspace or path if the repo is a monorepo

If the safe version is not known, inspect the advisory or existing tool output. If you still cannot identify a safe target confidently, stop and ask instead of guessing.

## Detect Repository Context

1. Find the package manager from the lockfile that owns the change.
   - pnpm-lock.yaml => pnpm
   - yarn.lock => yarn
   - package-lock.json or npm-shrinkwrap.json => npm
2. Find the workspace root that owns that lockfile.
   - In monorepos, prefer the root package.json that declares workspaces or otherwise owns the relevant lockfile.
   - If the user names a workspace, start there and walk upward to the owning root.
3. Identify the manifest that declares the dependency.
   - Use the affected workspace package.json for direct dependencies.
   - Use the workspace root package.json when adding overrides or resolutions.
4. If multiple roots or lockfiles are plausible, present the ambiguity and ask before patching.

## Classify The Dependency

Treat the package as direct only if it is declared in the relevant package.json under one of:

- dependencies
- devDependencies
- optionalDependencies
- peerDependencies

Otherwise treat it as indirect.

If the package is a peer dependency, treat it as direct but note that the consuming package may still resolve the vulnerable version independently.

Use package-manager-native graph inspection to explain why it is present:

- npm explain <package> and, when needed, npm ls <package>
- yarn why <package>
- pnpm why <package>

Capture at least one concrete dependency chain. When the tool output makes it visible, note whether the vulnerable path is dev-only. Do not let installed-tree output override package.json plus the lockfile during analysis.

## Check Whether The Vulnerability Applies

Before recommending a patch, confirm that the current repo is actually affected.

- Determine the currently resolved version or versions from the lockfile.
- Compare those versions against the advisory's affected range when that range is known.
- If the package is no longer present, or the resolved versions are already outside the vulnerable range, treat the advisory as no longer applicable to the current repo.

When the vulnerability no longer applies:

- do not recommend a patch just because the package appears in an old report
- trace why the package disappeared or moved to a safe version by inspecting git history for the relevant manifest and lockfile, and by checking whether a direct parent now resolves to a release that dropped or replaced the vulnerable subdependency
- explain why it no longer applies now, for example because the package was removed entirely, upgraded past the affected range, or removed from the graph because a direct dependency no longer pulls it in

If git history is unavailable, shallow, or inconclusive, still report that the current repo does not appear affected and explain any current graph evidence that a direct parent no longer includes the vulnerable subdependency.

## Choose The Remediation Strategy

Always explain both the classification and the recommended fix.

### Direct dependency

- Bump the dependency in the relevant manifest to the smallest safe version or range that satisfies the advisory.
- Refresh the lockfile with the matching package manager.
- Prefer a targeted install/update command over a general upgrade sweep.

### Indirect dependency

Default order:

1. Identify the direct parent that introduces the vulnerable package.
2. Check whether upgrading that parent to a safe compatible version removes the vulnerable chain.
3. If that does not work, add a transitive pin at the workspace root that owns the lockfile:
   - overrides for npm and pnpm
   - resolutions for Yarn

Choose the smallest safe change that removes all identified vulnerable paths. Prefer upgrading the direct parent over adding overrides or resolutions when both fix the issue cleanly, but weigh the override/resolution alternative explicitly if the safe parent is a major version bump. Existing overrides or resolutions entries elsewhere in the repo do not change that priority.

## Approval Gate

Before any edit, present a concise approval summary covering:

- whether the vulnerability still applies to the current repo
- whether the dependency is direct or indirect
- why the package is present, including at least one dependency chain
- affected workspace or manifest
- recommended fix
- why that fix was chosen over the main alternative, including whether a direct dependency bump or direct-parent upgrade was evaluated first

Do not patch until the user explicitly approves.

If the vulnerability no longer applies, present the explanation and historical trace, then stop unless the user explicitly asks for follow-up work.

If the user only wants analysis, stop after the explanation and recommendation.

## Patching After Approval

Use the detected package manager and make the minimum necessary manifest and lockfile changes.

Skip patching entirely when the current repo is already outside the affected range or the vulnerable package is no longer present.

- For direct dependencies, update the relevant workspace manifest and refresh the lockfile.
- For indirect dependencies fixed through a parent upgrade, update the direct parent in the relevant manifest and refresh the lockfile.
- For indirect dependencies fixed through a transitive pin, add overrides or resolutions at the workspace root that owns the lockfile, then refresh the lockfile.

Do not move the fix to another workspace just because the package appears there transitively. Patch the manifest or root that actually controls resolution.

## Validation

After patching, auto-discover validation commands from package.json scripts.

Validation order:

1. Inspect root scripts first.
2. Run available lint scripts.
3. Run available typecheck or compile scripts such as typecheck, tsc, build, or another clearly equivalent compile step.
4. Run available test scripts.
5. If root scripts are missing but the affected workspace has them, run the workspace-level equivalents.

Do not invent commands that are not defined by the repo. If no relevant scripts exist, say so clearly.

If validation fails:

- keep the patch in place
- report the failing command and the relevant error
- do not auto-rollback unless the user asks

## Output Shape

When reporting the analysis before approval, present a compact Markdown table with these columns:

| Package | Applies | Path | Plan |
| --- | --- | --- | --- |

Table rules:

- Package should include the resolved version when known, for example nth-check@1.0.2.
- Applies should be Yes when the current repo is affected.
- Applies should be No¹, No², and so on when the vulnerability no longer applies and an explanation is needed below the table. Use Unicode superscript digits such as ¹, ², ³, and ¹⁰.
- Path should use the shortest useful dependency chain, for example @svgr/plugin-svgo -> svgo -> css-select -> nth-check.
- For direct dependencies, Path can be axios (direct).
- If multiple affected paths exist, show the shortest chain first; if lengths are equal, show the one whose direct parent is easiest to bump. Suffix with (+N more) when needed.
- Plan should stay short: Bump axios, Bump svgo, Add root override, or No patch.

After the table, add a short Evidence list for every Noⁿ row. Each evidence item must match the row marker and explain why no patch is needed now.

Each Evidence item should include:

- repo commit and PR links when they can be identified
- otherwise, a plain explanation of the current graph evidence
- the relevant parent-dependency change when that is the reason, for example that eslint@9.x no longer pulls glob-parent
- or the resolved-version explanation when the package is still present but already outside the advisory's affected range

Example pre-approval output:

| Package | Applies | Path | Plan |
| --- | --- | --- | --- |
| axios@0.27.2 | Yes | axios (direct) | Bump axios to ^1.8.2 |
| nth-check@1.0.2 | Yes | @svgr/plugin-svgo -> svgo -> css-select -> nth-check | Bump @svgr/plugin-svgo |
| glob-parent | No¹ | eslint -> chokidar -> glob-parent | No patch |

Evidence

- ¹ Current graph no longer resolves glob-parent. The package disappeared because eslint@9.x no longer pulls that subdependency. Include the relevant repo commit and PR links when they are identifiable; otherwise state this parent-dependency explanation directly.

After approval, report:

- files changed
- package manager command used
- validation commands run
- final status, including any remaining failures or follow-up work

## Example Requests

- Bump axios for a security advisory
- Figure out why nth-check is in this repo and patch it safely
- Is minimist direct or transitive here?
- Patch this vulnerable package with overrides if needed
- This audit says lodash is vulnerable, but I think we already removed it
- Patch the dependency from this security advisory: https://example.com/link/to/advisory
- Patch braces in the packages/frontend workspace only