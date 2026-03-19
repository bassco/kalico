# Kalico Plugin: PyPI & GitHub Actions Setup Guide

This document walks through setting up automated releases for a Kalico plugin using:
- GitHub Actions for CI/CD
- PyPI with Trusted Publishing (no API tokens needed)
- GitHub App for secure, long-lived authentication

## Overview

```
push to main
    └── release.yml  (python-semantic-release)
            ├── bumps version, commits, creates git tag
            └── creates GitHub Release  ──→  release:published event
                                                    └── publish.yml
                                                            ├── builds dist/
                                                            └── pypa/gh-action-pypi-publish
                                                                    └── PyPI (OIDC / Trusted Publisher)
```

## Prerequisites

- GitHub account with owner/admin access to the repo
- PyPI account with 2FA enabled
- Python 3.9+ locally for testing

---

## Step 1: Register PyPI Project with Trusted Publisher

### 1.1 Create PyPI Account (if needed)

1. Go to <https://pypi.org/account/register/>
2. Enable 2FA on your account

### 1.2 Register Trusted Publisher

1. Go to <https://pypi.org/manage/account/publishing/>
2. Create a **Pending Trusted Publisher** with:

| Field | Value |
|-------|-------|
| PyPI project name | `<your-plugin-name>` (e.g., `kalico-my-plugin`) |
| GitHub owner | `<your-github-username>` |
| GitHub repository | `<your-repo-name>` |
| Workflow filename | `publish.yml` |
| Environment | `pypi` |

> **Note:** This creates the project on first publish — you don't need to create it manually.

---

## Step 2: Create GitHub Environment

### 2.1 Create via GitHub API

```bash
# Authenticate with GitHub CLI
gh auth login

# Create 'pypi' environment (or do it manually in repo Settings > Environments)
gh api repos/<owner>/<repo>/environments/pypi -X PUT -H "Accept: application/vnd.github+json"
```

Or manually:
1. Go to `https://github.com/<owner>/<repo>/settings/environments`
2. Click **New environment**
3. Name it: `pypi`
4. Click **Save environment**

---

## Step 3: Create GitHub App for Authentication

Using a GitHub App (instead of a PAT) is recommended because:
- Tokens are generated on-the-fly (no expiry)
- Not tied to a user account
- Granular permissions

### 3.1 Create the GitHub App

1. Go to **<https://github.com/settings/apps/new>**
2. Fill in:
   - **GitHub App name:** `<your-app-name>` (e.g., `my-plugin-release`)
   - **Homepage URL:** `https://github.com/<owner>/<repo>`
   - **Webhook:** Uncheck "Active"
3. **Permissions** (Repository permissions):
   - Contents: **Read and write**
   - Deployments: **Read and write**
4. **Where can this GitHub App be installed?** — leave at default
5. Click **Create GitHub App**
6. Note the **App ID** (shown at top of page)
7. Scroll to **"Private keys"** and click **"Generate a private key"**
8. Download the `.pem` file

### 3.2 Install the App on Your Repository

1. On the GitHub App page, click **Install App**
2. Choose **"Only select repositories"**
3. Select your repository
4. Click **Install**

### 3.3 Add App Credentials as GitHub Secrets

```bash
# Set App ID
gh secret set GH_APP_ID --repo <owner>/<repo> --body "<your-app-id>"

# Set Private Key (from downloaded .pem file)
gh secret set GH_APP_PRIVATE_KEY --repo <owner>/<repo> --body "$(cat ~/Downloads/<your-app-name>.private-key.pem)"
```

---

## Step 4: Add GitHub Actions Workflows

### 4.1 Create `.github/workflows/release.yml`

```yaml
name: Release

on:
  push:
    branches: [main]

permissions:
  contents: write

jobs:
  release:
    if: github.repository == '<owner>/<repo>'
    runs-on: ubuntu-latest
    concurrency: release
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Generate GitHub App installation token
        id: generate-token
        uses: tibdex/github-app-token@v2
        with:
          app_id: ${{ secrets.GH_APP_ID }}
          private_key: ${{ secrets.GH_APP_PRIVATE_KEY }}

      - uses: astral-sh/setup-uv@v6
        with:
          enable-cache: true
          python-version: "3.11"

      - name: Install semantic-release
        run: uv pip install python-semantic-release

      - name: Semantic release
        run: semantic-release publish
        env:
          GH_TOKEN: ${{ steps.generate-token.outputs.token }}
```

### 4.2 Create `.github/workflows/publish.yml`

