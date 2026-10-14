# Keeping the OpenAPI spec in sync

When the spec is produced by the server (framework export, gateway, another repo), don't hand-edit the copy in `fern/`. Point Fern at the source and let CI refresh it; keep Fern customisations in an overlay/overrides file so they survive updates (see `fern-openapi`).

```yaml
# fern/generators.yml
api:
  specs:
    - openapi: ./openapi/openapi.json
      origin: https://api.acme.com/openapi.json
      overlays: ./overlays/sdk.yml
```

## Scheduled sync with `fern-api/sync-openapi`

```yaml
name: Sync OpenAPI spec
on:
  workflow_dispatch:
  schedule: [{ cron: "0 3 * * *" }]
jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          token: ${{ secrets.OPENAPI_SYNC_TOKEN }}
      - uses: fern-api/sync-openapi@v2
        with:
          update_from_source: true           # runs `fern api update` in this repo
          token: ${{ secrets.OPENAPI_SYNC_TOKEN }}
          branch: update-api                 # default: fern/sync-openapi
          auto_merge: false                  # true = push to branch directly instead of opening a PR
```

`OPENAPI_SYNC_TOKEN` is a GitHub token (fine-grained PAT or GitHub App token) with **Contents: read/write** and **Pull requests: read/write** on this repo. The default `GITHUB_TOKEN` works only if "Allow GitHub Actions to create and approve pull requests" is enabled in the repo settings; PRs opened with it won't trigger other workflows, so prefer a PAT/App token.

Action inputs: `update_from_source`, `token`, `branch`, `auto_merge`, and for copying spec files between repos instead of using `origin`: `repository` (target `owner/repo`) + `sources` (array of `{from, to}` mappings). Source: https://github.com/fern-api/sync-openapi.

## Copying the spec from the server repo

Run the action in the **server** repo after the spec is built, pushing the file into the Fern config repo:

```yaml
- uses: fern-api/sync-openapi@v2
  with:
    repository: acme/fern-config
    token: ${{ secrets.FERN_CONFIG_TOKEN }}
    branch: update-api
    sources: |
      - from: build/openapi.json
        to: fern/openapi/openapi.json
```

## Without the action

```yaml
- uses: fern-api/setup-fern-cli@v1
- run: fern api update && fern check
- uses: peter-evans/create-pull-request@v6
  with: { branch: update-api, title: "chore: update OpenAPI spec" }
```

`fern api update` requires the target file to exist already and replaces it with the download from `origin`.

## Chaining

Spec sync PR merged → `fern check` passes on `main` → (optional) SDK release workflow or autorelease → docs publish. If the docs embed the API reference, run the docs publish workflow on `push` to `main` so it picks up the merged spec.
