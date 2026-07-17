# carmenda-workflows

Central repository for reusable GitHub Actions workflows used across Carmenda repositories.

## Workflows

### `python-tests.yml`

Lints, type-checks and tests a Python project managed with [uv](https://github.com/astral-sh/uv). 
Requires a `.python-version` file in the working directory. 
Runs Ruff (check + format), Mypy, then pytest, and uploads the coverage report as an artifact.

**Inputs**

| Name                | Description                                                    | Default |
| ------------------- | -------------------------------------------------------------- | ------- |
| `working-directory` | Directory containing the Python project (`pyproject.toml`)     | `app`   |
| `pre-test-command`  | Optional command to run before pytest (e.g. Django migrations) | `''`    |

**Usage**

```yaml
jobs:
  tests:
    uses: Carmenda-nl/carmenda-workflows/.github/workflows/python-tests.yml@main
    with:
      working-directory: app
```

### `python-release-beta.yml`

Builds a Windows PyInstaller executable and publishes it as a beta GitHub release (prerelease). 
Also updates `CHANGELOG.md` and the version file on `develop`.

**Inputs**

| Name                | Description                                         | Required | Default                |
| ------------------- | --------------------------------------------------- | -------- | ---------------------- |
| `working-directory` | Directory containing `pyproject.toml`               | No       | `app`                  |
| `build-output-name` | Name of the PyInstaller output folder under `dist`  | Yes      | —                      |
| `artifact-name`     | Name of the GitHub Actions upload/download artifact | Yes      | —                      |
| `zip-prefix`        | Prefix for the release zip                          | Yes      | —                      |
| `version-file`      | Path to the version file                            | No       | `app/main/_version.py` |

**Secrets**: `secrets: inherit`

**Usage**

```yaml
jobs:
  release:
    uses: Carmenda-nl/carmenda-workflows/.github/workflows/python-release-beta.yml@main
    with:
      working-directory: app
      build-output-name: my-product
      artifact-name: my-product
      zip-prefix: My_product_win
      version-file: app/main/_version.py
    secrets: inherit
```

### `python-prepare-stable.yml`

Strips the `-beta` suffix from the version file and `CHANGELOG.md` on `develop`, 
so that `main` receives the correct stable version once the PR is merged. Commits and pushes the change to `develop`.

**Inputs**

| Name           | Description              | Default                |
| -------------- | ------------------------ | ---------------------- |
| `version-file` | Path of the version file | `app/main/_version.py` |

**Usage**

```yaml
jobs:
  prepare-stable-version:
    if: >
      github.event.review.state == 'approved' &&
      github.event.pull_request.base.ref == 'main' &&
      github.event.pull_request.head.ref == 'develop'
    uses: Carmenda-nl/carmenda-workflows/.github/workflows/python-prepare-stable.yml@main
    with:
      version-file: app/main/_version.py
```

### `python-release-stable.yml`

Promotes the latest beta release to a stable release: 
downloads the beta assets, renames them, creates a new stable release, deletes the beta release/tag and 
updates `CHANGELOG.md` on both `develop` and `main`.

**Inputs**: none

**Secrets**: `secrets: inherit`

**Usage**

```yaml
jobs:
  promote-beta-to-stable:
    if: >
      github.event.pull_request.merged == true &&
      github.event.pull_request.base.ref == 'main' &&
      github.event.pull_request.head.ref == 'develop'
    uses: Carmenda-nl/carmenda-workflows/.github/workflows/python-release-stable.yml@main
    secrets: inherit
```
