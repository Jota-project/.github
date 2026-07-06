# Jota Project — Standards & Conventions

This document defines the standards, workflows, and CI/CD setup for all repositories in the Jota-project organization.

---

## Branch Protection (GitHub Settings →Branches → Branch protection rules)

Every repo must have these rules on `main`:

### Required settings
- ✅ **Require pull request reviews before merging** — 1 approval minimum
- ✅ **Dismiss stale reviews** when new commits are pushed
- ✅ **Require status checks to pass before merging** — `Tests` must be green
- ✅ **Do not allow bypassing the above rules** (even for admins)

### Why
Without these, anyone (including bots) can push directly to `main`. That breaks semantic-release, pollutes the git history, and makes it impossible to track what changed in each release.

---

## Pull Requests

### Workflow
1. **Create a branch** from `main` — never work on `main` directly
   ```
   git checkout main && git pull origin main
   git checkout -b feat/my-feature
   ```
2. **Commit your changes** following the commit convention below
3. **Open a PR** — the PR title becomes the merge commit message
4. **Wait for CI** — tests must pass
5. **Merge** — use **Squash and merge** (preferred) or **Merge commit**

### PR Title Convention (important!)

The PR title is used as the **merge commit message** and by semantic-release to determine the version bump. Format:

```
<type>(<scope>): <description>
```

**Types:**
| Type | When to use | Version bump |
|---|---|---|
| `feat` | New feature | Minor |
| `fix` | Bug fix | Patch |
| `perf` | Performance improvement | Patch |
| `refactor` | Code restructure (no behavior change) | None |
| `docs` | Documentation only | None |
| `test` | Tests only | None |
| `ci` | CI/CD changes | None |
| `chore` | Maintenance, deps, build changes | None |
| `BREAKING CHANGE` | Breaking API change | Major |

**Examples:**
```
feat(speaker): add GET /v1/voices endpoint
fix(gateway): prevent websocket reconnection loop on downstream timeout
ci: add dependabot for Python dependencies
docs: update architecture diagram
```

### PR Description

Every PR must include:

```markdown
## Summary
What does this PR do?

## Changes
- bullet points of what changed

## Testing
How was this tested? (unit tests, integration tests, manual test)

## Notes
Any known limitations, side effects, or things reviewers should pay attention to.
```

---

## Semantic Release

Configured per-repo via `.github/workflows/release.yml` + `.releaserc.json`.

### How it works
1. You merge a PR to `main` with a conventional commit title
2. GitHub Actions pushes to `main`, triggering the release workflow
3. semantic-release analyses the commit messages since the last release
4. It bumps the version (`patch`, `minor`, or `major`), updates `CHANGELOG.md`, and creates a GitHub Release

### Requirements per repo

**Python repos** (jota-gateway, jota-speaker, jota-db, jota-orchestrator):
- `.releaserc.json` (no `@semantic-release/npm` — use pyproject.toml)
- `CHANGELOG.md` (initially empty)
- `.github/workflows/release.yml`
- `APP_ID` + `APP_PRIVATE_KEY` secrets (GitHub App token)

**C++ repos** (jota-transcriber, jota-inference):
- Same structure but C++ build in CI

### Release workflow template

```yaml
name: Semantic Release

on:
  push:
    branches:
      - main

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      issues: write
      pull-requests: write
    steps:
      - uses: actions/checkout@v7
        with:
          fetch-depth: 0

      - name: Generate Token from Jota-App
        uses: tibdex/github-app-token@v2
        with:
          app_id: ${{ secrets.APP_ID }}
          private_key: ${{ secrets.APP_PRIVATE_KEY }}

      - name: Setup Node.js
        uses: actions/setup-node@v6
        with:
          node-version: 'lts/*'

      - name: Install Semantic Release
        run: |
          npm install -g semantic-release \
            @semantic-release/changelog \
            @semantic-release/git \
            @semantic-release/github

      - name: Run Semantic Release
        env:
          GITHUB_TOKEN: ${{ steps.generate_token.outputs.token }}
        run: npx semantic-release
```

### .releaserc.json template

```json
{
  "branches": ["main"],
  "plugins": [
    "@semantic-release/commit-analyzer",
    "@semantic-release/release-notes-generator",
    "@semantic-release/changelog",
    [
      "@semantic-release/git",
      {
        "assets": ["CHANGELOG.md"],
        "message": "chore(release): ${nextRelease.version} [skip ci]\n\n${nextRelease.notes}"
      }
    ],
    "@semantic-release/github"
  ]
}
```

---

## Dependabot

Configured per-repo via `.github/dependabot.yml`. Updates dependencies automatically via PRs.

### Template

```yaml
version: 2
updates:
  # Python dependencies
  - package-ecosystem: "pip"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
    open-pull-requests-limit: 5
    labels:
      - "dependencies"
      - "python"

  # Docker base images
  - package-ecosystem: "docker"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
    open-pull-requests-limit: 3
    labels:
      - "dependencies"
      - "docker"

  # GitHub Actions versions
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "monthly"
    open-pull-requests-limit: 3
    labels:
      - "dependencies"
      - "github-actions"
```

### C++ repos (add to dependabot.yml)

```yaml
  # C++ dependencies (CMake submodules like whisper.cpp)
  - package-ecosystem: "gitsubmodule"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 3
    labels:
      - "dependencies"
      - "submodule"
```

---

## CI/CD Testing

Every repo must have `.github/workflows/test.yml` that runs on every push and PR.

### Python template

```yaml
name: Tests

on:
  push:
    branches: ["**"]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"
          cache: pip

      - name: Install dependencies
        run: pip install -e ".[dev]"

      - name: Lint (ruff)
        run: ruff check src/ tests/

      - name: Unit tests
        run: pytest tests/unit/ -v

      - name: Integration tests
        run: pytest tests/integration/ -v
```

### Required dev dependencies
```toml
[project.optional-dependencies]
dev = [
    "pytest>=8",
    "pytest-asyncio>=0.23",
    "ruff>=0.4",   # linting
    "httpx>=0.27", # HTTP testing
    "respx>=0.21", # mock HTTP
]
```

---

## Commit Message Convention

Use conventional commits for all commit messages (same format as PR titles):

```
feat(speaker): add per-request voice override
fix(gateway): handle nil context in auth middleware
ci: add dependabot for Python dependencies
```

This is enforced by semantic-release's commit analyser. squash-merged PR titles are also interpreted as commits, so the PR title is the most important place to get the format right.

---

## Repository Structure Checklist

When creating a new repo, set up from day 1:

### Files
- [ ] `README.md` — with quick start, architecture, and API reference
- [ ] `CHANGELOG.md` — initially empty
- [ ] `.releaserc.json` — semantic release config
- [ ] `.github/workflows/test.yml` — lint + unit + integration
- [ ] `.github/workflows/release.yml` — semantic release
- [ ] `.github/dependabot.yml` — automated dependency updates
- [ ] `.gitignore` — appropriate for the language

### GitHub Settings
- [ ] Branch protection on `main` (require PR + status checks)
- [ ] Secrets: `APP_ID`, `APP_PRIVATE_KEY` (for semantic-release)
- [ ] Topics: set meaningful tags (e.g. `tts`, `kokoro`, `websocket`, `python`)

---

## Secrets Reference

| Secret | Used by | Purpose |
|---|---|---|
| `APP_ID` | release workflow | GitHub App ID (numeric) |
| `APP_PRIVATE_KEY` | release workflow | GitHub App private key (PEM) |
| `HA_TOKEN` | Home Assistant scripts | HA long-lived access token |

Store secrets at: **Repo Settings → Secrets and variables → Actions**
