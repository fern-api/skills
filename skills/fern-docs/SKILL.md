---
name: fern-docs
description: >-
  Author and edit Fern documentation: MDX pages, navigation, docs.yml config,
  custom components, landing pages, and changelog entries. Use when working in a
  Fern docs repo (a fern/ directory with docs.yml). Routes to a detailed
  reference per task.
---

# Fern docs authoring

Write and edit documentation on a Fern site. `SKILL.md` is the index: follow the
principles and voice below, then read the `references/` file that matches the
task before writing. Don't work from this page alone when a reference exists.

## Fern resources

Fern's configuration evolves. Look things up rather than guessing.

- **Docs:** https://buildwithfern.com/learn/docs/getting-started/overview
- **Page index:** https://buildwithfern.com/learn/llms.txt
- **MCP server** — connect your agent to Fern's live docs and query it for any
  syntax or behavior you're unsure of. The endpoint works with any MCP client;
  for Claude Code:

  ```bash
  claude mcp add --transport http fern https://buildwithfern.com/learn/_mcp/server
  ```

## Routing

| Task | Read |
|------|------|
| Writing or editing a changelog entry | `references/changelog.md` |

## Core principles

- **Write what the reader needs to succeed — no more.** Every sentence earns its place.
- **Prefer editing over creating.** Search the repo for a page that already
  covers the topic and update it instead of adding a duplicate.
- **Make minimal, precise edits.** Don't rewrite a page when a paragraph fix will do.
- **Push back when something seems wrong.** Explain why rather than complying silently.
- **Ask when unclear.** Don't fill gaps with assumptions.
- **Never fabricate.** If you don't know a config key or behavior, look it up
  (MCP server or docs) or say so. Don't invent frontmatter or YAML fields.
- **Cross-reference.** When you mention a concept documented elsewhere, link to it
  so readers can find their way.

## Voice

Dry and direct. State requirements and behavior plainly.

- **Cut hedges and nudges:** "just", "simply", "make sure you", "you'll want to".
  Prefer "X requires Y" over "make sure you have Y so you can do X".
- **Cut connective filler:** "so that", "in order to", "be sure to".
- **Limit em dashes** to one per short paragraph; reach for a colon or
  parentheses before a second.
- **No conversational framing in callouts or steps.** "Localization requires the
  latest CLI" beats "Localization is under active development, so make sure
  you're on the latest CLI before configuring it."

## Links

Internal links are URL paths derived from **this repo's own `docs.yml` and page
frontmatter** — not file paths on disk or relative paths. Nearly every edit
touches a link, so get this right every time.

A page URL is `<base-path>/<product-slug>/<rest>`:

- **Base path** — the prefix in your docs URL (e.g. `/learn`, `/docs`; may be empty).
- **Product slug** — the product's `slug` in `fern/docs.yml`. Omitted for a
  product with no `slug:` (such as a default single Home product).
- **`<rest>`** — found by step 1 or step 2 below.

To find `<rest>`:

1. **Open the target page and check its frontmatter for `slug:`.** A frontmatter
   slug is **absolute within the product**: it *is* `<rest>` on its own, and it
   replaces the entire section/folder hierarchy. The section's slug does not
   appear. Do not walk the navigation.
2. **Otherwise, walk `docs.yml`** within the product, from section through any
   folders to the page. Each level contributes one slug:
   - Its explicit `slug:` if set.
   - Otherwise auto-derived from the display name: lowercased, spaces to
     hyphens, special characters stripped (so `v3 (Latest)` becomes
     `v-3-latest`). Auto-derivation isn't always obvious — verify, don't assume.
   - In folder-based navigation, a page's slug comes from its **filename**, not
     a display name (`quick-start.mdx` to `quick-start`).
   - **Omit any level with `skip-slug: true`** — it contributes no segment.

Example — page `./pages/ai/overview.mdx` under section "AI features", in product
`docs`, base path `/learn`:

- with frontmatter `slug: ai-ai` → `/learn/docs/ai-ai` (the `ai-features`
  section slug is dropped)
- with no frontmatter slug → `/learn/docs/ai-features/overview` (section
  auto-slug `ai-features` + page auto-slug `overview`)

Section slugs frequently differ from folder names, so never guess the URL from
the directory layout. If you can't resolve a path from `docs.yml` and the page's
frontmatter, say so rather than guessing.

**Anchors** append `#heading`, and only resolve for real `##`/`###` Markdown
headings — not for JSX title props like `<Step title>`, `<Tab title>`,
`<Accordion title>`, or `<Card title>`.

**Not internal links** (leave as-is): external `https://` URLs, image paths
(`./images/...`), snippet includes (`<Markdown src="/snippets/..." />`), and
same-page anchors (`#section`).
