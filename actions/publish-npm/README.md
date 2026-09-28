# publish-npm

Composite GitHub Action that builds and publishes an npm package to the npm
registry via **OIDC Trusted Publishing** (no long-lived `NPM_TOKEN` needed).

It handles: computing/validating the target version (from a short git SHA or
from a pushed tag), loading build/test commands from a `.github-ci.yml` file
in the consuming repo, an optional test gate, overwriting `package.json`'s
version, building, and publishing `dist/`.

## Requirements on the consuming repo

1. A `.github-ci.yml` file at the repo root:
   ```yaml
   NODE_VERSION: "20"
   DEV_BUILD_COMMANDS: "npm ci && npm run build"   # used when build-kind: dev
   PROD_BUILD_COMMANDS: "npm ci && npm run build"  # used when build-kind: prod
   TEST_RUN_COMMANDS: ""                            # empty = skip test gate
   ```
   The build command's own `postbuild` step (or equivalent) is expected to
   produce a `dist/` directory containing a `package.json` — that's what
   `npm publish` runs against.
2. `package.json` must declare a `repository.url` matching the GitHub repo
   (npm's provenance verification checks this), e.g.:
   ```json
   "repository": { "type": "git", "url": "git+https://github.com/OWNER/REPO.git" }
   ```
3. An npm **Trusted Publisher** entry configured on npmjs.com for the
   package, matching: Publisher = GitHub Actions, this exact repo, and the
   **workflow filename** of the calling workflow (not this action's path).
   If the Trusted Publisher entry has an "Environment name" set, the calling
   job must declare a matching `environment:` — otherwise npm can't match
   the OIDC token and publish fails with a 404.

## Requirements on the calling workflow

Because this is referenced as `owner/repo/actions/...@ref` (not a local
`./` path), GitHub Actions downloads the action independently — but the
action's steps still operate on the calling repo's checked-out files, so
the calling job must run `actions/checkout` before calling this action, and
must grant `permissions: id-token: write` for OIDC to work.

```yaml
jobs:
  publish:
    runs-on: ubuntu-latest
    environment: STAGING   # only if your Trusted Publisher entry sets one
    permissions:
      contents: read
      id-token: write
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0   # only needed when version-mode: tag + require-ancestor-branch

      - uses: ngthucdotcom/gh-actions-templates/actions/publish-npm@v1
        with:
          version-mode: short-sha        # or 'tag'
          version-prefix: 1.0.0-rc       # required when version-mode: short-sha
          # tag-pattern: '^v[0-9]+\.[0-9]+\.[0-9]+$'   # used when version-mode: tag
          # require-ancestor-branch: master             # used when version-mode: tag
          npm-dist-tag: rc                # empty/omit = publish as "latest"
          build-kind: dev                 # or 'prod' -> reads PROD_BUILD_COMMANDS
          run-tests: 'true'
```

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `version-mode` | yes | – | `short-sha` or `tag` |
| `version-prefix` | when `version-mode: short-sha` | `''` | e.g. `1.0.0-dev` -> published as `1.0.0-dev-<7charsha>` |
| `tag-pattern` | no | `^v[0-9]+\.[0-9]+\.[0-9]+$` | Bash regex the pushed tag must match (used when `version-mode: tag`) |
| `require-ancestor-branch` | no | `''` | When `version-mode: tag`, the tagged commit must be reachable from this branch. Empty skips the check |
| `npm-dist-tag` | no | `''` | `npm publish --tag <value>`. Empty publishes as `latest` |
| `build-kind` | yes | – | `dev` (reads `DEV_BUILD_COMMANDS`) or `prod` (reads `PROD_BUILD_COMMANDS`) from `.github-ci.yml` |
| `run-tests` | no | `'true'` | Whether to run `TEST_RUN_COMMANDS` (if set in `.github-ci.yml`) before building |

## Versioning

Tagged releases (`v1`, `v1.0.0`, ...) are the supported way to consume this
action — pin `@v1` rather than `@master` so changes here don't silently
change behavior in every consuming repo.
