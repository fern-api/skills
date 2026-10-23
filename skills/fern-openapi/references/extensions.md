# `x-fern-*` extensions

All extensions are optional and documented at https://buildwithfern.com/learn/api-definitions/openapi/extensions/overview. Unknown `x-` keys are ignored silently, so spell them exactly. They can be set in the spec, an overlay, or an overrides file.

## Naming

```yaml
paths:
  /users/{id}:
    get:
      operationId: getUser
      x-fern-sdk-group-name: users          # client.users.get(); use a list for nesting: [admin, users] → client.admin.users
      x-fern-sdk-method-name: get
      x-fern-request-name: GetUserRequest   # name of the generated request object
      parameters:
        - name: id
          in: path
          x-fern-parameter-name: userId     # SDK-side parameter name
components:
  schemas:
    user_record:
      x-fern-type-name: User                # SDK type name for this (or an inline) schema
      properties:
        created_ts:
          x-fern-property-name: createdAt   # SDK-side property name (wire name unchanged)
    Suit:
      type: string
      enum: [clubs, diamonds]
      x-fern-enum:
        clubs: { description: The clubs suit }
        diamonds: { description: Deprecated. Use hearts., deprecated: true }
tags:
  - name: users
    x-displayName: User management        # docs sidebar label
```

- `x-fern-sdk-namespace` on a schema in `components.schemas` (or on a shared error response with `settings.namespaced-errors: true`) places that one type in a namespace (`garden.Plant`). Namespacing a whole spec is done in `generators.yml` with the per-spec `namespace` key, not with this extension.
- `x-fern-base-path` (top level) prepends a path to every endpoint.
- `x-fern-default` sets a default value on a parameter or header (`x-fern-default: "2024-02-08"`).

## Hiding and scoping

```yaml
paths:
  /internal/metrics:
    get:
      x-fern-ignore: true                   # dropped from SDKs and docs; also valid on schemas, parameters, properties
  /users:
    get:
      x-fern-audiences: [public]            # kept only when generators.yml / docs.yml select that audience
      x-fern-availability: beta             # alpha | beta | preview | generally-available | deprecated | legacy
  /pets:
    put:
      x-fern-availability:
        status: deprecated
        message: Use PATCH /pet instead.
```

Audience filtering only applies when `groups.<g>.audiences` (SDKs) or `docs.yml` `audiences` selects something. Untagged elements are always included; tagged elements are dropped when their audience isn't selected.

## Pagination

```yaml
# offset
x-fern-pagination:
  offset: $request.page_number
  results: $response.results
# cursor
x-fern-pagination:
  cursor: $request.cursor
  next_cursor: $response.next
  results: $response.results
# next URL / next path
x-fern-pagination:
  next_uri: $response.next_page_url     # or next_path: $response.next_page_path
  results: $response.results
```

`$request.x` refers to a query/body parameter, `$response.x` to a response-body field (dot paths allowed, e.g. `$response.meta.next`). Generated SDKs expose an iterator/pager over `results`. Only endpoints carrying the extension are paginated.

## Examples

```yaml
paths:
  /users/{userId}:
    get:
      x-fern-examples:
        - name: Get user 1234
          path-parameters: { userId: user-1234 }
          query-parameters: { expand: profile }
          headers: { X-Request-Id: req_1 }
          response:
            body: { id: user-1234, name: Foo }
  /users:
    post:
      x-fern-examples:
        - name: Create user
          request: { name: Alice, email: alice@example.com }
          response:
            body: { id: user-5678, name: Alice, email: alice@example.com }
```

Examples feed README snippets, `reference.md`, docs code samples and generated wire tests. Schema-level `example`/`examples` are also honoured. `fern api enrich` writes Fern's auto-generated examples back into the spec so you can edit them.

## Headers, variables, versioning

```yaml
# document level
x-fern-global-headers:
  - header: X-Tenant-Id        # wire name
    name: tenantId             # SDK option name
    optional: true
    env: ACME_TENANT_ID
  - header: X-Partner-Id
    name: partnerId
    x-fern-default: acme   # don't reuse the x-fern-version header name here

x-fern-sdk-variables:          # path params lifted to the client constructor
  accountId:
    type: string
    description: Account the client operates on
paths:
  /accounts/{accountId}/users:
    get:
      parameters:
        - name: accountId
          in: path
          required: true
          schema: { type: string }
          x-fern-sdk-variable: accountId

x-fern-version:                # header-based API versioning exposed as a client option
  header: X-API-Version
  default: "2.0.0"
  values: ["1.0.0", "2.0.0"]
```

`x-fern-global-headers` entries are removed from each endpoint's parameters and become client-level options (same as `api.headers` in `generators.yml`, which takes precedence).

## Retries and idempotency

```yaml
paths:
  /payments:
    post:
      x-fern-retries: { disabled: true }       # per-endpoint; overrides user retry config
      x-fern-idempotent: true                  # pair with document-level x-fern-idempotency-headers
x-fern-idempotency-headers:
  - header: Idempotency-Key
    name: idempotencyKey
```

Pages: `retry-behavior`, `idempotency`.

## Servers

`x-fern-server-name` names a `servers[]` entry and, on an operation, assigns it to a host in a multi-host setup; server `variables` become client options (`server-names-and-url-templating`). Details in auth-and-servers.md.

## Docs-only

- `x-fern-explorer` controls the API Explorer/playground for an operation (`api-explorer-control`).
- `x-fern-discriminator-context: data | protocol` on a `discriminator` object says whether the discriminator is a property of the JSON body (default) or protocol framing such as the SSE `event` field (`discriminator-context`).
- `x-fern-global-parameters` lifts common query parameters to the client (`global-parameters`).

If an extension isn't on the overview page, Fern doesn't support it; look for a `generators.yml` setting instead (`fern-sdks` skill) or ask on https://github.com/fern-api/fern/issues.
