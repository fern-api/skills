# Auth, headers, and environments

Auth can come from the OpenAPI `securitySchemes` (default) or be declared in `generators.yml`. `generators.yml` wins when both exist; declare it there when the spec is wrong, when OAuth client-credentials flow is needed, or when SDKs need different auth from the raw API.

## Auth schemes

```yaml
auth-schemes:
  ApiKey:                       # header auth
    header: X-API-Key
    name: apiKey                # SDK parameter name
    env: ACME_API_KEY           # read from this env var when not passed
    prefix: "Key "              # optional value prefix
  Bearer:
    scheme: bearer
    token:
      name: accessToken
      env: ACME_TOKEN
  Basic:
    scheme: basic
    username: { name: accountSid, env: ACME_ACCOUNT_SID }
    password: { name: authToken, env: ACME_AUTH_TOKEN }
  OAuth:
    scheme: oauth
    type: client-credentials
    client-id-env: ACME_CLIENT_ID
    client-secret-env: ACME_CLIENT_SECRET
    get-token:
      endpoint: auth.getToken          # or "POST /oauth/token"; "ns::POST /path" when namespaced
      request-properties:
        client-id: $request.client_id
        client-secret: $request.client_secret
      response-properties:
        access-token: $response.access_token
        expires-in: $response.expires_in
    refresh-token:                     # optional
      endpoint: auth.refresh
      request-properties: { refresh-token: $request.refresh_token }
      response-properties: { access-token: $response.access_token }

api:
  auth: ApiKey                  # one scheme
  # auth: { any: [ApiKey, Bearer] }   # client accepts either
  # auth: { all: [Basic, ApiKey] }    # client requires both
  # auth: endpoint-security           # honour per-operation `security` from the spec
```

Per-SDK override: `groups.<g>.generators[].api.auth: Bearer`.

Generated clients take the auth values as constructor options and fall back to the `env` variable. Omit `env` if you want callers to always pass it explicitly.

## Global headers

```yaml
api:
  headers:
    X-Client-Version: "1.0"               # constant, baked in
    X-Tenant-Id:                          # configurable
      name: tenantId
      type: optional<string>
      env: ACME_TENANT_ID
      client-default: default
```

Resolution order at runtime: per-request option → client constructor → `env` → `client-default`. Headers can also be declared in the spec with `x-fern-global-headers` (see `fern-openapi`).

## Environments and base URLs

Single-host APIs: use OpenAPI `servers` (each gets a name via `x-fern-server-name`), or override in `generators.yml`:

```yaml
api:
  default-environment: Production
  base-url-env: ACME_BASE_URL       # sibling of environments, not inside it. TypeScript only: env var read when no base URL/environment is passed
  environments:
    Production: https://api.acme.com
    Staging: https://api.staging.acme.com
```

Multi-host APIs (different endpoints on different hosts) need both `generators.yml` and the spec:

```yaml
api:
  default-environment: Production
  default-url: api                  # which URL key is the default for endpoints without x-fern-server-name
  environments:
    Production:
      urls:                         # multi-host environments need the urls: wrapper
        api: https://api.acme.com
        auth: https://auth.acme.com
    Staging:
      urls:
        api: https://api.staging.acme.com
        auth: https://auth.staging.acme.com
```

Endpoints are assigned to a host with `x-fern-server-name: auth` in the spec. SDKs expose `environment: AcmeEnvironment.Staging` (naming varies per language) and a `baseUrl`/`base_url` override.

Templated hosts (`https://{region}.api.acme.com`) use OpenAPI server `variables`; the SDK exposes each variable as a client option with the spec's default. Check the language page for which generators support server variables before relying on it.

## SDK variables

Path parameters that are the same for every call (account IDs, project IDs) can be lifted to the client with `x-fern-sdk-variables` / `x-fern-sdk-variable` in the spec, so users set them once at construction. Spec-side detail is in `fern-openapi`.

## Debugging auth problems

- Wrong header name/prefix in generated code → check `prefix` and whether the spec's `securitySchemes` or `auth-schemes` is in effect (`generators.yml` wins).
- SDK demands a credential the API doesn't need for some endpoints → use `api.auth: endpoint-security` or mark operations with `security: []`.
- OAuth token never refreshed → verify `response-properties.expires-in` points at the real field.
