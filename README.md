# poc-legacy-relay-repo

Reusable GitHub Actions for syncing a **legacy repository** to a **new repository**.
Every merge to the legacy repo's default branch triggers a Pull Request in the new repo
with the synced changes — ready for human review before merging.

---

## Features

- 🔄 **Automatic sync** — every merge to the legacy repo creates or updates a PR in the new repo
- 🚫 **Exclude paths** — configure files/directories in legacy that should **not** be copied to new
- 🛡️ **Preserve new-only files** — files that exist only in the new repo are never deleted
- 🔀 **One open PR at a time** — pushes to the existing sync branch instead of creating duplicates
- 🏷️ **Configurable** — customize branch names, PR title, labels, commit message and more

---

## Quick start — legacy repository

Copy [`examples/legacy-repo-sync.yml`](examples/legacy-repo-sync.yml) to
`.github/workflows/sync-to-new.yml` in your **legacy repository** and adjust the values:

```yaml
name: Sync to New Repo

on:
  push:
    branches:
      - main

jobs:
  sync:
    uses: robertoaokistone/poc-legacy-relay-repo/.github/workflows/sync-legacy-to-new.yml@main
    with:
      new_repo: 'your-org/new-repo'
      exclude_paths: |
        .legacy-only/
        legacy-config.txt
        scripts/deploy-legacy.sh
    secrets:
      new_repo_token: ${{ secrets.NEW_REPO_TOKEN }}
```

### Required secret

Create a secret called **`NEW_REPO_TOKEN`** in the legacy repository. Two options:

**GitHub App (recommended for cross-org)**
Create a GitHub App installed in both organizations with the permissions below,
generate an installation token, and store it as `NEW_REPO_TOKEN`.
This avoids relying on a personal account and works cleanly across org boundaries.

**Personal Access Token (simpler for same-org)**
Use a fine-grained PAT with the permissions below on the **new** repository.

| Permission | Scope |
|---|---|
| `contents` | **Write** |
| `pull-requests` | **Write** |

---

## How it works

```
Legacy repo (merge to main)
        │
        ▼
  [sync workflow]
        │
        ├─ checkout legacy repo at merge commit
        ├─ checkout new repo
        ├─ rsync legacy → new
        │     ├─ skip files in exclude_paths
        │     └─ keep files that exist only in new  (no --delete)
        ├─ commit to sync/from-legacy branch
        └─ open (or update) a PR in the new repo
                │
                ▼
          Human review → merge
```

1. On every `push` to the configured branch the reusable workflow runs in your legacy repo.
2. The workflow checks out the legacy repo at the triggering commit and the new repo.
3. A sync branch (`sync/from-legacy` by default) is created or reset to the new repo's base branch.
4. `rsync` copies files from legacy → new with these rules:
   - Files listed in `exclude_paths` are **not** copied.
   - Files that exist **only in the new repo** are preserved (rsync runs without `--delete`).
   - Files modified in **both** repos will show the legacy version in the PR diff — the reviewer decides.
5. If there are changes, a commit is pushed and a Pull Request is created (or the existing one updated).
6. A human reviews the PR and merges when ready.

> **Conflict / hotfix policy**
> If a file was hotfixed directly in the new repo and the same file was later modified in legacy,
> the sync PR will contain the legacy version. The PR reviewer should inspect the diff and
> cherry-pick the hotfix into the PR branch as needed.
> The recommended direction is **not to auto-merge** sync PRs — always review.

---

## Reusable workflow inputs

Used when calling `.github/workflows/sync-legacy-to-new.yml` via `workflow_call`:

| Input | Required | Default | Description |
|---|---|---|---|
| `new_repo` | ✅ | — | Target new repository (`owner/repo`) |
| `base_branch` | ❌ | `main` | Base branch in the new repo |
| `sync_branch` | ❌ | `sync/from-legacy` | Branch for the sync PR |
| `exclude_paths` | ❌ | `''` | Newline-separated paths to exclude from sync |
| `commit_message` | ❌ | `chore: sync changes from legacy repo` | Commit message |
| `pr_title` | ❌ | `chore: sync changes from legacy repo` | PR title |
| `pr_body` | ❌ | *(default warning message)* | PR body |
| `pr_labels` | ❌ | `''` | Comma-separated labels to add to the PR |

**Secret:** `new_repo_token` *(required)* — token with `contents:write` + `pull-requests:write` on the new repo.

---

## Composite action inputs

For advanced scenarios you can call the composite action directly in your own workflow:

```yaml
steps:
  - uses: robertoaokistone/poc-legacy-relay-repo/.github/actions/sync-legacy-to-new@main
    with:
      new_repo: 'your-org/new-repo'
      new_repo_token: ${{ secrets.NEW_REPO_TOKEN }}
      exclude_paths: |
        .legacy-only/
        docs/legacy-guide.md
```

All reusable workflow inputs are available plus:

| Input | Required | Default | Description |
|---|---|---|---|
| `legacy_repo` | ❌ | `${{ github.repository }}` | Source legacy repository |
| `legacy_ref` | ❌ | `${{ github.sha }}` | Git ref to sync from |
| `legacy_token` | ❌ | `${{ github.token }}` | Token to read the legacy repo |

**Outputs:** `pr_url` (URL of the PR or empty), `has_changes` (`true`/`false`).

---

## Version pinning

The examples in this document reference `@main`, which always uses the latest version.
For **production** use, pin to a specific tag or commit SHA to avoid unexpected breaking
changes:

```yaml
# Reusable workflow — pin to a release tag
uses: robertoaokistone/poc-legacy-relay-repo/.github/workflows/sync-legacy-to-new.yml@v1.0.0

# Composite action — pin to a release tag
uses: robertoaokistone/poc-legacy-relay-repo/.github/actions/sync-legacy-to-new@v1.0.0
```

---

## Archiving the legacy repo

When the legacy repo is ready to be archived:

1. Merge or close any open sync PRs in the new repo.
2. Remove the sync workflow from the legacy repo (or simply archive the repo on GitHub).
3. The new repo continues independently.

---

## Examples

See the [`examples/`](examples/) directory for ready-to-use workflow files.