```yaml
name: Publish to PyPI

on:
  release:
    types: [published]

permissions:
  contents: read

jobs:
  publish:
    runs-on: ubuntu-latest
    environment:
      name: pypi
      url: https://pypi.org/p/<your-plugin-name>
    permissions:
      id-token: write

    steps:
      - name: Download dist artifacts
        uses: actions/download-artifact@v4
        with:
          name: dist
          path: dist/

      - name: Publish to PyPI
        uses: pypa/gh-action-pypi-publish@release/v1
```

### 4.3 Create `.github/workflows/ci.yml`

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: read

jobs:
  pre-commit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v6
        with:
          enable-cache: true
      - uses: pre-commit/action@v3.0.1

  lint:
    needs: pre-commit
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v6
        with:
          enable-cache: true
          python-version: "3.11"
      - run: uv sync --all-extras
      - run: uv run ruff check .
      - run: uv run ruff format --check .

  typecheck:
    needs: pre-commit
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v6
        with:
          enable-cache: true
          python-version: "3.11"
      - run: uv sync --all-extras
      - run: uv run mypy src/

  test:
    needs: pre-commit
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v6
        with:
          enable-cache: true
          python-version: "3.11"
      - run: uv sync --all-extras
      - run: uv run pytest
```

---

## Step 5: Update pyproject.toml

```toml
[project]
name = "<your-plugin-name>"
version = "0.1.0"
description = "<Your plugin description>"
readme = "README.md"
license = { text = "MIT" }
requires-python = ">=3.9"
dependencies = []

[project.optional-dependencies]
dev = [
    "pre-commit>=4.0",
    "pytest>=8.0",
    "ruff>=0.11",
    "mypy>=1.14",
]

[project.entry-points."klippy.extras"]
<extra_name> = "<package_name>.<extra_name>"

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/<package_name>"]

[tool.semantic_release]
version_toml = ["pyproject.toml:project.version"]
branch = "main"
commit_message = "chore(release): {version}"
build_command = "uv build"

[tool.semantic_release.changelog]
changelog_file = "CHANGELOG.md"

[tool.semantic_release.branches.main]
match = "main"
prerelease = false
```

---

## Step 6: Create Initial Version Tag

Before the first release, create the initial tag:

```bash
git tag v0.1.0
git push origin v0.1.0
```

This triggers `release.yml`, which will create the GitHub Release and publish to PyPI.

---

## Step 7: Verify

After pushing the tag, check:

1. **GitHub Actions** — `release.yml` should run and create a GitHub Release
2. **PyPI** — Package should appear at `https://pypi.org/project/<your-plugin-name>/`
3. **Install test** — Run:
   ```bash
   pip install <your-plugin-name>
   ```

---

## Troubleshooting

### Release workflow doesn't trigger publish workflow

This happens when using `GITHUB_TOKEN`. Use a GitHub App or PAT instead:
```yaml
# Wrong - won't trigger downstream workflows
token: ${{ secrets.GITHUB_TOKEN }}

# Correct - triggers downstream workflows
token: ${{ secrets.GH_TOKEN }}  # from PAT
# OR use tibdex/github-app-token action (recommended)
```

### semantic-release fails with "no commits"

Ensure:
1. Tag exists (`git tag v0.1.0`)
2. Tag is pushed (`git push origin v0.1.0`)
3. `commit_message` in pyproject.toml doesn't have `[skip ci]`

### PyPI publish fails with "Trusted Publisher not found"

1. Verify PyPI Trusted Publisher settings match your repo exactly
2. Check that environment name is `pypi` in both PyPI and `publish.yml`

---

## File Structure

```
<your-repo>/
├── .github/
│   └── workflows/
│       ├── ci.yml           # lint, typecheck, test
│       ├── release.yml      # semantic-release → GitHub Release
│       └── publish.yml     # PyPI publish (triggered by release)
├── src/
│   └── <package_name>/
│       ├── __init__.py
│       ├── <extra_name>.py
│       └── version_check.py
├── install.sh
├── pyproject.toml
├── uv.lock
└── README.md
```

---

## References

| Resource | URL |
|----------|-----|
| Kalico plugin docs | <https://docs.kalico.gg/Kalico_Additions.html#plugins> |
| PyPI Trusted Publishing | <https://docs.pypi.org/trusted-publishers/> |
| pypa/gh-action-pypi-publish | <https://github.com/pypa/gh-action-pypi-publish> |
| tibdex/github-app-token | <https://github.com/tibdex/github-app-token> |
| astral-sh/setup-uv | <https://github.com/astral-sh/setup-uv> |
| semantic-release | <https://python-semantic-release.readthedocs.io/> |
