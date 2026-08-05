# Prose updates from an API change

When an API definition changes, the auto-generated API reference updates itself —
but the prose around it can go stale. This reference covers how to update
**prose/guide/narrative pages** so they still match the API. Read it whenever a
diff to an API spec (OpenAPI, Fern Definition, or the generated SDK) prompts a
docs change.

## Don't touch the generated API reference

Fern regenerates the API reference from the spec on every build — endpoint pages,
parameter tables, schemas, and request/response examples. Editing those by hand
is wasted work: the change is either overwritten on the next build or, worse,
drifts from the source of truth.

- **Only edit prose** — guides, tutorials, quickstarts, conceptual pages,
  overviews, and hand-written narrative. These are the `.mdx` files you actually
  authored, not the ones Fern emits from the API definition.
- If the reference itself is wrong, fix the **API definition**, not the generated
  page. That's outside this skill — flag it rather than patching the rendered output.

## Find the prose the change made stale

A spec diff rarely touches every page. Identify the prose that now contradicts the
API before editing anything.

- Pull the concrete identifiers out of the diff: endpoint paths, method names,
  field and parameter names, enum values, auth scheme. Grep the prose for each:

  ```bash
  grep -rln "<endpoint>\|<field>\|<methodName>" fern --include="*.mdx"
  ```

- Read each hit and decide whether the change actually invalidates it — a renamed
  field in a code sample, a removed parameter described in a step, a changed
  default mentioned in passing. Skip pages that merely link to the reference; the
  reference updates itself.
- **Prefer editing existing pages over creating new ones.** A changed API usually
  means an existing guide is now slightly wrong, not that a new guide is needed.
  Search for the page that already covers the workflow and fix it in place.
- **Make minimal, targeted edits.** Change the sentence, sample, or step the diff
  invalidated — don't rewrite a page that's still mostly correct.

## Additive-only changes often need no prose edits

When the change is purely additive — a new optional field, a new endpoint, a new
enum value — and nothing in the existing prose now reads as wrong, it's fine to
make **no prose edits at all**. The generated reference already reflects the
addition. Don't invent a guide for every new endpoint; add prose only when a
reader following the existing docs would otherwise be misled or stuck.

## Cross-reference the pages you touch

A prose edit that introduces or changes a concept usually leaves related pages
pointing at stale behavior or failing to mention the new one. Apply the
**Cross-referencing** rules in `SKILL.md`: one canonical home, grep for the
feature and related keywords, use the lightest link form that works, and phrase
links on a noun already in the sentence.

Internal links are URL paths from this repo's `docs.yml` and page frontmatter —
**not** file paths or relative paths on disk. Build every link per the **Links**
section of `SKILL.md`; never derive a URL from the directory layout.

## What not to do

- Don't hand-edit generated API reference pages — Fern overwrites them.
- Don't create a new page when an existing one can be corrected.
- Don't rewrite a page when a targeted edit fixes the staleness.
- Don't manufacture prose for a purely additive change the reference already covers.
- Don't write links from file paths — resolve them from `docs.yml` per `SKILL.md`.
