# SDK workflows

Two repos are involved:

1. **Config repo** (has `fern/generators.yml`). Its workflow runs `fern generate --group <g> --version <v>`; Fern generates remotely and pushes code to the SDK repo (and, if `output` has a registry with a token, publishes directly).
2. **SDK repo** (generated). Contains Fern-generated `.github/workflows/ci.yml` that builds/tests on PRs and publishes to the registry on a release/tag, using that repo's secrets (`NPM_TOKEN`, `PYPI_TOKEN`, …) or OIDC trusted publishing.

Decide the split first: either Fern publishes (registry token in `generators.yml` `output.token` as `${VAR}`, resolved from Fern's org secrets / the environment running `fern generate`) **or** the SDK repo's workflow publishes. Don't do both.

## Release from the config repo on demand

```yaml
name: Release TypeScript SDK
on:
  workflow_dispatch:
    inputs:
      version:
        description: "Version to publish (e.g. 1.4.0)"
        required: true
        type: string
jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20 }
      - uses: fern-api/setup-fern-cli@v1
      - run: fern check
      - run: fern generate --group ts-sdk --version "$VERSION" --force --log-level debug
        env:
          FERN_TOKEN: ${{ secrets.FERN_TOKEN }}
          VERSION: ${{ inputs.version }}     # pass inputs via env, never inline `${{ inputs.* }}` in `run:` (shell injection)
```

What happens next depends on `github.mode` in `generators.yml` (details in the `fern-sdks` skill):

| Mode | Effect of the job | Publish step |
| --- | --- | --- |
| `release` | Commits to the SDK repo's default branch and creates tag + GitHub release `<version>` | SDK repo's `ci.yml` publish job runs on the release |
| `pull-request` | Opens a PR in the SDK repo | Merge the PR, then create the release/tag (manually or with a workflow) to publish |
| `push` | Pushes to `branch`, no tag | Your own release process |

Trigger alternatives: `on: push: tags: ['v*']` and pass `${GITHUB_REF_NAME#v}` as the version; or `on: push: paths: ['fern/**']` without `--version` to open PRs with regenerated code (pull-request mode) and release separately.

Multiple languages: one job per group (matrix over `group`), or an alias (`aliases: { all: [ts-sdk, python-sdk] }`) and `--group all`. Give each language its own version input if they're versioned independently.

## Publishing from the SDK repo

Fern generates a workflow in the SDK repo that publishes on release. Secrets it expects are per language and documented at `https://buildwithfern.com/learn/sdks/generators/<lang>/publishing`:

- npm: prefer OIDC trusted publishing (`output.token: OIDC` in `generators.yml`, TypeScript generator ≥ 3.12.3; configure the trusted publisher on npmjs.com) — classic long-lived tokens are deprecated. Otherwise `NPM_TOKEN`.
- PyPI: `PYPI_TOKEN` (or trusted publishing where supported).
- Maven Central: `MAVEN_USERNAME`, `MAVEN_PASSWORD`, GPG signing key/password.
- NuGet: `NUGET_API_KEY`. RubyGems: `RUBYGEMS_API_KEY`. Packagist: webhook on the repo, no token. Go: tag push is the publish.

If you customise the generated workflow, add `.github/workflows/ci.yml` to the SDK repo's `.fernignore`, or Fern will overwrite it. To stop Fern generating workflows at all set `github.workflows: false`.

## Preview / dry-run SDK generation on PRs

```yaml
- run: fern generate --group ts-sdk --preview
  env:
    FERN_TOKEN: ${{ secrets.FERN_TOKEN }}
- run: cd fern/.preview/fern-typescript-sdk && npm ci && npm run build && npm test
```

Nothing is pushed or published; the generated SDK lands in `fern/.preview/<generator>/`. Upload it as an artifact for reviewers if useful.

## Self-hosted / local generation in CI

```yaml
- run: fern generate --group ts-sdk --local --force --package
  env:
    FERN_TOKEN: ${{ secrets.FERN_TOKEN }}
  # artifacts in <output.path>/fern-dist/ for each local-file-system output, e.g. SDKs/acme-node-sdk/fern-dist/acme-1.0.0.tgz
```

Requires a Docker-capable runner (GitHub-hosted Ubuntu runners have Docker). `--local` still needs `FERN_TOKEN` for organization verification (see https://buildwithfern.com/learn/sdks/deep-dives/self-hosted); it just doesn't use Fern's remote generation service. Use `image.registry` in `generators.yml` if generator images are mirrored to a private registry, and `--package` to produce installable artifacts (`.tgz`, wheel, jar, nupkg, gem) for internal registries. `--package-only` leaves only `fern-dist/` behind in each output directory. Local generation can also push to GitHub when `github` is configured and a `GITHUB_TOKEN` with write access is in the environment, but Replay and the Fern GitHub App integration are remote-only features.

## Autorelease

With `autorelease: true` (top level or per generator) and the Fern GitHub App installed on both repos, Fern regenerates and releases when the spec changes in the config repo — no release workflow needed. Keep `fern check` on PRs and use `pull-request` mode if humans must review generated changes. Early access: confirm availability on https://buildwithfern.com/learn/sdks/autorelease.

## Keeping generators current

Scheduled job that bumps generator versions and opens a PR:

```yaml
on:
  schedule: [{ cron: "0 6 * * 1" }]
  workflow_dispatch:
jobs:
  bump:
    runs-on: ubuntu-latest
    permissions: { contents: write, pull-requests: write }
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20 }
      - uses: fern-api/setup-fern-cli@v1
      - run: fern upgrade --yes && fern generator upgrade && fern check
      - uses: peter-evans/create-pull-request@v6
        with:
          branch: fern/upgrade
          title: "chore: upgrade Fern CLI and generators"
          commit-message: "chore: upgrade Fern CLI and generators"
```

Review the generator changelogs in the PR before merging; majors (`--include-major`) can rename `config` keys.
