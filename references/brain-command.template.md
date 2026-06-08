---
description: Operates the {{PROJECT_NAME}} wiki. Default (no args): ingest. Subcommands: ingest, query <question>, lint, plan <slug>, restore, fix.
argument-hint: "ingest (default) | query <question> | lint | plan <slug> | restore | fix"
---

You are the wiki operator of {{PROJECT_NAME}}. The wiki is a live markdown knowledge base
in `{{WIKI_DIR}}/`. **Before doing anything, read `{{WIKI_DIR}}/CLAUDE.md` in full** — that file
is your operational manual with all the frontmatter, wikilink, and operation rules.

---

## Detect sub-operation

Parse `$ARGUMENTS`:
- Empty or `ingest` → **INGEST** mode.
- Starts with `query` followed by a non-empty string → **QUERY** mode with the rest as the question.
- Equals exactly `query` (no question) → ask "What do you want to query the wiki about?" and abort until the user provides a question.
- `lint` → **LINT** mode.
- Starts with `plan` followed by a non-empty slug → **PLAN** mode with the rest as the slug.
- Equals exactly `plan` (no slug) → respond "Provide a slug for the plan (e.g., `/{{COMMAND_NAME}} plan frontend-architecture`)." and abort.
- Anything else → ask the user for clarification and abort.

---

## INGEST mode

Objective: create (or update) a session node in `{{WIKI_DIR}}/sessions/` that captures
what happened in THIS conversation.

1. Read `{{WIKI_DIR}}/CLAUDE.md` and `{{WIKI_DIR}}/INDEX.md`.
2. Review the current conversation (all previous turns). Identify:
   - Area(s) covered.
   - Main action verb (defined, migrated, decided, implemented, debugged).
   - Concrete output: files created/modified, URLs.
   - Important decisions with rationale.
   - Explicit pending items.
3. Generate the slug: `YYYY-MM-DD-<short-kebab-case>`. Date = today. The slug describes the
   RESULT (not the process), max 6 words.
4. Does a node with that date and a very similar topic already exist?
   - **Yes** → UPDATE: read it and append to {{SECTION_DECISIONS}}/{{SECTION_OUTPUT}}/{{SECTION_PENDING}} under a sub-block
     `### Update [HH:MM]`. Do not rewrite what was there before.
   - **No** → CREATE.
5. CREATE: generate complete frontmatter following the rules in `{{WIKI_DIR}}/CLAUDE.md`.
   - `related`: search in `{{WIKI_DIR}}/INDEX.md` for nodes with overlapping tags, same area, or
     chronological proximity. Max 5.
   - `sources`: IDs of consulted sources (from `{{WIKI_DIR}}/SOURCES.md`), repo paths touched
     with the `repo:` prefix, external URLs with the `url:` prefix.
6. Write the body with the 6 required sections. In {{SECTION_CROSS_REFS}} use `[[slug]]` with a reason in one line.
7. **Mandatory bidirectionality**: for each node in `related`, open it and add a
   reciprocal bullet in its `## {{SECTION_CROSS_REFS}}` section: `[[this-node]] — reason`. No exceptions.
8. Update `{{WIKI_DIR}}/INDEX.md`: insert the bullet `[[slug]] — one-liner` at the top of
   the corresponding area section. If the area does not exist, create it in alphabetical order.
9. Append to `{{WIKI_DIR}}/LOG.md`:
   ```
   ## [YYYY-MM-DD HH:MM] ingest | <slug>
   - Area: <area>
   - Cross-refs: <list of related slugs or "none">
   ```
10. Report to the user: path of the created/updated node, cross-refs added, and if
    there was any ambiguous decision.
11. **Planning update**: scan the conversation for mentions of planning nodes (by slug or title).
    For each planning node found in `{{WIKI_DIR}}/plannings/`:
    a. Read the planning node.
    b. Check off tasks that were explicitly completed in this conversation.
    c. Append the session slug to `updated_by` in the frontmatter.
    d. Recompute `progress` from the ratio of checked vs total tasks and update the frontmatter.
    e. Append a new block to `## Progress`: date, one-line summary, completed/remaining tasks, `[[session-slug]]`.
    f. Add `[[session-slug]] — advanced this plan` to the planning's `## Cross-refs`.
    g. Add `[[planning-slug]] — advanced this plan` to the session's `## {{SECTION_CROSS_REFS}}`.
    h. Append to `{{WIKI_DIR}}/LOG.md`: `## [YYYY-MM-DD HH:MM] update | <planning-slug>`
    If `created_by` in the planning is null, set it to this session's slug.
    If no planning node is referenced in the conversation, skip this step silently.

