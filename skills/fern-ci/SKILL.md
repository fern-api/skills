---
name: fern-ci
description: >-
  Set up CI/CD for Fern projects: GitHub Actions (or any CI) that run fern
  check on pull requests, publish docs previews and production docs, generate
  and release SDKs with pinned CLI/generator versions, keep OpenAPI specs in
  sync with fern-api/sync-openapi, and handle FERN_TOKEN and registry secrets
  safely. Use when adding or debugging a workflow that runs the fern CLI.
  Routes to a detailed reference per task.
---

# Fern CI

Use this skill when wiring the Fern CLI into automation: validating on PRs, docs preview links and publishing, SDK generation/release, spec sync, and the failures that only show up on a CI runner. Configuration of `generators.yml` itself is in `fern-sdks`; spec edits in `fern-openapi`; docs content in `fern-docs`.

## Fern resources

- CLI reference: https://buildwithfern.com/learn/cli-api-reference/cli-reference/overview (append `.md` to any docs URL for Markdown)
- Publish docs (with workflow examples): https://buildwithfern.com/learn/docs/preview-publish/publishing-your-docs · Preview links: https://buildwithfern.com/learn/docs/preview-publish/preview-changes-locally
- Sync OpenAPI action: https://buildwithfern.com/learn/api-definitions/openapi/sync-your-open-api-specification · https://github.com/fern-api/sync-openapi
- SDK publishing per language: `https://buildwithfern.com/learn/sdks/generators/<lang>/publishing`
- Reference workflows: https://github.com/fern-api/docs/tree/main/.github/workflows
- llms.txt: https://buildwithfern.com/learn/llms.txt · MCP: `claude mcp add --transport http fern https://buildwithfern.com/learn/_mcp/server`

## Routing

| Task | Reference |
| --- | --- |
| Install the CLI in CI, pin versions, authenticate with `FERN_TOKEN`, run `fern check` on every PR, non-interactive flags | `references/validate-and-auth.md` |
| Docs: preview link per PR (stable `--id`), publish to staging/production on merge, multiple instances | `references/docs-workflows.md` |
| SDKs: generate on demand or on tag with `--version`, `pull-request` vs `release` mode, publishing from the generated SDK repo, local/self-hosted generation with Docker and `--package`, autorelease | `references/sdk-workflows.md` |
| Keep the spec current: `fern-api/sync-openapi`, `fern api update`, token permissions | `references/spec-sync.md` |
| Runner-only failures: login, Docker, proxies/registries, permissions, versions | `references/troubleshooting.md` |

## Working rules

1. Start from what exists: check `.github/workflows/` for a Fern workflow before adding one, and look at `fern/fern.config.json` for the CLI version the repo expects.
2. Install the CLI with `npm install -g fern-api` or `uses: fern-api/setup-fern-cli@v1` (needs a Node setup step first). The installed CLI re-execs the version pinned in `fern.config.json`, so the pin, not the install, decides the version. Don't `npx fern-api@latest` in CI.
3. Secrets come from the CI secret store (`${{ secrets.FERN_TOKEN }}`), never from files in the repo. Generate `FERN_TOKEN` with `fern token` (needs `fern login` once, locally). Registry tokens live in the **SDK** repo's secrets, not the config repo.
4. Every PR runs `fern check`. Publishing runs only on `main` (or a tag / manual dispatch). Previews never publish.
5. Make commands non-interactive: `--force` / `--no-prompt` where a prompt could appear (overwriting a preview `--id`, confirming a release), `--yes` for `fern upgrade`.
6. Separate "generate" from "publish" for SDKs: Fern pushes code to the SDK repo (preferably `pull-request` mode); the SDK repo's own workflow publishes on tag/release. Don't put registry tokens in the config repo's workflow unless Fern is configured to publish directly (`output.token`).
7. Be explicit about working directory when `fern/` isn't at the repo root (`working-directory:`).
8. Report the result where humans look: preview URL as a PR comment, release URL in the job summary.

## Quick reference

Minimal PR validation workflow:

```yaml
name: Fern check
on:
  pull_request:
jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20 }
      - uses: fern-api/setup-fern-cli@v1   # or: run: npm install -g fern-api
      - run: fern check
```

Add `env: { FERN_TOKEN: ${{ secrets.FERN_TOKEN }} }` to any step that runs `fern generate` (docs or SDKs, remote or `--local`). `fern check` does not need a token.
