# Project layout and `generators.yml`

## The `fern/` folder

```
fern/
  fern.config.json      # { "organization": "acme", "version": "<fern CLI version>" }
  generators.yml        # API specs + SDK generators (this file)
  openapi/openapi.yml   # or asyncapi/, definition/ (Fern Definition), openrpc/
  .fernignore           # optional; files Fern must not overwrite (see custom-code.md)
  .preview/             # output of `fern generate --preview` (gitignored)
```

Multi-API repos use `fern/apis/<api-name>/generators.yml` (one API per folder) and `fern generate --api <api-name>`. One `generators.yml` can also list several specs, each with a `namespace`, to produce a single combined SDK.

`fern.config.json` pins the CLI. `"version": "*"` means "whatever is installed" and is only for local experiments; commit a concrete version. Change it with `fern upgrade` (runs config migrations) rather than editing by hand.

## Top-level keys

| Key | Purpose |
| --- | --- |
| `api` | `specs` (OpenAPI / AsyncAPI / OpenRPC / Fern Definition / proto), plus `auth`, `headers`, `environments`, `settings` |
| `auth-schemes` | Named auth schemes referenced from `api.auth` (see auth-and-environments.md) |
| `default-group` | Group (or alias) used when `fern generate` is run without `--group` |
| `groups` | Named sets of generators; `fern generate --group <name>` |
| `aliases` | `alias-name: [group-a, group-b]` — run several groups with one `--group` |
| `whitelabel` | Hide Fern branding in generated READMEs/packages |
| `metadata` | Package metadata shared by all SDKs (`description`, `authors`, `license`, `homepage`, `tags`) |
| `readme` | Generated README controls (`defaultEndpoint`, `bannerLink`, `apiReferenceLink`, `features`) |
| `autorelease` | `true` to let Fern regenerate and release on spec changes (early access; see publishing.md) |
| `replay` | `enabled: false` to disable Replay (see custom-code.md) |

### `api.specs`

```yaml
api:
  specs:
    - openapi: ./openapi/openapi.yml
      origin: https://api.acme.com/openapi.json   # enables `fern api update`
      overlays: ./overlays/sdk.yml                # OpenAPI Overlay (preferred)
      overrides: ./overrides.yml                  # Fern deep-merge overrides (string or list)
      namespace: billing                          # only when combining several specs
      settings:
        title-as-schema-name: true
        inline-path-parameters: false
        idiomatic-request-names: true             # ListUsersRequest instead of UsersListRequest
        respect-nullable-schemas: false
        ignore-tags: [internal]
        filter:
          endpoints: ["GET /users", "POST /users"]
        error-responses:                          # typed errors from a shared error schema
          schema: ./schemas/Error.yml
          apply-to: untyped                       # or all
          ensure:
            - status-code: 429
              methods: [GET, POST]
    - asyncapi: ./asyncapi.yml
```

Overlays vs overrides vs editing the spec is covered in the `fern-openapi` skill; from the SDK side only the file reference above matters.

### Groups and generator entries

```yaml
default-group: sdks
aliases:
  all: [sdks, postman]
groups:
  sdks:
    audiences: [public]              # only endpoints/schemas tagged x-fern-audiences: [public]
    reviewers:                       # default reviewers on generated PRs (pull-request mode)
      teams: [sdk-team]
    generators:
      - name: fernapi/fern-python-sdk
        version: 5.41.2
        config:
          client_class_name: Acme
          pydantic_config:
            skip_validation: true
        output:
          location: pypi
          package-name: acme-sdk
          token: ${PYPI_TOKEN}
        github:
          repository: acme/acme-python
          mode: pull-request
        metadata:                    # per-SDK overrides of top-level metadata
          package-description: Acme Python SDK
        api:                         # per-SDK auth override
          auth: ApiKey
        snippets:
          path: ./snippets/python.json   # also enables reference.md generation
  postman:
    generators:
      - name: fernapi/fern-postman
        version: 0.6.1
        output:
          location: postman
          api-key: ${POSTMAN_API_KEY}
          workspace-id: ${POSTMAN_WORKSPACE_ID}
```

Generator entry fields:

- `name` — Docker image, e.g. `fernapi/fern-typescript-sdk`. Use `image: { name, registry }` instead when pulling from a private registry.
- `version` — pinned generator version. Check the language changelog for current versions. `fern generator upgrade` bumps minor/patch; `--include-major` bumps majors and runs the generator's migrations on `config`.
- `config` — language-specific. Every key must exist for that generator version; look it up at `https://buildwithfern.com/learn/sdks/generators/<lang>/configuration`. Common ones: TypeScript `namespaceExport`, `outputSourceFiles`, `allowCustomFetcher`, `inlineFileProperties`; Python `client_class_name`, `pydantic_config`, `follow_redirects_by_default`; Java `client-class-name`, `package-prefix`, `enable-forward-compatible-enums`; Go `module.path`, `packageName`; C# `namespace`, `client-class-name`; PHP `namespace`; Ruby `clientClassName`; Rust `crateName`.
- `smart-casing: true` — better casing for acronyms (e.g. `apiUrl` → `APIUrl` where idiomatic).
- `output` — where to publish (see publishing.md). Omit `output` to only push code to GitHub.
- `github` — repo to push the SDK source to and the mode (see publishing.md).
- `snippets.path` — writes a `snippet.json` with per-endpoint snippets; required for `reference.md` in the SDK and for docs code samples.

## Checklist for adding a new language

1. Add a generator entry to an existing group (or a new group if release cadence differs) with `name`, `version` (latest from the changelog), `output`, `github`, and the minimum `config` the language needs (client class name, package/module path).
2. Create the GitHub repo and install the Fern GitHub App on it (https://github.com/apps/fern-api); otherwise `fern generate` cannot push.
3. Add the registry token as a secret and reference it as `${TOKEN_NAME}`.
4. `fern check`, then `fern generate --group <g> --preview` and inspect `fern/.preview/`.
5. `fern generate --group <g> --version 0.1.0` for the first release.
