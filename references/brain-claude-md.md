# {{WIKI_DIR}} — Operational Schema

This directory is the project's "brain": an interconnected markdown wiki
maintained by Claude. If you are reading this file, you were probably invoked
by the `/{{COMMAND_NAME}}` command. Follow these rules precisely.

---

## What each file is

- `INDEX.md` — dense node catalog. Entry point for any query or ingest.
- `LOG.md` — append-only log of **operations on the wiki** (ingest, lint, rename, merge). This is not the work log — that lives in the nodes.
- `SOURCES.md` — external sources registry. A directory, not a content cache.
- `sessions/` — session nodes. Flat, one file per session.
- `CLAUDE.md` — this file.

---

## Node types

At startup, only **session** exists. Concepts, ADRs, entities, and persons will emerge
organically via `lint` when a pattern repeats in 3+ nodes.

**DO NOT** create nodes of other types preemptively.

---

## Required frontmatter in each node

```yaml
---
type: session
area: <{{AREA_ENUM}}>
date: YYYY-MM-DD
slug: <slug-of-the-filename-without-date-or-.md>
title: "<human-readable title>"
tags: [<free, kebab-case>]
status: active                  # active | superseded | archived
related:
  - <slug-without-.md>
sources:
  - {{SOURCE_PREFIX}}:<id-or-path>
superseded_by: null             # slug of the node that replaces this one, or null
---
```

Rules:
- `slug`: same as the filename without date or `.md`. Kebab-case, ≤6 words. Describes the result, not the process.
- `area`: unambiguous relative value. Single-value. If a session touches two areas, create two cross-linked nodes, or one with a primary area and a cross-ref.
- `related`: slugs without `.md`. Max 5. The LLM detects them by overlapping tags, same area, or chronological proximity.
- `sources`: valid prefixes: `{{SOURCE_PREFIX}}:<id>`, `repo:<relative-path>`, `url:<full-url>`.

---

## Link convention — Foam Wikilinks

**Critical rule**: all internal wiki links use `[[slug]]`, compatible with Foam/Markdown Notes in VSCode.

- **Syntax**: `[[slug]]` — only the filename without `.md`, without path, without alias. E.g.: `[[2026-04-05-node-name]]`.
- **Resolution**: Foam resolves the slug to the file by name. Slugs are unique by construction (they include a date).
- **Where they are used**:
  - `INDEX.md`: each catalog entry
  - Nodes: `## {{SECTION_CROSS_REFS}}` section
  - Nodes: `## {{SECTION_SOURCES}}` section when pointing to another wiki node or internal anchor (`[[sources#id]]`)
- **Where they are NOT used**:
  - External URLs: use markdown link `[text](url)`
  - Files outside the vault `{{WIKI_DIR}}/`: literal backticks
  - "Historical origin" in {{SECTION_SOURCES}}: plain text, **do not link**

---

## Required body of each session node

Sections in this exact order:

```
# <{{SECTION_TITLE}}>

## {{SECTION_CONTEXT}}

## {{SECTION_DECISIONS}}

## {{SECTION_OUTPUT}}

## {{SECTION_PENDING}}

## {{SECTION_CROSS_REFS}}
- [[slug]] — reason in one line

## {{SECTION_SOURCES}}
- [[sources#id]]
- Historical origin (do not modify): `original/path.md`
```

Body rules:
- `{{SECTION_CROSS_REFS}}`: **only wikilinks** to other wiki nodes. Each bullet with a reason in one line. No reason → orphan-link candidate in lint.
- `{{SECTION_SOURCES}}`: external references. For external sources: `[[sources#id]]`. For web URLs: `[text](url)`. For repo files outside the vault: plain text or backticks.

---

## Ingest rules

When `/{{COMMAND_NAME}} ingest` is invoked:

