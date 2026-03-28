---
description: Safely update this project's dependencies in isolated commits
name: update-project
---

# Goal

Perform a full dependency update in a safe, reproducible way using the repository execution model.

If any step fails, STOP immediately and report the error.

## Phase 1 --- Repository Safety Checks

1. Verify the current branch is `main`. If not, abort and instruct the user to switch to `main` first.
2. Ensure the working (git) tree is clean:
    - No staged changes
    - No unstaged changes
    - No untracked files
    - If not clean: Abort immediately
        - Inform the user to commit or stash changes
3. Synchronize with remote: `git fetch --prune`
4. Prepare branch `vendor-updates`:
    - If branch does not exist → create from `origin/main`
    - If branch exists:
        - If fully merged into `main` → delete locally and recreate from `origin/main`.
        - If behind on `main` and has NO unique commits → reset `vendor-updates` to `origin/main`.
        - If it contains unique commits → abort and notify user.
5. Check if the branch does not already exist on the remote. If it does, abort and inform the user to delete the
remote branch first.

Always branch from latest `origin/main`.

## Phase 2 --- GitHub Actions Updates

1. Scan the following locations for pinned `uses:` action references:
    - `.github/workflows/*.yml`
    - `.github/actions/**/*.yml`
    - `*/action.yml` (composite actions in subdirectories at the repository root)
2. For each external action (e.g. `actions/checkout@v6`, `docker/login-action@v4`), use the GitHub Releases API to find
the latest release tag. Check every action individually — do not assume that a major-version tag (e.g. `@v6`) is already
current. A newer major version may exist.
   - If the reference uses a floating major-version tag (e.g. `@v6`) and that major version is already the latest,
     leave it as-is — no update is needed.
   - If the reference is pinned to a specific semver tag (e.g. `@v2.5.0`, `@v22.0.0`) and a newer version exists,
     update it to the latest release tag.
   - If the reference is pinned to a commit SHA, leave it as-is — do not convert SHA pins to tags.
3. Skip self-referencing actions (actions that reference the current repository, e.g.
`specsnl/github-actions/build-image@1.0.12`) — they are handled in Phase 4.
4. Update any outdated action versions in-place across all scanned files.
5. Check `git status` for all changed files. Commit separately if there are any changes.
Commit message: `chore(deps): Update GitHub Actions versions`.
6. If there are no changes, skip the commit step and move on to the next phase.

Abort on any failure.

## Phase 3 --- License Year Update

1. Check if the `LICENSE` file contains a year range (e.g. `2020-2025`) or a single year (e.g. `2024`).
2. If the current year (2026) is not already the end year:
   - If the LICENSE file contains a single year (e.g. `2024`) → update it to the current year: `2026`.
   - If the LICENSE file contains a year range (e.g. `2020-2025`) → update the end year: `2020-2026`.
3. If the LICENSE file already ends with the current year (e.g. `2024-2026` or `2026`), no update is needed.
4. Commit separately if there are any changes.
Commit message: `chore: Update copyright year`.
5. If there are no changes, skip the commit step.

## Phase 4 --- Self-referencing Version Bump

1. Determine the current latest release tag of this repository using the GitHub Releases API (e.g., `1.0.12`).
2. Ask the user whether to bump the **patch** version (e.g., `1.0.12` → `1.0.13`) or the **minor** version
(e.g., `1.0.12` → `1.1.0`).
3. Compute the new version tag based on the user's choice.
4. Update all self-referencing action references across all scanned files to use the new version tag.
   - Files to scan: same as Phase 2 (`.github/workflows/*.yml`, `.github/actions/**/*.yml`,
     `*/action.yml`)
   - Self-referencing actions are those that reference the current repository
     (e.g., `specsnl/github-actions/*@<version>`)
   - Update every occurrence across all files before committing — all changes go into a single commit.
5. If any files were changed, stage them all and create one commit.
   Commit message: `chore: Bump self-referencing actions to <new-version>`.
6. If there are no changes, skip the commit step.

Abort on any failure.

------------------------------------------------------------------------

## Final State

Abort if there are no changes after all update steps, and inform the user that everything is already up to date.

- There could be up to 3 commits on the `vendor-updates` branch:
    1. GitHub Actions version updates
    2. Copyright year update
    3. Self-referencing version bump
- Branch: `vendor-updates`
- Based on latest `origin/main`.
- The working tree should be clean.
- Push the branch to the remote and create a pull request for review and merging targeting `origin/main`. The title
should be `chore: Update dependencies`.
- Add a Pull Request description that lists the changes based on the commits that were made. Stick to the
commit messages, but remove the "chore" prefix. Include a post-merge task section with a checkbox to create the
new git tag. For example:

```md
Updated the following dependencies:

- Updated GitHub Actions versions
- Updated copyright year
- Bumped self-referencing actions to <new-version>

## Post-merge tasks

- [ ] Create and push git tag `<new-version>`
```
