<br/>

<div align="center">
  <a href="https://www.buildwithfern.com/?utm_source=github&utm_medium=readme&utm_campaign=fern&utm_content=logo">
    <picture>
      <source media="(prefers-color-scheme: dark)" srcset="/fern/docs/assets/fern-logo-white.svg">
      <source media="(prefers-color-scheme: light)" srcset="/fern/docs/assets/fern-logo-primary.svg">
      <img alt="logo" src="/fern/docs/assets/fern-logo-primary.svg" height="50" align="center">
    </picture>
  </a>
  
<br/>
  
  # Fern skills

Official [Agent Skills](https://www.skills.sh) for building with
[Fern](https://buildwithfern.com). These teach coding agents — Claude Code,
Cursor, Codex, Copilot, and others — how to work well in a Fern repository.

## Install

```bash
npx skills add fern-api/skills
```

This discovers the skills in this repo and installs them into your agent
(`.claude/skills/`, `.agents/skills/`, etc.). Install one explicitly with
`--skill`, or all without prompting with `--all`:

```bash
npx skills add fern-api/skills --skill fern-docs
npx skills add fern-api/skills --all
```

Add `-g` to install globally instead of into the current project.

## Skills

| Skill | Use for |
|-------|---------|
| `fern-docs` | Building Fern docs sites: `docs.yml`, navigation, pages, custom MDX, landing pages, and changelog entries. |

Once installed, your agent uses a skill automatically when the task matches —
ask it to add a changelog entry or build out your docs and it pulls in
`fern-docs` on its own. Nothing else to configure.
