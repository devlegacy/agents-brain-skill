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
- `plannings/` — planning nodes. Living documents that span multiple sessions.
- `CLAUDE.md` — this file.

---

## Node types

Two explicit types exist from the start:

**`session`** — point-in-time log of a conversation. Append-only after creation. Created automatically by `/{{COMMAND_NAME}} ingest`.

**`planning`** — living specification document that spans multiple sessions. Tracks tasks, progress, and the sessions that created and advanced it. Created explicitly via `/{{COMMAND_NAME}} plan <slug>`. Updated in-place as tasks are completed across sessions.

Concepts, ADRs, entities, and persons emerge organically via `lint` when a pattern repeats in 3+ nodes. **DO NOT** create those types preemptively.

---

## Required frontmatter — session nodes

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

## Required frontmatter — planning nodes

```yaml
---
type: planning
area: <{{AREA_ENUM}}>
slug: <kebab-case>              # NO date prefix — plannings are living docs, not dated events
title: "<human-readable title>"
tags: [<free, kebab-case>]
status: draft | active | in-progress | completed | cancelled
progress: 0                     # integer 0–100, updated as tasks are completed
created_by: <session-slug>      # slug of the session that created this plan
updated_by:                     # ordered list of session slugs that advanced this plan
  - <session-slug>
related:                        # related planning nodes (not sessions — those go in updated_by)
  - <planning-slug>
superseded_by: null
---
```

Rules:
- `slug`: kebab-case, no date prefix. Must be globally unique across all nodes. ≤6 words. Describes what is being planned, not when.
- `progress`: updated by ingest when session tasks advance this plan. Never set manually to 100 — use `status: completed` instead.
- `created_by`: set by ingest on the session that first mentions creating this plan. May be null until the first ingest after plan creation.
- `updated_by`: append-only. Ingest appends the session slug each time tasks are completed. Never remove entries.

---

## Link convention — Foam Wikilinks

**Critical rule**: all internal wiki links use `[[slug]]`, compatible with Foam/Markdown Notes in VSCode.

- **Syntax**: `[[slug]]` — only the filename without `.md`, without path, without alias. E.g.: `[[2026-04-05-node-name]]` for sessions, `[[frontend-architecture]]` for plannings.
- **Resolution**: Foam resolves the slug to the file by name. Session slugs are unique by date prefix. Planning slugs are unique by name convention.
- **Where they are used**:
  - `INDEX.md`: each catalog entry
  - Nodes: `## {{SECTION_CROSS_REFS}}` section
  - Nodes: `## {{SECTION_SOURCES}}` section when pointing to another wiki node or internal anchor (`[[sources#id]]`)
  - Planning nodes: `created_by` and `updated_by` fields reference session slugs (but written as plain slugs in frontmatter, not wikilinks)
- **Where they are NOT used**:
  - External URLs: use markdown link `[text](url)`
  - Files outside the vault `{{WIKI_DIR}}/`: literal backticks
  - "Historical origin" in {{SECTION_SOURCES}}: plain text, **do not link**

---

## Required body — session nodes

Sections in this exact order:

```
# <title from frontmatter>

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

## Required body — planning nodes

Sections in this exact order:

```
# <title from frontmatter>

## Objective
What this plan achieves and why it exists.

## Context
Background, constraints, dependencies, and architectural decisions already in place.

## Tasks
- [ ] Task 1
  - [ ] Subtask 1.1
- [x] Completed task

## Decisions
Key choices made during planning and execution.
Link to the session where each decision was made: [[session-slug]] — decision summary.

## Progress
### [YYYY-MM-DD] <one-line summary of session advance>
- Completed: <tasks done>
- Remaining: <tasks left>
- Updated by: [[session-slug]]

## Cross-refs
- [[session-slug]] — created this plan
- [[session-slug]] — completed task X
- [[related-planning-slug]] — depends on / blocks this plan
```

Body rules:
- `## Tasks`: the only mutable section updated across sessions. Use `- [x]` for completed, `- [ ]` for pending. Indent subtasks with 2 spaces.
- `## Progress`: append-only. Each session that advances this plan adds one block. Do NOT rewrite previous blocks.
- `## Decisions`: append decisions as they are made. Link each to the session where it was decided.
- `## Cross-refs`: link sessions that created or advanced this plan. Link related planning nodes with a reason.

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
11. **Planning update**: scan the conversation for mentions of planning nodes (by slug or title).
    For each planning node referenced:
    a. Read the planning node from `{{WIKI_DIR}}/plannings/`.
    b. Check off any tasks that were explicitly completed in this conversation.
    c. Append the new session slug to `updated_by` in the frontmatter.
    d. Estimate `progress` from the ratio of checked tasks and update the frontmatter field.
    e. Append a new block to `## Progress`: date, one-line summary, completed tasks, remaining tasks, session wikilink.
    f. Add `[[this-session-slug]] — advanced this plan` to the planning's `## Cross-refs`.
    g. Add `[[planning-slug]] — advanced this plan` to the session's `## {{SECTION_CROSS_REFS}}`.
    h. Append to `{{WIKI_DIR}}/LOG.md`: `## [YYYY-MM-DD HH:MM] update | <planning-slug>`
    If no planning is referenced, skip this step silently.

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

