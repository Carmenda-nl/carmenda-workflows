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

### `node-tests.yml`

Lints, checks formatting and type-checks a Node project managed with [pnpm](https://pnpm.io/).
Runs ESLint, a Prettier formatting check, then the TypeScript compiler in check-only mode.

**Inputs**

| Name                 | Description                                            | Required |
| -------------------- | ------------------------------------------------------ | -------- |
| `working-directory`  | Directory containing the Node project (`package.json`) | Yes      |

**Usage**

```yaml
jobs:
  tests:
    uses: Carmenda-nl/carmenda-workflows/.github/workflows/node-tests.yml@main
    with:
      working-directory: privacytool
```

### `python-release-beta.yml`

Builds a Windows PyInstaller executable and publishes it as a beta GitHub release (prerelease). 
Also updates `CHANGELOG.md` and the version file on `develop`.

**Inputs**

| Name                | Description                                        | Required | Default                |
| ------------------- | -------------------------------------------------- | -------- | ---------------------- |
| `working-directory` | Directory containing `pyproject.toml`              | No       | `app`                  |
| `build-name`        | Name of the PyInstaller output folder under `dist` | Yes      | —                      |
| `version-file`      | Path to the version file                           | No       | `app/main/_version.py` |

**Secrets**: `secrets: inherit`

**Usage**

```yaml
jobs:
  release:
    uses: Carmenda-nl/carmenda-workflows/.github/workflows/python-release-beta.yml@main
    with:
      working-directory: app
      build-name: my-product
      version-file: app/main/_version.py
    secrets: inherit
```

### `node-release-beta.yml`

Builds a Windows installer for an Electron app that bundles a separately-released
API gateway and one or more separately-released engines, then publishes it
as a beta GitHub release (prerelease). Also updates `CHANGELOG.md` on `develop`.

**Inputs**

| Name                | Description                               | Required | Default       |
| ------------------- | ----------------------------------------- | -------- | ------------- |
| `version`           | Release tag to publish                    | Yes      | -             |
| `build-name`        | Directory containing `package.json`       | No       | `privacytool` |
| `api-gateway-repo`  | Repo of the API gateway release to bundle | Yes      | -             |
| `engines`           | JSON array of engines to bundle           | Yes      | -             |

**Secrets**: `secrets: inherit`

**Usage**

```yaml
jobs:
  release:
    uses: Carmenda-nl/carmenda-workflows/.github/workflows/node-release-beta.yml@main
    with:
      build-name: privacytool
      version: ${{ github.ref_name }}
      api-gateway-repo: Carmenda-nl/privacytool-api-gateway
      engines: |
        [
          {"name": "carmenda-deduce-engine", "repo": "Carmenda-nl/carmenda-deduce-engine"},
          {"name": "other-engine", "repo": "Carmenda-nl/other-engine"}
        ]
    secrets: inherit
```

### `python-prepare-stable.yml`

Strips the `-beta` suffix from the version file and `CHANGELOG.md` on `develop`, 
so that `main` receives the correct stable version once the PR is merged manually after approval.
Commits and pushes the change to `develop`. Runs on PR open/update.

**Inputs**

| Name           | Description              | Default                |
| -------------- | ------------------------ | ---------------------- |
| `version-file` | Path of the version file | `app/main/_version.py` |

**Usage**

```yaml
on:
  pull_request:
    types: [opened, synchronize]

jobs:
  prepare-stable-version:
    if: >
      github.event.pull_request.base.ref == 'main' &&
      github.event.pull_request.head.ref == 'develop'
    uses: Carmenda-nl/carmenda-workflows/.github/workflows/python-prepare-stable.yml@main
    with:
      version-file: app/main/_version.py
```

### `python-release-stable.yml`

Promotes the latest beta release to a stable release: 
downloads the beta assets, patches the bundled version file to strip the `-beta` suffix, repackages them, 
creates a new stable release, deletes the beta release/tag and updates `CHANGELOG.md` on both `develop` and `main`.

**Inputs**

| Name           | Description                                    | Default                |
| -------------- | ---------------------------------------------- | ---------------------- |
| `version-file` | Path of the version file inside the app source | `app/main/_version.py` |

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
    with:
      version-file: app/main/_version.py
    secrets: inherit
```
