# Dev-kitx Reusable Workflows

Centralised GitHub Actions workflows shared across all Dev-kitx repositories.

---

## Table of Contents

- [github-release](#github-release)
- [python-publish-pypi](#python-publish-pypi)
- [run-npm-vite-tests](#run-npm-vite-tests)
- [vscode-extension-release](#vscode-extension-release)

---

## `github-release`

**File:** `.github/workflows/github-release.yml`

Maintains a "Release PR" that bumps `package.json` and auto-generates a `CHANGELOG` from [Conventional Commits](https://www.conventionalcommits.org/). When the Release PR is merged the workflow creates the git tag and GitHub Release via [`Dev-kitx/smart-gh-release`](https://github.com/Dev-kitx/smart-gh-release).

### Version bump rules

| Commit prefix                 | Version bump            |
|-------------------------------|-------------------------|
| `feat:`                       | minor (`0.1.0 → 0.2.0`) |
| `fix:`                        | patch (`0.1.0 → 0.1.1`) |
| `feat!:` / `BREAKING CHANGE:` | major (`0.1.0 → 1.0.0`) |
| `chore:`, `docs:`, `style:`   | no Release PR update    |

### One-time setup

Create a [GitHub App](https://docs.github.com/en/apps/creating-github-apps) and add the following to the calling repo:

| Secret / Variable     | Description                        |
|-----------------------|------------------------------------|
| `vars.APP_ID`         | The app's numeric ID               |
| `secrets.PRIVATE_KEY` | The app's private key (PEM format) |

### Inputs

| Input                   | Type      | Required | Default | Description                                                                                                                                                   |
|-------------------------|-----------|----------|---------|---------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `auto_version`          | `boolean` | No       | `true`  | Determine the next version from Conventional Commit messages. Set to `false` to specify the version manually in the Release PR title (e.g. `Release v1.2.3`). |
| `auto_release`          | `boolean` | **Yes**  | `false` | Automatically publish the release after creating it.                                                                                                          |
| `generate_badge`        | `boolean` | No       | `true`  | Generate a badge for the release.                                                                                                                             |
| `bump_version_in_files` | `string`  | No       | `""`    | Comma-separated list of file paths whose version field should be updated.                                                                                     |

### Usage

```yaml
jobs:
  release:
    uses: Dev-kitx/.github/.github/workflows/github-release.yml@main
    with:
      auto_version: true
      auto_release: false
      generate_badge: true
      bump_version_in_files: "pyproject.toml,src/version.py"
    secrets: inherit
```

---

## `python-publish-pypi`

**File:** `.github/workflows/python-publish-pypi.yml`

Runs your test suite, builds an `sdist` + `wheel`, and publishes to [PyPI](https://pypi.org/) using **Trusted Publishing (OIDC)** — no API tokens required.

### Pipeline stages

1. **Tests** — runs `test_command`, blocks the rest of the pipeline on failure.
2. **Build** — runs `build_command`, uploads `dist/` as an artifact.
3. **Publish** — downloads the artifact and publishes via `pypa/gh-action-pypi-publish`.

### One-time setup

Configure a Trusted Publisher on PyPI at <https://pypi.org/manage/account/publishing/>:

| Field       | Value                                                     |
|-------------|-----------------------------------------------------------|
| Publisher   | GitHub Actions                                            |
| Owner       | `<your-github-username-or-org>`                           |
| Repository  | `<repo-name>`                                             |
| Workflow    | The **caller** workflow filename (e.g. `publish.yml`)     |
| Environment | Must match the `environment_name` input (default: `pypi`) |

### Inputs

| Input                     | Type     | Required | Default                                                                              | Description                                                                                                        |
|---------------------------|----------|----------|--------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| `package_name`            | `string` | **Yes**  | —                                                                                    | PyPI package name, used to construct the environment URL `https://pypi.org/p/<package_name>`.                      |
| `python_version`          | `string` | No       | `"3.11"`                                                                             | Python version used for testing and building.                                                                      |
| `test_command`            | `string` | No       | `pip install -e ".[dev]" pytest pytest-cov && python -m pytest tests/ --tb=short -q` | Command(s) to install dependencies and run the test suite. A non-zero exit blocks publishing.                      |
| `build_command`           | `string` | No       | `pip install build && python -m build`                                               | Command(s) that produce distribution files under `dist/`.                                                          |
| `fetch_depth`             | `number` | No       | `0`                                                                                  | `git` checkout fetch-depth. Use `0` (full history) when tag-based versioning tools like `setuptools-scm` are used. |
| `environment_name`        | `string` | No       | `pypi`                                                                               | GitHub deployment environment name.                                                                                |
| `artifact_retention_days` | `number` | No       | `7`                                                                                  | Days to retain the `dist` artifact before GitHub deletes it.                                                       |

### Rollback / revert strategy

PyPI does **not** allow re-uploading or deleting a released version. Two safe options:

1. **Yank** *(preferred for bugs)* — marks the version as bad; `pip` skips it for loose installs but pinned installs still resolve it. Reversible. Use the PyPI project dashboard or a "Yank Release" workflow.
2. **Patch release** *(preferred for critical fixes)* — fix the bug, cut a new patch version (e.g. `1.2.1`), and publish that. Optionally yank the bad version at the same time.

### Usage

```yaml
jobs:
  publish:
    uses: Dev-kitx/.github/.github/workflows/python-publish-pypi.yml@main
    with:
      package_name: my-package
      python_version: "3.12"
      test_command: "make setup-env && make run-pytest"
      build_command: "pip install build && python -m build"
    secrets: inherit
```

---

## `run-npm-vite-tests`

**File:** `.github/workflows/run-npm-vite-tests.yml`

Installs Node.js dependencies and runs typechecking, bundling, and tests (with coverage) for NPM/Vite projects. Optionally uploads coverage reports to [Codecov](https://codecov.io/).

### Steps

1. Checkout code
2. Set up Node.js (configurable version) with `npm` cache
3. `npm ci`
4. `npm run typecheck`
5. `npm run bundle`
6. `npm run test:coverage`
7. Upload coverage to Codecov (skipped when `codecov_slug` is empty)

### Inputs

| Input          | Type     | Required | Default | Description                                                                      |
|----------------|----------|----------|---------|----------------------------------------------------------------------------------|
| `node-version` | `string` | No       | `"22"`  | Node.js version to use.                                                          |
| `codecov_slug` | `string` | No       | `""`    | Codecov slug in `owner/repo` format. Coverage upload is skipped when left empty. |

### Secrets

| Secret          | Required | Description                                      |
|-----------------|----------|--------------------------------------------------|
| `CODECOV_TOKEN` | **Yes**  | Token for uploading coverage reports to Codecov. |

### Usage

```yaml
jobs:
  test:
    uses: Dev-kitx/.github/.github/workflows/run-npm-vite-tests.yml@main
    with:
      node-version: "22"
      codecov_slug: "my-org/my-repo"
    secrets:
      CODECOV_TOKEN: ${{ secrets.CODECOV_TOKEN }}
```

---

## `vscode-extension-release`

**File:** `.github/workflows/vscode-extension-release.yml`

Runs tests as a pre-publish gate, then builds a `.vsix`, publishes it to the [VS Code Marketplace](https://marketplace.visualstudio.com/), and (on `release` events) attaches the `.vsix` to the GitHub Release and commits the version bump back to `main`.

### Pipeline stages

1. **Tests** — typechecks and runs `npm test`. Publishing is blocked on failure.
2. **Build & Publish** — bundles the extension, packages the `.vsix`, publishes to the Marketplace, and (on `release` events) uploads the `.vsix` to the GitHub Release and commits the version back to `main`.

### One-time setup

| Secret     | Description                                                                                                                                                        |
|------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `VSCE_PAT` | Personal Access Token for the VS Code Marketplace (`vsce` publisher). Generate one at [marketplace.visualstudio.com](https://marketplace.visualstudio.com/manage). |

### Inputs

| Input          | Type     | Required | Default | Description             |
|----------------|----------|----------|---------|-------------------------|
| `node-version` | `string` | No       | `"22"`  | Node.js version to use. |

### Secrets

| Secret     | Required | Description                                |
|------------|----------|--------------------------------------------|
| `VSCE_PAT` | **Yes**  | VS Code Marketplace Personal Access Token. |

### Usage

```yaml
jobs:
  release:
    uses: Dev-kitx/.github/.github/workflows/vscode-extension-release.yml@main
    with:
      node-version: "22"
    secrets:
      VSCE_PAT: ${{ secrets.VSCE_PAT }}
```

> **Note:** This workflow expects your repo to have the following npm scripts defined:
> `typecheck`, `test`, `bundle`.
