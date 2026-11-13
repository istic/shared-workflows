# shared-workflows

Reusable GitHub Actions workflows (`workflow_call`) shared across the `istic` repositories. Each one lives in `.github/workflows/` and is called from a consumer repo with `uses:`.

| Workflow | Purpose |
| --- | --- |
| [`laravel-tests.yml`](#laravel-tests) | PHPUnit (and optionally Dusk) for Laravel apps |
| [`python-tests.yml`](#python-tests) | pytest via Poetry on Linux and Windows |
| [`claude-code-review.yml`](#claude-code-review) | Claude Code review of PRs marked ready for review |
| [`auto-rebase-dependabot.yml`](#auto-rebase-dependabot) | Keep a single `dependabot-updates` branch/PR in sync with `main` |
| [`auto-merge-dependabot.yml`](#auto-merge-dependabot) | Merge Dependabot PRs once checks settle |
| [`auto-manage-artifacts.yml`](#auto-manage-artifacts) | Delete workflow artifacts older than 7 days |

`lint-actions.yml` is not reusable: it runs actionlint on this repo's own workflows.

## Using a workflow

The caller chooses the trigger; the shared workflow does the work.

```yaml
# .github/workflows/tests.yml in a consumer repo
name: Tests
on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read

jobs:
  tests:
    uses: istic/shared-workflows/.github/workflows/laravel-tests.yml@main
    with:
      php_versions: '["8.4","8.5"]'
```

Pass secrets explicitly, or use `secrets: inherit`.

## laravel-tests

Runs on `ubuntu-latest` with SQLite. Installs Composer and npm dependencies (Composer's download cache is keyed on `composer.lock`; npm uses `setup-node`'s `cache: npm`), builds assets (`npm run build`), copies `.env.example`, generates an app key, creates the database, then runs the tests.

| Input | Default | Notes |
| --- | --- | --- |
| `php_version` | `8.4` | |
| `php_versions` | `""` | JSON array for a matrix, e.g. `'["8.4","8.5"]'`. Overrides `php_version`. |
| `node_version` | `24` | |
| `has_dusk` | `false` | Adds a second job (`needs` the first) that runs Dusk against a Selenium Chrome service. It uses the GitHub environment `Testing`. |
| `test_command` | `vendor/bin/phpunit` | |
| `setup_command` | `""` | Extra shell run before tests, e.g. `php artisan passport:keys --force`. |
| `environment` | `""` | GitHub environment for scoped variables/secrets. Leave empty to avoid registering a deployment. |

## python-tests

Runs pytest on `ubuntu-latest` and `windows-latest`. Installs Poetry (`poetry install --with dev`) with an in-project virtualenv, cached per OS, Python version and `poetry.lock`.

| Input | Default |
| --- | --- |
| `python_version` | `3.12` |
| `working_directory` | `.` |

## claude-code-review

Reviews a PR with [`anthropics/claude-code-action`](https://github.com/anthropics/claude-code-action), posting inline comments and a short summary. Draft PRs and fork PRs are skipped (forks can't read secrets).

The caller must trigger it on `ready_for_review` (add `opened` to also review PRs opened non-draft) and grant the permissions below, **including `id-token: write` even when `use_claude_app` is off**: reusable workflows can't request permissions conditionally, so a caller that omits it fails at startup.

```yaml
on:
  pull_request:
    types: [ready_for_review]

jobs:
  review:
    uses: istic/shared-workflows/.github/workflows/claude-code-review.yml@main
    secrets: inherit
    permissions:
      contents: read
      pull-requests: write
      issues: read
      id-token: write
    # with:
    #   use_claude_app: true
```

| Input | Default | Notes |
| --- | --- | --- |
| `model` | `""` | Empty uses the action's default model. |
| `use_claude_app` | `false` | Post as the Claude GitHub App (must be installed on the repo/org) instead of `github-actions[bot]`. |

| Secret | Notes |
| --- | --- |
| `ANTHROPIC_API_KEY` | Provide this **or** `CLAUDE_CODE_OAUTH_TOKEN`. The job fails early if neither is set. |
| `CLAUDE_CODE_OAUTH_TOKEN` | Claude subscription token. |

## auto-rebase-dependabot

Maintains one aggregate branch, `dependabot-updates`, by merging `main` into it (creating it from `main` if missing), then opens a draft PR titled "🛡️ dependabot updates" if one doesn't exist and the branch differs from `main`. If the merge conflicts only in lockfiles (`poetry.lock`, `composer.lock`, `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`), they are regenerated.

| Input | Default | Notes |
| --- | --- | --- |
| `php_version` | `8.4` | Used to regenerate `composer.lock`. |
| `node_version` | `""` | Empty uses `.nvmrc`, else `24`. |

| Secret | Notes |
| --- | --- |
| `REBASE_TOKEN` | Optional PAT, needed to push to branches protected by rulesets. Falls back to `github.token`. The token is validated up front. |

Needs `contents: write` and `pull-requests: write`.

## auto-merge-dependabot

Waits up to 15 minutes for the PR's checks to settle, then squash-merges using [`fastify/github-action-merge-dependabot`](https://github.com/fastify/github-action-merge-dependabot). It retries twice with auto-merge enabled, then falls back to a direct merge.

| Secret | Notes |
| --- | --- |
| `MERGE_TOKEN` | Optional PAT. Falls back to `github.token`. A real-identity token is needed for the direct-merge fallback to trigger push-based workflows downstream. The token is validated up front. |

Needs `contents: write` and `pull-requests: write`.

## auto-manage-artifacts

Deletes workflow artifacts in the calling repo that are more than 7 days old. No inputs or secrets; typically called on a `schedule`. It declares no permissions of its own, so the caller must grant `actions: write`.

## Conventions

- Callers should set an explicit `permissions:` block (`contents: read` at minimum). The Dependabot and review workflows declare their own; `laravel-tests`, `python-tests` and `auto-manage-artifacts` don't, so they get whatever the caller grants.
- Don't add an `environment:` key that would register a deployment unless the input is explicitly set.
- Third-party actions that receive credentials are pinned to a commit SHA; Dependabot (`github-actions`, weekly) keeps pins current.
- Workflows are linted with actionlint (in CI and via pre-commit).
