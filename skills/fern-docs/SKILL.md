---
name: fern-docs
description: >-
  Author and edit Fern documentation: MDX pages, navigation, docs.yml config,
  API reference, landing pages, and changelog entries. Use when working in a
  Fern docs repo (a fern/ directory with docs.yml). Routes to a detailed
  reference per task.
---

# Fern docs authoring

This skill covers writing and editing documentation on a Fern site. `SKILL.md`
is an index: identify the task, then read the matching file in `references/`
before writing. Don't work from this page alone when a reference exists.

## Routing

| Task | Read |
|------|------|
| Writing or editing a changelog entry | `references/changelog.md` |

## Shared conventions

These apply to every docs task, regardless of which reference you're following.

- **Voice.** Dry and direct. Cut hedges ("just", "simply", "make sure you") and
  filler ("in order to", "so that"). At most one em dash per short paragraph.
- **Internal links are URL paths, not disk paths.** Build them from `docs.yml`,
  not from a file's location on disk. Full rules will live in
  `references/navigation.md`.
- **Prefer editing over duplicating.** Search for an existing page on the topic
  before creating a new one; one page owns each topic and others link to it.
- **Don't fabricate.** If a config option or behavior isn't confirmed, say so
  rather than inventing frontmatter keys or YAML fields.
