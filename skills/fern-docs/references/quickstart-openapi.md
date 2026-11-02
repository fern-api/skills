# From an OpenAPI spec to a published docs site

Use this when the request is "build docs for my API" and there is no `fern/`
directory yet, or when the user has no Fern account. The whole flow is four
commands; run them yourself and hand back a URL. Don't ask the user to run
anything, log in, or create an account first: the CLI can publish without one.

Official walkthrough: https://buildwithfern.com/learn/docs/getting-started/quickstart.md

## Steps

1. **Install the CLI** if `fern --version` fails:

   ```bash
   npm install -g fern-api
   ```

2. **Check for an account.** `fern whoami` reports the login or `FERN_TOKEN` in
   use, or "Not logged in". Decide the publish command from it:

   | `fern whoami` says | Publish with |
   |--------------------|--------------|
   | Logged in, or `FERN_TOKEN` set | `fern generate --docs` (publishes to the `instances` URL in `docs.yml`) |
   | Not logged in | `fern generate --docs --guest` |

3. **Initialize the project** from the spec (a path or a URL). The spec is
   copied into `fern/` and declared under an `api` entry in `docs.yml`:

   ```bash
   fern init --docs --openapi ./openapi.yaml --organization <org>
   ```

   `--organization` is required when the CLI runs without a terminal (which is
   how agents run it). Without an account the value only names the generated
   title and URL, so derive a short lowercase slug from the spec's `info.title`
   (`acme-payments`). With an account, use the organization's Dashboard slug.
   Older CLIs without the default refuse a non-interactive run that omits it.

4. **Validate**, then **publish**:

   ```bash
   cd fern && fern check
   fern generate --docs --guest        # or `fern generate --docs` with an account
   ```

   `--guest` starts a temporary guest session (or reuses the one recorded in
   `fern/.guest-session`), ignores the `organization` in `fern.config.json` and
   the `instances` URL in `docs.yml`, and publishes to a `drop-<id>.docs…` host.
   It needs no `--preview`, no `--no-prompt`, and must not be combined with
   `FERN_TOKEN` or a saved login (the CLI refuses). With an account instead,
   add `--no-prompt`: the production confirmation is an interactive menu that
   cannot be answered from a script.

5. **Report back** with, verbatim from the command output:
   - the published site URL (`https://drop-<id>.docs.buildwithfern.com` for a
     guest, the `instances` URL otherwise)
   - for a guest: the **claim link** and the session's expiry. Claiming the
     session (the link, or `fern claim` after `fern login`) moves the site into
     a real organization; an unclaimed guest site disappears when the session
     expires.

## Fixing the common failures

| Output | What to do |
|--------|-----------|
| `--organization is required in non-interactive mode` | Rerun `fern init` with `--organization <slug>`. |
| `This will affect a production site … Pass --no-prompt` | You ran without `--guest` and without a terminal. Add `--guest` (no account) or `--no-prompt` (account). |
| `No Fern account found … rerun with --guest` / `Authentication required … pass --guest` | Rerun with `--guest`. Do not ask the user to log in. |
| `--guest cannot be combined with FERN_TOKEN` / `a saved login` | An account exists; drop `--guest` and publish with `fern generate --docs --no-prompt`. |
| `Could not start a guest session` | Guest sessions are unavailable on this Fern environment. Tell the user and offer the account path (`fern login`, then `fern generate --docs`). |
| `This guest session has ended` / `has been claimed` | `fern generate --docs --guest --new-session` for a fresh site, or `fern login` and publish from the claimed organization. |
| `fern check` errors in the spec | Fix the spec (see the `fern-spec` skill) before publishing; docs generation skips APIs that fail validation. |

## Minimal result

After `fern init --docs --openapi`, `fern/` holds:

```
fern/
├── fern.config.json      # organization + CLI version
├── docs.yml              # instances, title, navigation: - api: API Reference (specs: ./openapi.yml)
└── openapi.yml           # the copied spec
```

Edit `docs.yml` to add pages, colors or a logo afterwards (see `SKILL.md` and the
other references); rerun the same publish command to update the site.