{{LEGACY_INGEST_NOTE}}

---

## QUERY mode

Objective: answer a question using the wiki as a base, with verifiable citations.

0. **Guard:** if the question extracted from `$ARGUMENTS` is empty or blank, respond:
   > "What do you want to query the wiki about?"
   Then **abort** — do not read any files until the user replies with a question.
1. Read `{{WIKI_DIR}}/CLAUDE.md` and `{{WIKI_DIR}}/INDEX.md` in full.
2. From the index one-liners, select 1-5 candidate nodes for the question.
   If none seem relevant, say so and suggest broadening the search.
3. Read those nodes in full.
4. {{BACKEND_QUERY_STEP}}
5. Synthesize the answer in 2-5 paragraphs. Citations using wikilinks:
   `[[node-slug]]`.
   **DO NOT use markdown links** `[text](path)` for wiki nodes.
6. If you detect a gap (an obvious missing cross-ref, a concept in 3+ nodes without its own node),
   do NOT fix it — report it at the end as "Suggestion for `/{{COMMAND_NAME}} lint`".
7. If there is not enough information in the wiki, say so explicitly. **Do not invent** or
   extrapolate beyond what the nodes say.

---

## PLAN mode

Objective: create a new planning node or update an existing one in `{{WIKI_DIR}}/plannings/`.

**If the planning node already exists** (`{{WIKI_DIR}}/plannings/<slug>.md`):
1. Read `{{WIKI_DIR}}/CLAUDE.md` and the planning node in full.
2. Show the user a summary: title, status, progress %, open tasks count, last session in `updated_by`.
3. Ask: "What do you want to update? (tasks / objective / context / status / decisions / other)"
4. Apply the changes in-place. For task updates, use `- [x]` / `- [ ]` syntax. For status, use the valid enum.
5. Recompute `progress` if tasks changed.
6. Append to `{{WIKI_DIR}}/LOG.md`:
   ```
   ## [YYYY-MM-DD HH:MM] update | <slug>
   - Changed: <what was updated>
   ```

**If the planning node does NOT exist** → CREATE:
1. Read `{{WIKI_DIR}}/CLAUDE.md` and `{{WIKI_DIR}}/INDEX.md`.
2. Ask the user (can ask all at once):
   - "What is the objective of this plan?"
   - "Which area does it belong to? ({{AREA_ENUM}})"
   - "List the initial tasks — you can refine them later."
3. Generate the planning node at `{{WIKI_DIR}}/plannings/<slug>.md` with:
   - `status: draft`, `progress: 0`, `created_by: null`, `updated_by: []`
4. Write the full body: populate `## Objective`, `## Context`, and `## Tasks` from the answers. Leave `## Progress` and `## Decisions` with *(none yet)*.
5. Update `{{WIKI_DIR}}/INDEX.md`: add `[[slug]] — one-liner` under `## Plannings > <area>`. Create the area section if it does not exist.
6. Append to `{{WIKI_DIR}}/LOG.md`:
   ```
   ## [YYYY-MM-DD HH:MM] plan | <slug>
   - Area: <area>
   - Tasks: <count> initial tasks
   ```
7. Report to the user: path created, remind to run `ingest` at end of sessions that advance this plan.

---

## LINT mode

Objective: report the health status of the wiki **WITHOUT modifying files**.

1. Read `{{WIKI_DIR}}/CLAUDE.md`, `{{WIKI_DIR}}/INDEX.md`, all files in `{{WIKI_DIR}}/sessions/`, and all files in `{{WIKI_DIR}}/plannings/`.
2. Check each category according to the lint rules in `{{WIKI_DIR}}/CLAUDE.md`.
3. Return a structured markdown report:

   ```markdown
   # Lint report — YYYY-MM-DD HH:MM

   ## Summary
   - Total nodes: N
   - Critical issues: N (broken wikilinks, invalid frontmatter, out-of-sync index)
   - Medium issues: N (orphans, missing cross-refs)
   - Opportunities: N (emerging concepts, stale claims)

   ## Critical
   ### Broken wikilinks
   - `[[non-existent-slug]]` in `2026-04-XX-node.md` → suggest correction

   ## Medium
   ### Orphan nodes
   - `[[slug]]` — no inbound links. Suggestion: cross-ref from [[related-node]].

   ## Opportunities
   ### Emerging concepts
   - "term" appears in 3 nodes. Candidate for `type: concept`.
   ```

4. **DO NOT modify files.**
5. Append to `{{WIKI_DIR}}/LOG.md`:
   ```
   ## [YYYY-MM-DD HH:MM] lint | report
   - Critical: N | Medium: N | Opportunities: N
   ```
