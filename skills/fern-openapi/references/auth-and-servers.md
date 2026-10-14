# Authentication and servers in the spec

## Authentication

Fern reads `components.securitySchemes` and the `security` requirement (global or per operation) and turns them into client constructor options. Supported: `http` `bearer`, `http` `basic`, `apiKey` in `header`. OAuth client-credentials with automatic token fetching is configured in `generators.yml` `auth-schemes` (see `fern-sdks`), not in the spec.

```yaml
security:
  - BearerAuth: []                 # applies to all operations unless overridden
components:
  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      x-fern-bearer:               # SDK option name + env var fallback
        name: apiKey
        env: ACME_API_KEY
    BasicAuth:
      type: http
      scheme: basic
      x-fern-basic:
        username: { name: accountSid, env: ACME_ACCOUNT_SID }
        password: { name: authToken, env: ACME_AUTH_TOKEN }
    ApiKeyAuth:
      type: apiKey
      in: header
      name: X-API-Key
      x-fern-header:
        name: apiKey
        env: ACME_API_KEY
        prefix: "Key "             # optional value prefix
```

Semantics:

- Multiple objects in a `security` list = **OR** (client accepts any one). Multiple schemes in one object = **AND** (client requires all).
- `security: []` on an operation makes it unauthenticated (the SDK still sends auth if configured but doesn't require it).
- Operations with different schemes: set `api.auth: endpoint-security` in `generators.yml` so each method uses its own, otherwise the global scheme wins.
- `generators.yml` `auth-schemes` + `api.auth` override whatever the spec says; use that when you can't change the spec or need OAuth.

Docs: https://buildwithfern.com/learn/api-definitions/openapi/authentication

## Servers and environments

```yaml
servers:
  - url: https://api.acme.com
    x-fern-server-name: Production
  - url: https://api.eu.acme.com
    x-fern-server-name: Production_EU
  - url: https://api.staging.acme.com
    x-fern-server-name: Staging
```

- The first server is the default. Named servers become an environment enum in SDKs (`AcmeEnvironment.Staging`) and selectable servers in the docs playground.
- Unnamed servers get generic names; always add `x-fern-server-name`.
- `x-fern-base-url-env: ACME_BASE_URL` (document level) lets SDKs read the base URL from an environment variable when none is passed (generator support varies; `generators.yml` `environments.base-url-env` takes precedence).

### Multiple hosts for one API

Declare the hosts per environment in `generators.yml` and pin operations to a host with `x-fern-server-name`:

```yaml
# generators.yml
api:
  default-environment: Production
  default-url: api
  environments:
    Production: { api: https://api.acme.com, ai: https://ai.acme.com }
    Staging:    { api: https://api.staging.acme.com, ai: https://ai.staging.acme.com }
```

```yaml
# openapi.yml
paths:
  /chat/completions:
    post:
      x-fern-server-name: ai     # everything else uses `api`
```

### URL templating / server variables

```yaml
servers:
  - url: https://{region}.api.acme.com/{version}
    x-fern-server-name: Production
    variables:
      region:
        default: us
        enum: [us, eu, apac]
      version:
        default: v1
```

Each variable becomes a client option with the default from the spec. Support is generator-specific; check https://buildwithfern.com/learn/api-definitions/openapi/extensions/server-names-and-url-templating and the language page before relying on it, and confirm with a `--preview` build.

Docs: https://buildwithfern.com/learn/api-definitions/openapi/servers
