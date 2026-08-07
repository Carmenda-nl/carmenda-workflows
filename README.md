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
| `api-gateway-repo`  | Name of the API gateway repo to bundle (bare name, same org as the caller) | Yes | - |
| `engines`           | JSON array of engines to bundle, each shaped `{"repo", "appTitle", "default"?}` — `repo` is a bare name (same org as the caller) and also doubles as the engine's id; exactly one entry must set `"default": true` | Yes | - |

**Secrets**: `secrets: inherit`

**Usage**

```yaml
jobs:
  release:
    uses: Carmenda-nl/carmenda-workflows/.github/workflows/node-release-beta.yml@main
    with:
      build-name: privacytool
      version: ${{ github.ref_name }}
      api-gateway-repo: privacytool-api-gateway
      engines: |
        [
          {"repo": "carmenda-deduce-engine", "appTitle": "Carmenda deduce engine", "default": true},
          {"repo": "other-engine", "appTitle": "Other engine"}
        ]
    secrets: inherit
```

Repo names passed to `api-gateway-repo` and each engine's `repo` are bare (no
`org/` prefix) — the workflow resolves them under `github.repository_owner`,
which in a reusable workflow reflects the *calling* repo's org. This only works
because every consumer of this workflow lives in the same GitHub org.

There's no separate "engine id/name" field — an engine's `repo` doubles as its
id (used for its `backend/engines/<repo>` folder and its `engines.json` entry),
since by convention each engine repo's own `release-beta.yml` sets `build-name`
to its own repo name, which becomes the PyInstaller binary name. So the repo
name is already the canonical id; keeping a separate field would just be two
copies of the same value.

**How the gateway/engine release is picked**

For the API gateway and for each engine, the workflow resolves which release to
bundle itself — it does *not* use `gh release download` without a tag, since that
only ever looks at `/releases/latest`, which excludes prereleases (so a beta would
never be found). Instead it lists all releases of the target repo, keeps only tags
matching `v<major>.<minor>.<patch>` (optionally followed by e.g. `-beta`), and sorts
by:

1. the base version, compared numerically (`v1.2.4-beta` beats `v1.2.3`, since `4 > 3`
   — the beta/stable suffix only matters as a tiebreaker, see next point);
2. whether it's a stable release (only used to break ties *within the same base
   version* — e.g. `v1.2.3` beats `v1.2.3-beta`).

The highest-sorting tag is downloaded. In practice this means: **the newest available
release wins, beta or not** — a beta release of the gateway/engine with a higher
version than the latest stable will be picked up automatically on the next
`node-release-beta.yml` run. This is not a pinned/matching version to the Electron
app's own tag; it always resolves independently at build time. If no matching
release exists for a repo, the build fails.

Only the Windows asset is downloaded (`gh release download -p "*win*.zip"`). The
gateway is extracted to `backend/gateway`; each engine is extracted to
`backend/engines/<name>`, where `<name>` is the `name` field from the `engines`
input — this must match the engine's `id` in the app's `engines.json` so
`backendManager.ts` can find the right folder/binary at runtime.

The resolved gateway and engine tags are listed under a "Bundled versions"
section in the published release's notes, so it's visible at a glance which
gateway/engine build is bundled without having to inspect `engines.json`.

### `create-release.yml`

Internal reusable workflow shared by `node-release-beta.yml` and
`python-release-beta.yml` — not meant to be called directly from a product
repository. Downloads a build artifact, zips it, generates release notes with
[git-cliff](https://git-cliff.org/), creates the GitHub release, and (unless
an entry for this version already exists) prepends `CHANGELOG.md` and pushes
the commit.

**Inputs**

| Name             | Description                                                                             | Required | Default       |
| ---------------- | ---------------------------------------------------------------------------------------- | -------- | ------------- |
| `artifact-name`  | Name of the build artifact to download                                                   | Yes      | —             |
| `zip-name`       | Filename for the release zip                                                             | Yes      | —             |
| `flatten`        | `true`: `zip -j` (junk paths, flat file list); `false`: `zip -r` (preserve directory structure) | No | `false` |
| `release-tag`    | Git tag for the release                                                                  | Yes      | —             |
| `release-name`   | Release title                                                                            | Yes      | —             |
| `is-prerelease`  | Whether this is a prerelease                                                             | Yes      | —             |
| `extra-notes`    | Extra markdown appended to the release body (below the git-cliff notes) — not included in the CHANGELOG.md entry | No | `''` |
| `version-file`   | If set, also write the tag into this file and include it in the changelog commit         | No       | `''`          |
| `changelog-file` | Path to the changelog file                                                               | No       | `CHANGELOG.md` |
| `push-branch`    | Branch to push the changelog (and version-file) commit to                                | No       | `develop`     |

**Secrets**: `secrets: inherit`

**Commit message conventions**

Release notes and `CHANGELOG.md` entries are generated by [git-cliff](https://git-cliff.org/)
using the shared [`cliff.toml`](cliff.toml) in this repo (fetched at release time, so it
applies to every consumer without needing its own config). Commits are grouped by the
prefix of their message, case-insensitive:

| Prefix                          | Changelog group |
| -------------------------------- | ---------------- |
| `feat`, `add`                    | `Added`          |
| `fix`, `bug`                     | `Fixed`          |
| `change`, `update`, `upgrade`, `bump` | `Changed`   |

`merge` commits and `chore(release):`/`bump: version to v...` commits are dropped entirely.
Any commit whose message doesn't start with one of the prefixes above is **also dropped** —
it will not show up in the release notes or `CHANGELOG.md` at all, so use one of the
prefixes above for anything that should be user-visible.

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
