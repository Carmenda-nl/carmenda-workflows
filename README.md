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

| Name                 | Description                                                  | Required |
| -------------------- | ------------------------------------------------------------ | -------- |
| `working-directory`  | Directory containing the Node project (`package.json`)       | Yes      |

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

| Name                | Description                            | Required | Default              |
| ------------------- | -------------------------------------- | -------- | -------------------- |
| `working-directory` | Directory containing `pyproject.toml`  | No       | `app`                |
| `build-name`        | Name of the PyInstaller output folder  | Yes      | -                    |
| `version-file`      | Path to the version file               | No       | `app/main/VERSION`   |

**Usage**

```yaml
jobs:
  release:
    uses: Carmenda-nl/carmenda-workflows/.github/workflows/python-release-beta.yml@main
    with:
      working-directory: app
      build-name: my-product
      version-file: app/main/VERSION
    secrets: inherit
```

### `node-release-beta.yml`

Builds a Windows installer for an Electron app that bundles the API gateway 
and one or more engines, then publishes it
as a beta GitHub release (prerelease). Also updates `CHANGELOG.md` on `develop`.

**Inputs**

| Name                | Description                                          | Required | Default |
| ------------------- | ---------------------------------------------------- | -------- | ------- |
| `version`           | Release tag to publish                               | Yes      | -       |
| `build-name`        | Directory containing `package.json`                  | No       | `app`   |
| `version-file`      | Path to the `VERSION` file.                          | Yes      | -       |
| `api-gateway-repo`  | Name of the API gateway repo to bundle               | Yes      | -       |
| `engines`           | JSON array of engines to bundle                      | Yes      | -       |

**Usage**

```yaml
jobs:
  release:
    uses: Carmenda-nl/carmenda-workflows/.github/workflows/node-release-beta.yml@main
    with:
      build-name: privacytool
      version: ${{ github.ref_name }}
      version-file: privacytool/VERSION
      api-gateway-repo: privacytool-api-gateway
      engines: |
        [
          {"repo": "carmenda-deduce-engine", "appTitle": "Carmenda deduce engine", "default": true},
          {"repo": "other-engine", "appTitle": "Other engine"}
        ]
    secrets: inherit
```

**How the gateway/engine release is picked**

1. The base version, compared numerically (`v1.2.4-beta` beats `v1.2.3`, since `4 > 3`
   The beta/stable suffix only matters as a tiebreaker, see next point);
2. Whether it's a stable release (only used to break ties *within the same base
   version* — e.g. `v1.2.3` beats `v1.2.3-beta`).

The highest-sorting tag is downloaded. In practice this means: **the newest available
release wins, beta or not**

### `prepare-stable.yml`

Strips the `-beta` suffix from the version file and `CHANGELOG.md` on `develop`, 
so that `main` receives the correct stable version once the PR is merged manually after approval.

**Inputs**

| Name             | Description                                     | Required | Default        |
| ---------------- | ----------------------------------------------- | -------- | -------------- |
| `version-file`   | Path of the version file                        | Yes      | -              |
| `changelog-file` | Path to the changelog file                      | No       | `CHANGELOG.md` |

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
    uses: Carmenda-nl/carmenda-workflows/.github/workflows/prepare-stable.yml@main
    with:
      version-file: app/main/VERSION
```

### `python-release-stable.yml`

Promotes the latest beta release to a stable release: 
downloads the beta assets, patches the bundled version file to strip the `-beta` suffix,
repackages them, creates a new stable release, deletes the beta release/tag and 
updates `CHANGELOG.md` on both `develop` and `main`.

**Inputs**

| Name           | Description                                             | Default             |
| -------------- | ------------------------------------------------------- | ------------------- |
| `version-file` | Path of the version file inside the app source          | `app/main/VERSION`  |

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
      version-file: app/main/VERSION
    secrets: inherit
```

### `node-release-stable.yml`

Promotes the latest beta release to a stable release for a node/Electron app. Unlike
`python-release-stable.yml`, it doesn't patch any file contents — the release asset is
a compiled NSIS `setup.exe` wrapped in a zip.

**Inputs**

| Name             | Description                                     | Required | Default        |
| ---------------- | ----------------------------------------------- | -------- | -------------- |
| `version-file`   | Path to the `VERSION` file                      | Yes      | -              |
| `changelog-file` | Path to the changelog file                      | No       | `CHANGELOG.md` |

**Secrets**: `secrets: inherit`

**Usage**

```yaml
on:
  pull_request:
    types: [closed]
    branches:
      - main

permissions:
  contents: write

jobs:
  promote-beta-to-stable:
    if: >-
      github.event.pull_request.merged == true &&
      github.event.pull_request.base.ref == 'main' &&
      github.event.pull_request.head.ref == 'develop'
    uses: Carmenda-nl/carmenda-workflows/.github/workflows/node-release-stable.yml@main
    with:
      version-file: privacytool/VERSION
    secrets: inherit
```
