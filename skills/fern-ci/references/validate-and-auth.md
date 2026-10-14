# Installing, authenticating, validating

## Installing the CLI on a runner

```yaml
- uses: actions/setup-node@v4
  with: { node-version: 20 }
- uses: fern-api/setup-fern-cli@v1        # inputs: version (default latest)
# equivalent: - run: npm install -g fern-api
```

The `fern-api` package is a launcher: on every run it reads `fern/fern.config.json` and downloads/executes that exact CLI version. So:

- Pin the project version in `fern.config.json` (`"version": "0.x.y"`), commit it, and let `fern upgrade` move it. The install step can stay `latest`.
- Set `FERN_NO_VERSION_REDIRECTION=true` only when you deliberately want the installed version (e.g. testing a new CLI before updating the pin).
- `fern upgrade --yes` is non-interactive; use it in a scheduled "bump Fern" job that opens a PR.

Other CI providers (GitLab CI, CircleCI, Azure Pipelines, Buildkite): any image with Node 18+ works; `npm install -g fern-api && fern check`. Docker is only needed for `fern generate --local`.

## Authentication

| Command | Needs `FERN_TOKEN`? |
| --- | --- |
| `fern check` | No |
| `fern generate --docs` (preview or publish) | Yes |
| `fern generate` (remote SDK generation, default) | Yes |
| `fern generate --local` | Yes (organization verification), plus registry/GitHub tokens if it publishes |
| `fern api update` | No (needs network access to `origin`) |
| `fern generator upgrade`, `fern upgrade` | No |

Create the token once locally: `fern login` then `fern token` (prints a long-lived token scoped to your Fern organization). Store it as a CI secret named `FERN_TOKEN` and pass it as an env var to the steps that need it:

```yaml
- run: fern generate --docs
  env:
    FERN_TOKEN: ${{ secrets.FERN_TOKEN }}
```

Secrets are not available to workflows triggered from forks or by Dependabot. Guard with a check so the job is skipped instead of failing:

```yaml
- id: token
  env: { FERN_TOKEN: ${{ secrets.FERN_TOKEN }} }
  run: echo "present=$([ -n "$FERN_TOKEN" ] && echo true || echo false)" >> "$GITHUB_OUTPUT"
- if: steps.token.outputs.present == 'true'
  run: fern generate --docs --preview
  env: { FERN_TOKEN: ${{ secrets.FERN_TOKEN }} }
```

Never echo the token, never write it to a file in the workspace, and don't pass it on the command line. Rotate it from the Fern Dashboard (https://dashboard.buildwithfern.com) if it leaks.

## Validating on every PR

```yaml
name: Fern check
on:
  pull_request:
  push:
    branches: [main]
jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20 }
      - uses: fern-api/setup-fern-cli@v1
      - run: fern check --warnings
        working-directory: ./api          # only if fern/ is not at the repo root
```

`fern check` validates every API definition, `generators.yml`, and (if present) `docs.yml`. Add `--warnings` to surface non-fatal issues in logs; treat it as a hard gate for new warnings if the team wants (parse the `Found N errors and M warnings` line).

For large repos, trigger only on relevant paths:

```yaml
on:
  pull_request:
    paths: ["fern/**", ".github/workflows/fern-*.yml"]
```

## Non-interactive flags

| Situation | Flag |
| --- | --- |
| Overwriting an existing preview `--id` | `--no-prompt` (or `--force`); GitHub Actions is detected automatically, other CIs are not |
| `fern generate` confirmations (e.g. releasing) | `--force` |
| `fern upgrade` | `--yes` |
| Verbose output for a failing job | `--log-level debug` |
