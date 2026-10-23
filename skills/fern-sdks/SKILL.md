---
name: fern-sdks
description: >-
  Configure and generate Fern SDKs: generators.yml (groups, generators,
  outputs, GitHub publishing modes), fern.config.json, auth schemes,
  environments, .fernignore and Replay for custom code, fern generate /
  generator upgrade / --local / --preview / --package, and common generation
  failures. Use when working in a Fern SDK repo (a fern/ directory with
  generators.yml). Routes to a detailed reference per task.
---

# Fern SDKs

Use this skill when a task involves generating or publishing SDKs with Fern: editing `fern/generators.yml` or `fern/fern.config.json`, adding a language, changing where an SDK is published, keeping hand-written code across regenerations, or debugging a `fern generate` run. For editing the API spec itself (OpenAPI, `x-fern-*` extensions, overlays) use the `fern-spec` skill; for CI workflows use `fern-workflows`; for docs sites use `fern-docs`.

## Fern resources

Look things up instead of guessing. Fern options are strict: an unknown key in `generators.yml` or a generator's `config` fails validation.

- Docs: https://buildwithfern.com/learn/sdks — append `.md` to any docs URL to get Markdown (for example `https://buildwithfern.com/learn/sdks/reference/generators-yml-reference.md`).
- `generators.yml` reference: https://buildwithfern.com/learn/sdks/reference/generators-yml-reference
- Per-language pages (configuration, publishing, custom code): `https://buildwithfern.com/learn/sdks/generators/<typescript|python|java|go|csharp|php|ruby|swift|rust>/...`
- CLI reference: https://buildwithfern.com/learn/cli-api-reference/cli-reference/sdk-commands
- llms.txt: https://buildwithfern.com/learn/llms.txt
- Fern MCP server (if the agent supports MCP): `claude mcp add --transport http fern https://buildwithfern.com/learn/_mcp/server`
- Schema for editor validation: add `# yaml-language-server: $schema=https://schema.buildwithfern.dev/generators-yml.json` to the top of `generators.yml`.

## Routing

Load only the reference you need.

| Task | Reference |
| --- | --- |
| Understand the `fern/` folder, `fern.config.json`, `generators.yml` top-level keys, groups, aliases, generator entries, generator `config` | `references/generators-yml.md` |
| Publish to npm / PyPI / Maven / NuGet / RubyGems / local filesystem; GitHub `release` vs `pull-request` vs `push`; autorelease; versioning | `references/publishing.md` |
| Auth schemes (bearer, basic, header, OAuth), global headers, environments / base URLs, SDK variables | `references/auth-and-environments.md` |
| Keep hand-written code: `.fernignore` vs Replay, adding custom methods, typed errors | `references/custom-code.md` |
| Run `fern generate` (groups, `--version`, `--preview`, `--local`, `--package`, `--force`), `fern generator upgrade`, `fern check`, debug failures | `references/cli-and-troubleshooting.md` |

## Working rules

1. Find the project first. Look for `fern/fern.config.json` and `fern/generators.yml` (or `fern/apis/<name>/generators.yml` for multi-API repos). Run `fern check` before and after edits.
2. Read the existing `generators.yml` and reuse its conventions (group names, output style, existing `config` keys) before adding new ones. Prefer changing an existing group over adding a parallel one.
3. Never invent configuration. Confirm every `config` key against the language page or the `generators.yml` reference; unknown keys fail generation. If a key only exists in a newer generator, bump `version` in the same change.
4. Generator versions are pinned per generator (`version: 3.x.y`) and the CLI version is pinned in `fern.config.json`. Upgrade with `fern generator upgrade` / `fern upgrade` rather than editing numbers by hand, unless the user asks for a specific version.
5. Validate without publishing: `fern generate --group <g> --preview` (remote, needs `fern login` or `FERN_TOKEN`) or `fern generate --group <g> --local --preview` (Docker, no login). Output lands in `fern/.preview/<generator>/`.
6. Custom code goes in files listed in `.fernignore` (whole-file) or is preserved by Replay (line-level, `pull-request` mode). Don't edit generated files in a `release`/`push` repo and expect the edit to survive.
7. Don't put real tokens in `generators.yml`; use `${ENV_VAR}` references and GitHub/CI secrets.
8. Changes to the SDK's public surface (method names, request wrappers, error types) are usually spec-side (`operationId`, `x-fern-sdk-method-name`, `x-fern-request-name`, `error-responses`) — hand off to `fern-spec` for those.

## Quick reference

Minimal `fern/generators.yml` for an OpenAPI-based TypeScript SDK published to npm from a GitHub repo:

```yaml
# yaml-language-server: $schema=https://schema.buildwithfern.dev/generators-yml.json
api:
  specs:
    - openapi: ./openapi.yml
default-group: ts-sdk
groups:
  ts-sdk:
    generators:
      - name: fernapi/fern-typescript-sdk
        version: 3.106.1
        output:
          location: npm
          package-name: "@acme/sdk"
          token: ${NPM_TOKEN}
        github:
          repository: acme/acme-typescript
          mode: pull-request
        config:
          namespaceExport: Acme
```

Common commands:

```bash
fern check                                   # validate api + generators.yml
fern generate --group ts-sdk --preview       # dry run into fern/.preview/
fern generate --group ts-sdk --version 1.2.0 # publish / open PR with that version
fern generate --group ts-sdk --local         # run generator in Docker, no Fern login
fern generator upgrade                       # bump generator versions (minor/patch)
fern generator upgrade --include-major       # also take majors (runs migrations)
fern upgrade                                 # bump CLI version in fern.config.json
```

Generator image names: `fernapi/fern-typescript-sdk`, `fernapi/fern-python-sdk`, `fernapi/fern-java-sdk`, `fernapi/fern-go-sdk`, `fernapi/fern-csharp-sdk`, `fernapi/fern-php-sdk`, `fernapi/fern-ruby-sdk`, `fernapi/fern-swift-sdk`, `fernapi/fern-rust-sdk`, `fernapi/fern-postman`, `fernapi/fern-openapi`. Current versions: `https://buildwithfern.com/learn/sdks/generators/<lang>/changelog`.
