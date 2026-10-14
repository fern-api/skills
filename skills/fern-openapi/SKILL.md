---
name: fern-openapi
description: >-
  Author and tune OpenAPI specs for Fern SDK and docs generation: operationId
  and schema hygiene, x-fern-* extensions (sdk-group-name, sdk-method-name,
  pagination, examples, availability, audiences, global headers, server
  variables, streaming, webhooks), overlays vs overrides, auth and servers,
  spec sync with fern api update, fern check. Use when editing an OpenAPI or
  AsyncAPI file that Fern consumes, or when generated SDK names/behaviour need
  to change at the spec level. Routes to a detailed reference per task.
---

# Fern OpenAPI

Use this skill when the task is about the API definition Fern reads: making an OpenAPI/AsyncAPI spec produce good SDK method names, groups, types, pagination, errors and docs; adding `x-fern-*` extensions; deciding between editing the spec, an overlay, or an overrides file; keeping the spec in sync with its source. For `generators.yml`/publishing use `fern-sdks`; for CI use `fern-ci`; for docs pages use `fern-docs`.

## Fern resources

- OpenAPI docs: https://buildwithfern.com/learn/api-definitions/openapi/overview — append `.md` to any docs URL for Markdown.
- Extensions index: https://buildwithfern.com/learn/api-definitions/openapi/extensions/overview (pages: `method-names`, `pagination`, `request-response-examples`, `availability`, `audiences`, `global-headers`, `global-parameters`, `sdk-variables`, `sdk-namespaces`, `schema-names`, `parameter-names`, `property-names`, `enum-descriptions-and-names`, `ignoring-elements`, `retry-behavior`, `idempotency`, `api-version`, `base-path`, `default-values`, `server-names-and-url-templating`, `tag-display-names`, `discriminator-context`, `api-explorer-control`).
- Endpoint types: https://buildwithfern.com/learn/api-definitions/openapi/endpoints/sse (streaming) · https://buildwithfern.com/learn/api-definitions/openapi/endpoints/webhooks · https://buildwithfern.com/learn/api-definitions/openapi/endpoints/multipart
- Overlays: https://buildwithfern.com/learn/api-definitions/openapi/overlays · Overrides: https://buildwithfern.com/learn/api-definitions/openapi/overrides
- Auth: https://buildwithfern.com/learn/api-definitions/openapi/authentication · Servers: https://buildwithfern.com/learn/api-definitions/openapi/servers
- Sync from a source: https://buildwithfern.com/learn/api-definitions/openapi/sync-your-open-api-specification
- llms.txt: https://buildwithfern.com/learn/llms.txt · MCP: `claude mcp add --transport http fern https://buildwithfern.com/learn/_mcp/server`

## Routing

| Task | Reference |
| --- | --- |
| Spec hygiene that drives SDK quality: `operationId`, `tags`, `$ref` schemas, nullable/optional, examples, error responses; `fern check`; `fern init`; `fern api update` / sync action; OpenAPI vs Fern Definition | `references/spec-hygiene.md` |
| Any `x-fern-*` extension: method/group names, request names, ignore, pagination, examples, availability, audiences, type/property/parameter names, enums, SDK variables, global headers, retries, idempotency, versioning, streaming, webhooks | `references/extensions.md` |
| Where to put Fern changes when you can't edit the source spec: overlays (OpenAPI Overlay 1.0) vs Fern overrides, `fern write-overrides` | `references/overlays-and-overrides.md` |
| Auth (`securitySchemes`, `security`, `x-fern-bearer`/`x-fern-header`/`x-fern-basic`, OAuth) and servers/environments (`servers`, `x-fern-server-name`, server `variables`, multi-host) | `references/auth-and-servers.md` |

## Working rules

1. Locate the spec from `fern/generators.yml` (`api.specs[].openapi`, plus `overlays`/`overrides` entries). Edit the file that owns the concern: a vendored spec you control → edit it; a spec pulled from `origin` → overlay (preferred) or overrides, never the generated copy.
2. Run `fern check` after every change. It validates the spec and reports Fern-specific problems (unknown extensions, bad `$ref`, missing `operationId` warnings with `--warnings`).
3. SDK naming comes from the spec: `operationId` → method name (or `x-fern-sdk-method-name`), `tags` → client group (or `x-fern-sdk-group-name`), schema key → type name. Fix names here rather than by hand in generated SDKs.
4. Only use extensions documented on the Fern extension pages. Fern ignores unknown `x-` keys silently, so a typo (`x-fern-sdk-method`) does nothing and gives no error.
5. Keep a single source of truth for shared shapes: define schemas under `components.schemas` and `$ref` them. Inline object schemas become auto-named types (`UsersListResponseItem`) unless you add `x-fern-type-name`.
6. Preview the effect with `fern generate --group <g> --preview` (or `--local --preview`) and inspect the generated client before publishing; see `fern-sdks`.
7. Don't put credentials or private hosts into a public spec; environments and auth env-vars can live in `generators.yml` instead.

## Quick reference

```yaml
openapi: 3.1.0
info: { title: Acme API, version: "1.0" }
servers:
  - url: https://api.acme.com
    x-fern-server-name: Production
  - url: https://api.staging.acme.com
    x-fern-server-name: Staging
security:
  - ApiKey: []
components:
  securitySchemes:
    ApiKey:
      type: apiKey
      in: header
      name: X-API-Key
      x-fern-header: { name: apiKey, env: ACME_API_KEY }
  schemas:
    User: { type: object, required: [id], properties: { id: { type: string }, email: { type: string } } }
paths:
  /users:
    get:
      operationId: listUsers            # → client.users.list(...)
      tags: [Users]
      x-fern-sdk-group-name: users
      x-fern-sdk-method-name: list
      x-fern-pagination:
        cursor: $request.cursor
        next_cursor: $response.next_cursor
        results: $response.data
      parameters:
        - { name: cursor, in: query, schema: { type: string } }
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema:
                type: object
                properties:
                  data: { type: array, items: { $ref: "#/components/schemas/User" } }
                  next_cursor: { type: [string, "null"] }   # 3.0: type: string, nullable: true
```

Commands:

```bash
fern check --warnings                 # validate spec + Fern config
fern init --openapi ./openapi.yml     # bootstrap a fern/ folder for an existing spec
fern api update                       # re-download specs that declare `origin`
fern write-overrides                  # scaffold overrides.yml with every operation
fern write-definition                 # see the Fern Definition Fern derived (debug naming/types)
fern generate --group <g> --preview   # check the SDK that results
```
