# Publishing SDKs

Fern generates code remotely by default (`fern generate`, requires `fern login` or `FERN_TOKEN`) and then does one or both of: push source to GitHub, publish a package to a registry. Both are configured per generator entry.

## GitHub (`github`)

```yaml
github:
  repository: acme/acme-typescript   # Fern GitHub App must be installed on this repo
  mode: pull-request                 # release (default) | pull-request | push
  branch: main                       # optional in every mode: release commits/tags here, pull-request/push target it
  license: MIT                       # or { custom: ./LICENSE }
  reviewers:
    teams: [{ name: sdk-owners }]    # entries are objects with `name`
    users: [{ name: octocat }]
  workflows: false                   # skip generating .github/workflows (token lacks `workflows` scope)
```

| Mode | Behaviour | Use when |
| --- | --- | --- |
| `release` (default) | Commits straight to the default branch and creates a git tag/GitHub release for `--version`. The SDK repo's generated CI publishes on that tag. | Fully automated, no review of generated code. |
| `pull-request` | Opens a PR with the regenerated code. Required for Replay (line-level custom-code preservation). Nothing is published until the PR is merged and a release/tag is cut. | Recommended for anything with custom code or human review. |
| `push` | Pushes to `branch` without tagging. | You run your own release process. |

Self-hosted Git: replace `repository` with `uri: https://github.acme.internal/org/repo` and `token: ${GIT_TOKEN}`. GitLab/Bitbucket are not supported as `github` targets; use `location: local-file-system` and commit from your own pipeline (see `fern-ci`).

## Registries (`output`)

Only one `output` per generator entry. Tokens are `${ENV_VAR}` references resolved from the environment running `fern generate` (locally) or from secrets Fern has for the org (remote). Never commit raw tokens.

```yaml
# npm
output:
  location: npm
  package-name: "@acme/sdk"
  token: ${NPM_TOKEN}
  # url: https://npm.acme.internal   # private registry
# PyPI
output:
  location: pypi
  package-name: acme
  token: ${PYPI_TOKEN}
# Maven (Sonatype / Central)
output:
  location: maven
  coordinate: com.acme:acme-java
  username: ${MAVEN_USERNAME}
  password: ${MAVEN_PASSWORD}
  signature:
    keyId: ${MAVEN_KEY_ID}
    password: ${MAVEN_KEY_PASSWORD}
    secretKey: ${MAVEN_SECRET_KEY}
# NuGet
output:
  location: nuget
  package-name: Acme.Sdk
  api-key: ${NUGET_API_KEY}
# RubyGems — needs a github: block too; the gem is pushed by the GitHub Actions workflow in that repo, not by Fern directly
output:
  location: rubygems
  package-name: acme
  api-key: ${RUBYGEMS_API_KEY}
github:
  repository: acme/acme-ruby
# Postman
output:
  location: postman
  api-key: ${POSTMAN_API_KEY}
  workspace-id: ${POSTMAN_WORKSPACE_ID}
# Local only (no publish). Also the only mode that works with --package
output:
  location: local-file-system
  path: ../sdks/typescript
```

Go has no registry: publishing is pushing a tagged commit to the GitHub repo (`github` + `release`/`pull-request`), and `config.module.path` must match the repo path.

In practice, registry publishing is usually done by the generated SDK repo's own GitHub workflow (`.github/workflows/ci.yml`, publish job triggered by a tag/release and reading `NPM_TOKEN`/`PYPI_TOKEN`/… from that repo's secrets). The `output` block tells Fern what package name and registry to wire into that workflow and the package manifest. Check the language's "Publishing to <registry>" page for the exact secret names and for trusted publishing (npm OIDC) setup.

## Versioning

- `fern generate --group <g> --version 1.4.0` sets the package version, tag, and release name. Format is `X.Y.Z` or `X.Y.Z-rc<N>`; other prerelease forms are rejected.
- Without `--version`, Fern pushes code without creating a release (useful for `push`/`pull-request` mode).
- To find what the next version should be, read the SDK repo's latest tag or run `fern diff` if the repo uses it.
- Document the version strategy in the SDK repo README if it differs from the API version.

## Autorelease (early access)

```yaml
autorelease: true          # top level: all generators
groups:
  sdks:
    generators:
      - name: fernapi/fern-typescript-sdk
        version: 3.106.1
        autorelease: true  # or per generator
```

When the spec in the Fern config repo changes, Fern regenerates and releases. Requires the Fern GitHub App on both repos. Pair with `mode: pull-request` if humans should review generated changes before release.

## Local artifacts without a registry

`fern generate --group <g> --local --package` builds installable artifacts (`.tgz`, wheel, `.jar`, `.nupkg`, `.gem`, …) into `fern-dist/` for `local-file-system` outputs — useful for internal registries or air-gapped environments. `--package-mode docker` builds inside Docker when the host lacks the toolchain; `--package-only` deletes everything but `fern-dist/` (and `.fernignore`d paths) after packaging, so only the artifact remains. `github` outputs are ignored by `--package`.

## Checklist before a first release

1. GitHub repo exists, Fern GitHub App installed, registry secrets added to the SDK repo.
2. `fern check` passes and a `--preview` build was inspected.
3. Package name is available on the registry.
4. `mode` chosen deliberately (`pull-request` if anyone will hand-edit the SDK).
5. `fern generate --group <g> --version 0.1.0` (or run the CI workflow described in `fern-ci`).
