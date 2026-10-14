# Docs workflows

Reference implementations: https://github.com/fern-api/docs/tree/main/.github/workflows (`preview-docs.yml`, `publish-docs.yml`). Docs: https://buildwithfern.com/learn/docs/preview-publish/publishing-your-docs

## Preview link on every PR

`fern generate --docs --preview` publishes to a throwaway URL (not indexed, never expires). `--id <name>` makes the URL stable so rerunning the job updates the same preview; use the branch name.

```yaml
name: Preview docs
on:
  pull_request:
    types: [opened, synchronize, ready_for_review]
permissions:
  contents: read
  pull-requests: write          # to comment the URL
jobs:
  preview:
    if: github.event.pull_request.head.repo.full_name == github.repository   # forks have no secrets
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20 }
      - uses: fern-api/setup-fern-cli@v1
      - id: preview
        env:
          FERN_TOKEN: ${{ secrets.FERN_TOKEN }}
          HEAD_REF: ${{ github.head_ref }}
        run: |
          OUTPUT=$(fern generate --docs --preview --id "$HEAD_REF" --no-prompt 2>&1) || { echo "$OUTPUT"; exit 1; }
          echo "$OUTPUT"
          URL=$(echo "$OUTPUT" | grep -oP 'Published docs to \K\S+')
          echo "url=$URL" >> "$GITHUB_OUTPUT"
      - uses: marocchino/sticky-pull-request-comment@v2
        with:
          header: fern-preview
          message: "Docs preview: ${{ steps.preview.outputs.url }}"
```

Notes:

- With several `instances` in `docs.yml`, add `--instance <url>` so the preview is built for the right site.
- The preview URL is printed as `[docs]: Published docs to https://fern-preview-<id>.docs.buildwithfern.com/<base-path>`.
- Previews don't need `fern check` first; `fern generate --docs` runs validation itself and fails on errors.

## Publish on merge

```yaml
name: Publish docs
on:
  push:
    branches: [main]
  workflow_dispatch:
jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20 }
      - uses: fern-api/setup-fern-cli@v1
      - run: fern generate --docs
        env:
          FERN_TOKEN: ${{ secrets.FERN_TOKEN }}
```

Staging + production: declare both in `docs.yml` `instances`, publish staging on push to `main` and production from `workflow_dispatch` (or a tag) with `--instance <prod-url>`. The docs page above has both workflows.

## Docs that embed an API reference

If `docs.yml` references an OpenAPI spec, the docs workflow should run after the spec-sync workflow (or be triggered by it) so published docs match the current API. For SDK code snippets in docs, the SDK generators need `snippets.path` configured; nothing extra is required in CI.

## Local build check without publishing

`fern docs dev` is interactive and not meant for CI. For a CI-side sanity check use `fern check` (validates `docs.yml`, pages, broken internal links) and rely on the preview for rendering.
