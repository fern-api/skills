<br/>

<div align="center">
  <a href="https://www.buildwithfern.com/?utm_source=github&utm_medium=readme&utm_campaign=fern&utm_content=logo">
    <picture>
      <source media="(prefers-color-scheme: dark)" srcset="assets/fern-logo-white.svg">
      <source media="(prefers-color-scheme: light)" srcset="assets/fern-logo-primary.svg">
      <img alt="logo" src="assets/fern-logo-primary.svg" height="50" align="center">
    </picture>
  </a>
  
<br/>
  
  # Fern skills

Official [Agent Skills](https://www.skills.sh) for building with
[Fern](https://buildwithfern.com). These teach coding agents — Claude Code,
Cursor, Codex, Copilot, and others — how to work well in a Fern repository.

</div>

## Install

```bash
npx skills add fern-api/skills
```

This lists the skills in this repo and installs the ones you pick into your
agent's skills directory (`.claude/skills/`, `.agents/skills/`, `.cursor/skills/`,
…). Common variants:

```bash
npx skills add fern-api/skills --skill fern-sdks        # one skill
npx skills add fern-api/skills --all                    # every skill, no prompt
npx skills add fern-api/skills -a claude-code -a cursor # pick the agents (repeat -a)
npx skills add fern-api/skills -g                       # user-level instead of this project
npx skills add fern-api/skills -l                       # just list what's available
npx skills ls                                           # what is installed here
npx skills update                                       # pull the latest versions
```

Requires Node 18+. Skills are plain Markdown, so you can also copy a
`skills/<name>/` folder into your agent's skills directory by hand.

## Skills

| Skill | Use for |
|-------|---------|
| `fern-docs` | Building Fern docs sites: `docs.yml`, navigation, pages, custom MDX, landing pages, changelog entries, and access control (authentication & RBAC). |
| `fern-sdks` | Generating and publishing SDKs: `fern.config.json`, `generators.yml` (groups, generators, outputs, GitHub modes), auth schemes and environments, `.fernignore` custom code, `fern generate` / `fern check` / `fern generator upgrade`, registry publishing, troubleshooting generated code. |
| `fern-spec` | Making OpenAPI/AsyncAPI specs Fern-friendly: `operationId`/schema hygiene, `x-fern-*` extensions (method names, pagination, examples, audiences, global headers, SDK variables), overlays vs overrides, auth and servers, spec sync. |
| `fern-workflows` | Running Fern in CI/CD: `fern check` on PRs, docs preview links and publishing, SDK release workflows and publishing modes, OpenAPI spec sync, `FERN_TOKEN`/registry secrets, runner-only failures. |

### Which one do I need?

| You want to… | Skill |
|--------------|-------|
| Add a docs page, change navigation, write a changelog entry, style the site | `fern-docs` |
| Add a language, change where an SDK is published, keep hand-written code across regenerations, debug `fern generate` | `fern-sdks` |
| Fix odd SDK method names, add pagination/examples/errors, decide between editing the spec, an overlay or an overrides file | `fern-spec` |
| Run `fern check` on pull requests, preview docs per PR, release an SDK from GitHub Actions, keep the spec in sync | `fern-workflows` |

The skills cross-reference each other, so install the ones that match your repo
(a docs repo wants `fern-docs` + `fern-workflows`; an SDK config repo wants
`fern-sdks` + `fern-spec` + `fern-workflows`) or just install `--all`.

## Using a skill

Once installed, your agent reads each skill's description and uses it when the
task matches — nothing else to configure. For example:

- "Add a changelog entry for the new webhooks endpoint" → `fern-docs`
- "Publish our Python SDK to PyPI as `acme-sdk`" → `fern-sdks`
- "The SDK method is called `postV1Users`, make it `users.create`" → `fern-spec`
- "Add a GitHub Action that runs `fern check` on every PR" → `fern-workflows`

If the agent doesn't pick a skill up, name it: "use the `fern-sdks` skill to…"
(Claude Code also accepts `/fern-sdks`). Each skill is an index (`SKILL.md`)
plus `references/*.md`; the agent reads the index first and then only the
reference that matches the task, so the skills stay cheap in context.

To try a skill without installing it:

```bash
npx skills use fern-api/skills@fern-sdks
```

### What the skills assume

- The Fern CLI: `npm install -g fern-api` (or `npx fern-api`). `fern check`
  works offline; `fern generate` needs `fern login` or a `FERN_TOKEN`.
- A `fern/` directory in your repo (`fern init` creates one). The skills tell
  the agent to run `fern check` after every config or spec change.
- Docs links in the skills point at https://buildwithfern.com/learn; append
  `.md` to any docs URL to fetch it as Markdown.

## Repository layout

```
skills/
  <skill-name>/
    SKILL.md          # frontmatter (name, description) + index + rules
    references/*.md   # one file per task area, read on demand
```

Every `generators.yml`/OpenAPI example in the skills is checked with
`fern check`, every GitHub Actions example with `actionlint`, and every docs
link is checked to resolve. Contributions welcome: `npx skills init <name>`
scaffolds a new skill; keep `SKILL.md` short and put detail in `references/`.
