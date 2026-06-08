# SOURCES.md templates by backend

Select the block that corresponds to the backend chosen by the user.
Replace `{{WIKI_DIR}}` with the name of the wiki directory.

---

## TEMPLATE A — Notion

```markdown
# {{WIKI_DIR}} — External Sources Registry

> Short IDs for referencing from frontmatter (`sources:`) and node bodies.
> Notion is the **source of truth**. When a node cites a source from here, Claude MUST
> query Notion via `mcp__notion__notion-fetch` before asserting its content.
> The "Cache summary" field is only used to decide whether it is worth fetching — it is NOT authoritative.

---

## Notion

<!-- Add one entry for each Notion page relevant to the project -->
<!-- Format: short ID in kebab-case, title, full URL, short summary -->

### example-page
- **Title:** Page name
- **URL:** https://www.notion.so/<page-id>
- **Cache summary:** Brief description of the content (to decide whether to fetch).
- **Rule:** Resolve real content via MCP before making any assertion.
- **Last verified:** YYYY-MM-DD

<!-- Duplicate the block above for each Notion source -->

---

## Repo (project files that are referenced as sources)

<!-- Important files that are NOT wiki nodes but are cited in sessions -->

### example-doc
- **Path:** `relative/path/to/file.md`
- **Description:** What this file contains.
```

---

## TEMPLATE B — Local files

```markdown
# {{WIKI_DIR}} — External Sources Registry

> Registry of repo files that serve as the source of truth.
> Reference from frontmatter with the `repo:<path>` prefix.
> For current content, always read the file directly — this registry
> is only a directory of what exists and where.

---

## Repo files

<!-- List important files that are the source of truth for sessions -->
<!-- Reference from nodes with: sources: [repo:path/to/file.md] -->

### example-doc
- **Path:** `relative/path/to/file.md`
- **Description:** What it contains and what it is used for.

<!-- Add more entries as needed -->

---

## External URLs

<!-- Documentation, APIs, external references that are not Notion -->

### example-url
- **URL:** https://example.com/docs
- **Description:** What it is referenced for.
```

---

## TEMPLATE C — None / standalone

```markdown
# {{WIKI_DIR}} — External Sources Registry

> This wiki operates without configured external sources.
> To reference project files, use the `repo:<path>` prefix in the frontmatter.
> For external URLs, use the `url:<url>` prefix in the frontmatter.

---

## Referenced repo files

<!-- Add when a session depends on a key project file -->

---

## External URLs

<!-- Add when a session depends on an external URL -->
```

---

## TEMPLATE D — Custom backend

```markdown
# {{WIKI_DIR}} — External Sources Registry

> External sources registry for the project.
> Configured backend: {{CUSTOM_BACKEND_NAME}}
>
> {{CUSTOM_BACKEND_INSTRUCTIONS}}

---

## {{CUSTOM_BACKEND_NAME}}

<!-- List sources according to your tool's convention -->

### example
- **ID / Path / URL:** ...
- **Description:** ...

---

## Referenced repo files

### example-doc
- **Path:** `relative/path/to/file.md`
- **Description:** ...
```