1. Read `{{WIKI_DIR}}/CLAUDE.md`, `{{WIKI_DIR}}/INDEX.md`, all files in `{{WIKI_DIR}}/sessions/`, and all files in `{{WIKI_DIR}}/plannings/`.
2. Check these categories:

   **Sessions:**
   - **Orphan nodes**: session nodes with no inbound wikilinks from other nodes or from `INDEX.md`.
   - **Broken wikilinks**: `[[slug]]` that does not resolve to any file in `sessions/` or `plannings/`.
   - **Stale claims**: pending item dates that have already passed.
   - **Missing cross-refs**: pairs with high tag overlap or the same entity without a wikilink between them.
   - **Emerging concepts**: terms that appear in 3+ nodes without their own node. Report as candidates, **do not create**.
   - **Contradictions**: opposing decisions without `superseded_by`.
   - **Invalid frontmatter**: missing fields or values outside the enum.
   - **Out-of-sync index**: a node in `sessions/` without an entry in `INDEX.md`, or vice versa.
   - **Misused markdown links**: links to wiki nodes using `[text](path)` instead of `[[slug]]`.

   **Plannings:**
   - **Stale plannings**: `status: in-progress` with no entry in `updated_by` in the last 30 days — flag as potentially abandoned.
   - **Completed plannings**: all tasks checked but `status` is not `completed` — suggest updating status.
   - **Broken planning refs**: `created_by` or `updated_by` slugs that do not resolve to any file in `sessions/`.
   - **Out-of-sync index**: a node in `plannings/` without an entry in `INDEX.md`, or vice versa.
   - **Orphan plannings**: planning with no session in `updated_by` and `created_by` is null — never linked to any session.

3. Return a markdown report structured by category with an actionable suggestion per item.
4. **DO NOT modify files.** Only append a short entry to `LOG.md` with counts.
5. Append to `{{WIKI_DIR}}/LOG.md`: `## [date] lint | report` with count per category.

---

## Plan rules

When `/{{COMMAND_NAME}} plan <slug>` is invoked:

**If the planning node already exists** (`{{WIKI_DIR}}/plannings/<slug>.md`):
1. Read the planning node in full.
2. Show the user: title, status, progress, open tasks, and last session in `updated_by`.
3. Ask: "What do you want to update? (tasks / objective / context / status / other)"
4. Apply the requested changes in-place.
5. Append to `{{WIKI_DIR}}/LOG.md`: `## [YYYY-MM-DD HH:MM] update | <slug>`

**If the planning node does NOT exist** → CREATE:
1. Ask the user (can ask all at once):
   - "What is the objective of this plan?"
   - "Which area does it belong to? ({{AREA_ENUM}})"
   - "List the initial tasks (you can refine them later)."
2. Generate the planning node with:
   - `status: draft`
   - `progress: 0`
   - `created_by: null` (will be set by the next ingest that references this plan)
   - `updated_by: []`
3. Write the full body with all required sections. Populate `## Objective`, `## Context`, and `## Tasks` from the user's answers. Leave `## Progress` and `## Decisions` empty with *(none yet)*.
4. Update `{{WIKI_DIR}}/INDEX.md`: insert `[[slug]] — one-liner` under `## Plannings > <area>`. If the area section does not exist, create it in alphabetical order.
5. Append to `{{WIKI_DIR}}/LOG.md`: `## [YYYY-MM-DD HH:MM] plan | <slug>`
6. Report to the user: path created, reminder to run `ingest` at the end of sessions that advance this plan.

**If no slug is provided**: respond "Provide a slug for the plan (e.g., `/{{COMMAND_NAME}} plan frontend-architecture`)." and abort.

---

## {{BACKEND_SECTION_TITLE}}

{{BACKEND_SECTION_BODY}}

---

## Language

{{LANGUAGE_RULE}}