1. Read `{{WIKI_DIR}}/CLAUDE.md` and `{{WIKI_DIR}}/INDEX.md`.
2. Review the current conversation. Identify:
   - Area(s) covered.
   - Main action verb (defined, migrated, decided, implemented, debugged).
   - Concrete output: files created/modified, URLs.
   - Decisions with rationale.
   - Explicit pending items.
3. Generate the slug: `YYYY-MM-DD-<kebab-case>`. Date = today. The slug describes the RESULT (not the process), max 6 words.
4. Does a node from the same day with a very similar topic already exist?
   - **Yes** → UPDATE: read it and append to {{SECTION_DECISIONS}}/{{SECTION_OUTPUT}}/{{SECTION_PENDING}} under `### Update [HH:MM]`. Do not rewrite what was there before.
   - **No** → CREATE.
5. CREATE: generate complete frontmatter. `related` searches in `INDEX.md` for nodes with overlapping tags, same area, or chronological proximity. Max 5.
6. Write the body with the 6 required sections. Cross-refs with `[[slug]]` and a reason in one line.
7. **Mandatory bidirectionality**: for each node in `related`, open it and add a reciprocal bullet in its `## {{SECTION_CROSS_REFS}}` section: `[[this-node]] — reason`. No exceptions.
8. Update `{{WIKI_DIR}}/INDEX.md`: insert the bullet `[[slug]] — one-liner` at the top of the area section. If the area does not exist, create it in alphabetical order.
9. Append to `{{WIKI_DIR}}/LOG.md`: `## [YYYY-MM-DD HH:MM] ingest | <slug>` with cross-ref metadata.
10. Report to the user: node path, cross-refs added, ambiguous decisions.

{{LEGACY_NOTE}}

---

## Query rules

When `/{{COMMAND_NAME}} query <question>` is invoked:

1. Read `{{WIKI_DIR}}/CLAUDE.md` and `{{WIKI_DIR}}/INDEX.md` in full.
2. From the index one-liners, select 1-5 candidate nodes.
3. Read those nodes in full.
4. {{BACKEND_QUERY_RULE}}
5. Synthesize the answer in 2-5 paragraphs with citations using wikilinks: `[[node-slug]]`. **DO NOT use markdown links** for wiki nodes.
6. If you detect a gap (an obvious missing cross-ref, a concept in 3+ nodes without its own node), do NOT fix it — report it at the end as "Suggestion for `/{{COMMAND_NAME}} lint`".
7. If there is not enough information, say so explicitly. **Do not invent or extrapolate** beyond what the nodes say.

---

## Lint rules

When `/{{COMMAND_NAME}} lint` is invoked:

1. Read `{{WIKI_DIR}}/CLAUDE.md`, `{{WIKI_DIR}}/INDEX.md`, and **all** files in `{{WIKI_DIR}}/sessions/`.
2. Check these categories:
   - **Orphan nodes**: nodes with no inbound wikilinks from other nodes or from `INDEX.md`.
   - **Broken wikilinks**: `[[slug]]` that does not resolve to any file in `sessions/`.
   - **Stale claims**: pending item dates that have already passed.
   - **Missing cross-refs**: pairs with high tag overlap or the same entity without a wikilink between them.
   - **Emerging concepts**: terms that appear in 3+ nodes without their own node. Report as candidates, **do not create**.
   - **Contradictions**: opposing decisions without `superseded_by`.
   - **Invalid frontmatter**: missing fields or values outside the enum.
   - **Out-of-sync index**: a node in `sessions/` without an entry in `INDEX.md`, or vice versa.
   - **Misused markdown links**: links to wiki nodes using `[text](path)` instead of `[[slug]]`.
3. Return a markdown report structured by category with an actionable suggestion per item.
4. **DO NOT modify files.** Only append a short entry to `LOG.md` with counts.
5. Append to `{{WIKI_DIR}}/LOG.md`: `## [date] lint | report` with count per category.

---

## {{BACKEND_SECTION_TITLE}}

{{BACKEND_SECTION_BODY}}

---

## Language

{{LANGUAGE_RULE}}
