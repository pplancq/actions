# actions

A repository for storing reusable actions.

| Action | Description |
|---|---|
| [`setup-node`](#setup-node) | Set up Node.js and npm (with mise or Volta metadata) and cache `node_modules` |
| [`renovate-update-limits`](#renovate-update-limits) | Update `prConcurrentLimit` and `prHourlyLimit` in `.github/renovate.json` |
| [`nx-release`](#nx-release) | Install, build and run Nx Release with the correct npm dist-tag |

## setup-node

Sets up Node.js and npm, then caches `node_modules`. The repository must be checked out first, because the action reads
`package.json` or the mise configuration.

### Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `use-mise` | no | `false` | `true` installs the tools declared in the repository's mise configuration (`mise.toml`) with `jdx/mise-action`. |
| `node-version` | no | – | Node.js version (without mise). Defaults to `volta.node` from `package.json`. |
| `npm-version` | no | – | npm version (without mise). Defaults to `volta.npm` from `package.json`; installed globally when set. |
| `disabled-cache` | no | `false` | `true` disables the `node_modules` cache. |

### How versions are resolved

| Mode | Node.js / npm versions |
|---|---|
| `use-mise: 'true'` | From `mise.toml` (`[tools]`). `node-version` and `npm-version` are ignored. |
| otherwise | From the inputs, falling back to the `volta` section of `package.json` (read with `jq`). |

### Cache

When enabled, `**/node_modules` is cached with the key `npm-<os>-<workflow name>-modules-<hash of **/package-lock.json>`.
The action only restores the cache: it does not install dependencies, so run `npm ci` yourself afterwards.

### Example

```yaml
steps:
  - name: Checkout repository
    uses: actions/checkout@<sha> # vX

  - name: Setup Node
    uses: pplancq/actions/setup-node@<sha> # vX.Y.Z
    with:
      use-mise: 'true'

  - name: Install dependencies
    run: npm ci
```

Without mise, versions come from `package.json`:

```json
{
  "volta": {
    "node": "24.21.0",
    "npm": "12.2.0"
  }
}
```

## renovate-update-limits

Updates `prConcurrentLimit` and `prHourlyLimit` in `.github/renovate.json`, commits the change and pushes it. Use it
to temporarily slow down (or stop) Renovate pull requests, then restore the shared configuration defaults.

### Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `limit` | yes | `reset` | A non-negative integer applied to both `prConcurrentLimit` and `prHourlyLimit` (`0` = unlimited), or `reset` to delete both keys and fall back to the shared config. Any other value fails the action. |

### Behavior

- `limit: 5` sets both limits to `5` and commits `ci(renovate): set prConcurrentLimit and prHourlyLimit to 5`.
- `limit: reset` removes both keys and commits `ci(renovate): reset PR limits to shared config defaults`.
- If `.github/renovate.json` does not change, nothing is committed.
- The commit is authored by `github-actions[bot]` and pushed to the current branch.

### Requirements

- The repository must be checked out on a branch (not a detached HEAD) with credentials that can push.
- The calling job needs `permissions: contents: write`.
- `jq` must be available (it is on GitHub-hosted runners).
- The branch must allow pushes from `GITHUB_TOKEN` (no protection rule blocking it).

### Example

```yaml
name: Renovate limits

on:
  workflow_dispatch:
    inputs:
      limit:
        description: Limit (integer, 0 = unlimited, or "reset")
        required: true
        default: reset
        type: string

permissions:
  contents: read

jobs:
  update:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - name: Checkout repository
        uses: actions/checkout@<sha> # vX

      - name: Update Renovate limits
        uses: pplancq/actions/renovate-update-limits@<sha> # vX.Y.Z
        with:
          limit: ${{ inputs.limit }}
```

## nx-release

Centralizes the Nx Release process shared by `pplancq/dev-tools`, `pplancq/svg-tools` and `pplancq/shelter-ui`, so that
fixes propagate from a single place and workflows do not drift.

It also selects the npm dist-tag explicitly. `nx release prerelease --preid alpha` only sets the version suffix
(`1.0.0-alpha.0`); without a dist-tag, npm publishes the package under `latest`. The action sets `NPM_CONFIG_TAG` for you.

| Release type | Nx command | npm dist-tag |
|---|---|---|
| `stable` | `nx release --yes` | `latest` |
| `alpha` | `nx release prerelease --yes --preid alpha` | `alpha` |
| `beta` | `nx release prerelease --yes --preid beta` | `beta` |
| `rc` | `nx release prerelease --yes --preid rc` | `rc` |

Changing the dist-tag of versions that are already published is out of scope.

### What the action does

In this order:

1. Validates `release-type` (`stable`, `alpha`, `beta` or `rc`) and computes the npm dist-tag. Fails fast otherwise.
2. Checks out the calling repository with the full Git history (`fetch-depth: 0`).
3. Sets up Node.js and npm from the repository's mise configuration (`mise.toml`) with `jdx/mise-action`.
4. Installs dependencies with `npm ci --ignore-scripts`.
5. Runs the build command (unless `skip-build` is `true`). A failing build stops the job before the release.
6. Configures npm for public publishing with provenance.
7. Runs `npx nx release` with the Git identity, `GITHUB_TOKEN` and the dist-tag.

The action checks out the repository itself (using `github-token`), so the caller must not add a checkout step. The full
history is required because Nx Release needs the tags and commits to compute the next version and the changelog.

### Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `github-token` | yes | – | Token used by Nx Release to push the release commit and tag and to create the GitHub release. Pass `${{ secrets.GITHUB_TOKEN }}`. |
| `release-type` | no | `stable` | `stable`, `alpha`, `beta` or `rc`. Any other value fails the action. |
| `first-release` | no | `false` | `true` adds `--first-release` when no previous release reference is available. |
| `dry-run` | no | `false` | `true` adds `--dry-run`: nothing is committed, tagged or published. |
| `debug` | no | `false` | `true` adds `--verbose`. |
| `build-command` | no | `npm run build` | Command executed after dependency installation. |
| `skip-build` | no | `false` | `true` skips the build step entirely; `build-command` is then ignored. |
| `skip-publish` | no | `false` | `true` adds `--skip-publish`: the version, changelog, tag and GitHub release are created but nothing is published to npm. |
| `git-author-name` | no | `release[bot]` | Git author name of the release commit. |
| `git-author-email` | no | `release[bot]@users.noreply.github.com` | Git author email of the release commit. |
| `git-committer-name` | no | `release[bot]` | Git committer name of the release commit. |
| `git-committer-email` | no | `release[bot]@users.noreply.github.com` | Git committer email of the release commit. |

Composite action inputs are strings: pass booleans as `true` / `false` (a `workflow_dispatch` boolean input is converted
automatically when used in an expression like `${{ inputs.dryRun }}`).

Enable `first-release` explicitly when Nx cannot find a previous release reference, for example for the first release
of a repository or a new prerelease channel with no matching prerelease or stable tag. The action does not detect this
automatically. Nx can fall back to the version in `package.json` and, when no matching tag exists, generate the changelog
from the beginning of the history. Preview the result with `dry-run: true` before publishing, and leave `first-release`
disabled for subsequent releases.

### Caller workflow example

The calling repository keeps its manual release form and its authorization policy.

```yaml
name: Release

on:
  workflow_dispatch:
    inputs:
      releaseType:
        description: Release type
        required: true
        type: choice
        default: stable
        options:
          - stable
          - alpha
          - beta
          - rc
      firstRelease:
        description: First release (allow missing previous release reference)
        required: false
        type: boolean
        default: false
      dryRun:
        description: Dry run
        required: false
        type: boolean
        default: false
      debug:
        description: Debug mode
        required: false
        type: boolean
        default: false

permissions:
  contents: read

jobs:
  release:
    if: github.event_name == 'workflow_dispatch' && github.actor == 'your-github-login'
    name: Release
    runs-on: ubuntu-latest
    permissions:
      contents: write
      issues: write
      pull-requests: write
      id-token: write
    steps:
      - name: Release
        uses: pplancq/actions/nx-release@<sha> # vX.Y.Z
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          release-type: ${{ inputs.releaseType }}
          first-release: ${{ inputs.firstRelease }}
          dry-run: ${{ inputs.dryRun }}
          debug: ${{ inputs.debug }}
          build-command: npm run build

  # Repository-specific jobs (deployment, notifications, ...) stay in the caller.
  deploy:
    if: ${{ !inputs.dryRun }}
    needs: release
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploy here"
```

### Permissions and authentication

Permissions are granted by the calling job, not by the action.

| Permission | Why |
|---|---|
| `contents: write` | Push the release commit and tag, create the GitHub release |
| `issues: write` / `pull-requests: write` | Allow Nx Release to interact with issues and pull requests referenced in the release |
| `id-token: write` | Required for npm trusted publishing (OIDC) and provenance |

Authentication to GitHub uses the repository's own `GITHUB_TOKEN`, passed through `github-token`. No personal access
token and no npm token secret is required.

The release commit is authored by `release[bot]` by default. Override it with the `git-*` inputs, for example with
repository variables:

```yaml
with:
  git-author-name: ${{ vars.GIT_AUTHOR_NAME }}
  git-author-email: ${{ vars.GIT_AUTHOR_EMAIL }}
  git-committer-name: ${{ vars.GIT_COMMITTER_NAME }}
  git-committer-email: ${{ vars.GIT_COMMITTER_EMAIL }}
```

### npm trusted publishing

Publishing uses [npm trusted publishing](https://docs.npmjs.com/trusted-publishers) (OIDC), so no `NPM_TOKEN` is needed.
For each published package, on npmjs.com go to **Settings → Trusted Publisher** and select **GitHub Actions** with:

- **Organization or user**: the owner of the calling repository (for example `pplancq`).
- **Repository**: the calling repository.
- **Workflow filename**: the file of the **calling** workflow (for example `release.yaml`), not this action.
- **Environment**: only if the calling job uses a GitHub environment.

The action configures `access=public` and `provenance=true`, and the calling job must grant `id-token: write`.
Make sure the npm CLI shipped by your toolchain supports trusted publishing (npm 11.5.1 or later).

### Dependency installation policy

Dependencies are always installed with `npm ci --ignore-scripts`. This is fixed and cannot be configured, so dependency
lifecycle scripts (`preinstall`, `install`, `postinstall`, `prepare`, ...) never run during a release, which limits
supply-chain risks.

Any preparation your project normally gets from those scripts (code generation, building a local plugin, setting up
tools, ...) must be run explicitly from `build-command`, for example:

```yaml
build-command: npm run build --workspace @pplancq/nx-release-plugin
```

Because `HUSKY=0` is set during the release, Git hooks do not run either.

### Customizing or disabling the build

| Goal | Configuration |
|---|---|
| Default build | nothing (`npm run build`) |
| Another script | `build-command: npm run build:packages` |
| Workspace build | `build-command: npm run build --workspace @pplancq/nx-release-plugin` |
| Several commands | `build-command: npm run codegen && npm run build` |
| No build | `skip-build: true` |

The build runs in `bash -e -o pipefail` after installation. A failure stops the job and no release is made.
With `skip-build: true`, no build command is executed.

Current adopters:

| Repository | `build-command` |
|---|---|
| `pplancq/shelter-ui` | `npm run build` (default) |
| `pplancq/svg-tools` | `npm run build:packages` |
| `pplancq/dev-tools` | `npm run build --workspace @pplancq/nx-release-plugin` |

### Pinning to a commit SHA

Pin the action to a full commit SHA rather than a mutable tag, and keep the version in a comment so that Renovate can
update it:

```yaml
uses: pplancq/actions/nx-release@66e6d183d1cb9c8f9a4bd1a347c13bca6025ba6a # v1.1.3
```

To find the SHA of a release, open the [releases page](https://github.com/pplancq/actions/releases) and use the commit
linked to the tag, or run:

```bash
git ls-remote https://github.com/pplancq/actions 'refs/tags/vX.Y.Z^{}'
```

The action must be available in the version you pin (use the first release that contains `nx-release`).

### Dry release before adopting

Before the first real release with the action:

1. Run your caller workflow manually (**Actions → Release → Run workflow**) with `release-type: stable`,
   `dry-run: true` and `debug: true`.
2. Check the logs:
   - the log `npm dist-tag: latest` (or `alpha`, `beta`, `rc` for a prerelease);
   - the `Running: npx nx release ...` line with `--dry-run`;
   - the computed version, changelog preview and tag;
   - the build step ran (or was skipped) as expected.
3. Repeat with a prerelease type (for example `alpha`) and confirm the dist-tag is `alpha`.
4. Nothing is committed, tagged or published in dry-run mode. Run without `dry-run` when everything looks right.

### Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `Invalid release-type '...'` | `release-type` is not `stable`, `alpha`, `beta` or `rc` |
| Nx cannot find the previous release or tags | Do not add your own checkout step: it would not be needed and a shallow one would hide the tags |
| Build failure, no release | Expected: the build must succeed before the release. Fix the build or `build-command` |
| Missing generated files or tools at build time | Lifecycle scripts are disabled; run the preparation in `build-command` |
| npm `E404` / `ENEEDAUTH` / OIDC error on publish | Check the trusted publisher settings (repository, calling workflow filename) and `id-token: write` |
| Push to the protected branch rejected | Allow `GITHUB_TOKEN` (or the release bot) to bypass the branch protection for the release commit |
| Prerelease published under `latest` | Make sure you use this action (it sets the dist-tag) and that no other `NPM_CONFIG_TAG` overrides it |
