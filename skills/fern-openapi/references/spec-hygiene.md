# Spec hygiene for good SDKs and docs

Fern reads the spec as-is; most SDK quality problems trace back to the spec. Check these before reaching for extensions.

## Checklist

| Spec element | Effect in Fern | Rule |
| --- | --- | --- |
| `operationId` | Method name (`listUsers` → `client.users.list()` when grouped, else `client.listUsers()`) | Required on every operation (and webhook). Verb-first camelCase. Override with `x-fern-sdk-method-name`. |
| `tags` | Client group / docs section | One primary tag per operation; tag name becomes `client.<tag>`. Override with `x-fern-sdk-group-name`. |
| `components.schemas` + `$ref` | Named types (`User`) | Define every reusable object once and `$ref` it. Inline objects get auto-generated names unless `x-fern-type-name` is set. |
| `required` | Optional vs required fields/params | List required properties explicitly; everything else is optional. |
| `nullable: true` / `type: [string, "null"]` | `Optional`/nullable types | Keep nullable distinct from optional; `settings.respect-nullable-schemas` / `coerce-optional-schemas-to-nullable` tune the mapping. |
| `enum` | Enum types | Use `x-fern-enum` to attach names/descriptions/deprecation to values. |
| `oneOf` + `discriminator` | Discriminated unions | Prefer discriminated unions over bare `oneOf`; for SSE streams whose discriminator is the `event` field set `x-fern-discriminator-context: protocol`. |
| `responses` with schemas for 4xx/5xx | Typed error classes | Give each error status a schema (or use `settings.error-responses` in `generators.yml`). Without schemas, SDKs throw a generic `ApiError`. |
| `examples` / `x-fern-examples` | README, reference.md, docs code samples, wire tests | Add at least one example per important endpoint; run `fern api enrich` to inline Fern-generated examples. |
| `servers` | Environments / base URL | At least one; name them with `x-fern-server-name`. |
| `securitySchemes` + `security` | Client auth options | See auth-and-servers.md. |
| `summary` / `description` | Docstrings and docs | Write both; `description` supports Markdown. |

Run `fern check --warnings` to see what Fern complains about; warnings often point at missing `operationId`s or unreferenced schemas.

## Request bodies and parameters

- Fern inlines query/path/header parameters and a JSON body into one request object per method. Set `x-fern-request-name` if the auto name (`{Method}{Group}Request` or `{Group}{Method}Request` depending on `idiomatic-request-names`) is wrong.
- `application/x-www-form-urlencoded` and `multipart/form-data` bodies are supported; file fields use `type: string, format: binary`. Multipart detail: https://buildwithfern.com/learn/api-definitions/openapi/endpoints/multipart
- Streaming responses: `text/event-stream` is auto-detected; for newline-delimited JSON add `x-fern-streaming: true`, or `x-fern-streaming: { format: sse }` to force SSE. https://buildwithfern.com/learn/api-definitions/openapi/endpoints/sse
- Webhooks: OpenAPI 3.1 top-level `webhooks:` (each with an `operationId`), or `x-fern-webhook: true` on a `paths` operation in OpenAPI 3.0. https://buildwithfern.com/learn/api-definitions/openapi/endpoints/webhooks

## Bootstrapping and keeping the spec in sync

```bash
fern init --openapi ./openapi.yml        # or a URL; creates fern/fern.config.json, generators.yml, openapi/openapi.yml
fern init --openapi https://api.acme.com/openapi.json
```

If the spec is produced elsewhere (server framework, API gateway), do not hand-edit the copy in `fern/`. Declare its source and refresh it:

```yaml
# generators.yml
api:
  specs:
    - openapi: ./openapi/openapi.json
      origin: https://api.acme.com/openapi.json
      overlays: ./overlays/sdk.yml     # Fern customisations live here, not in the downloaded file
```

```bash
fern api update        # re-download every spec with an origin (the target file must already exist)
```

Automate it with the `fern-api/sync-openapi` GitHub Action (see the `fern-ci` skill): it runs `fern api update` on a schedule and opens a PR with the diff. Fern customisations survive because they live in the overlay/overrides file.

## OpenAPI vs Fern Definition

Stay on OpenAPI when the spec already exists or is generated from code. Use a Fern Definition (`fern/definition/*.yml`) when writing an API from scratch and you want first-class unions, inheritance, and simpler syntax. `fern write-definition` prints the Fern Definition that Fern derived from the OpenAPI spec, which is the fastest way to see how Fern interpreted names and types. Docs: https://buildwithfern.com/learn/api-definitions/ferndef/overview

## Validation failures that look like Fern bugs but are spec problems

- `$ref` to a schema that doesn't exist or is defined under a different key.
- Two operations resolving to the same `operationId` (or same group+method after extensions).
- A path parameter in the URL template with no matching `parameters` entry.
- `type: object` with `additionalProperties` and `properties` both set (becomes a map, not an object).
- Response declared with no `content` but the SDK is expected to return a body.
