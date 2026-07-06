# Contributing to Jota

Thank you for contributing! Please read the full standards before submitting PRs:

👉 **[STANDARDS.md](./STANDARDS.md)**

It covers:
- Branch protection and PR requirements
- Commit/PR title convention (required for semantic-release)
- How to configure semantic-release, Dependabot, and CI for a new repo
- CI/CD testing requirements

## Quick rules

1. **Never push to `main`** — always use a feature branch + PR
2. **PR titles must follow conventional commits** — they become the merge commit message and drive semantic-release
3. **CI must be green before merging** — tests + lint must pass
4. **Document new features** — update the relevant README

## Quick start

```bash
# Create a feature branch
git checkout main && git pull origin main
git checkout -b feat/my-feature

# Run tests
pip install -e ".[dev]"
ruff check src/ tests/
pytest tests/

# Commit (PR title is the commit message)
git commit -m "feat(repo): description of the change"

# Push and open PR
git push -u origin feat/my-feature
```

## Reporting bugs

Use the [bug report template](./ISSUE_TEMPLATE/bug_report.md).
