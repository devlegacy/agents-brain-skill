---
name: agents-brain
description: >
  Installs an LLM Wiki (interconnected markdown knowledge base, Karpathy/Obsidian style)
  in any project. Generates the wiki folder with an operational schema, index, log,
  and sources registry, plus the slash command (ingest | query | lint) and the Foam
  configuration for VSCode. Use this skill when the user says "install the brain",
  "I want an LLM wiki", "configure the memory system", "set up the brain", "mount the
  knowledge base", or when they ask for a cumulative knowledge system maintained by Claude.
---

# agents-brain

Interactive installer of an LLM Wiki for any project. Asks questions,
generates the entire structure adapted to the user's context, and installs the necessary
configuration so Claude can maintain the wiki with `/brain ingest`, `/brain query`,
and `/brain lint`.

The system is based on the Karpathy pattern: an interconnected markdown knowledge base
with Foam wikilinks, session nodes with YAML frontmatter, a dense index for the LLM,
an append-only log, and an external sources registry.

---

## Phase 1: Context collection

**Before creating any file**, ask the user the following questions. Do not assume answers — you need all of them to generate correctly.

You can ask the questions in a single conversational block (it is not necessary to wait for a response one by one, unless one answer drastically changes what you need to ask next).

### Required questions

**1. Wiki name (directory)**
> "What do you want to call the wiki directory? (default: `brain`, alternatives: `wiki`, `knowledge`, `memory`)"

**2. Slash command name**
> "What do you want to call the slash command? (default: same as the wiki name)"
> If the user chooses `memory` as the wiki name, suggest `memory` as the command as well — it is the pattern used in AustralisAI.

**3. Project name**
> "What is the name of your project or organization? (used in the command description)"

**4. Language of the node content**
> "What language will you use to write the wiki content?"
> Options: Spanish / English / Other (specify)

Depending on the language, the sections of each node are named differently:
- Spanish: Contexto, Decisiones, Output, Pendiente, Cross-refs, Fuentes
- English: Context, Decisions, Output, Pending, Cross-refs, Sources
- Other: the user defines the names

**5. Project areas**
> "What are the areas of your project? List the ones you need (e.g.: marketing, ops, dev, research, product)."
> These become the `area` enum of the frontmatter and sections of the index.

**6. Source of truth backend**
> "What tool do you use as the source of truth for the project's knowledge?"
> - **A — Notion**: the wiki stores pointers with short IDs; Claude fetches via MCP when it needs fresh content
> - **B — Local files**: the source of truth consists of files in the same repo (docs, specs, etc.)
> - **C — Another tool**: specify which one (Confluence, Linear, Airtable, etc.) and what instructions Claude should follow
> - **D — None**: standalone wiki, without an external source

**7. Do you have historical sessions to migrate?**
> "Are there files from previous sessions (logs, notes, changelogs) that you want to migrate to the wiki?"
> - Yes → we generate an immutability note in the schema and you can migrate manually later
> - No → the wiki starts clean

---

## Phase 2: File generation

With all the answers, generate the following files using the templates in `references/`.

### Variables to resolve before generating

Define these variables internally based on the responses:

```
WIKI_DIR        = answer Q1 (e.g.: "brain")
COMMAND_NAME    = answer Q2 (e.g.: "brain" or "memory")
PROJECT_NAME    = answer Q3 (e.g.: "Acme Corp")
AREA_ENUM       = areas separated by " | " (e.g.: "marketing | ops | dev")
SECTION_CONTEXT     = "Contexto" | "Context" | custom
SECTION_DECISIONS   = "Decisiones" | "Decisions" | custom
SECTION_OUTPUT      = "Output" (same in all languages)
SECTION_PENDING     = "Pendiente" | "Pending" | custom
SECTION_CROSS_REFS  = "Cross-refs" (same in all languages)
SECTION_SOURCES     = "Fuentes" | "Sources" | custom
BACKEND_TYPE    = A | B | C | D
HAS_LEGACY      = true | false
```

### Files to create

#### 1. `<WIKI_DIR>/CLAUDE.md`

Take the template from `references/brain-claude-md.md` and replace all placeholders:

- `{{WIKI_DIR}}` → WIKI_DIR
- `{{COMMAND_NAME}}` → COMMAND_NAME
- `{{AREA_ENUM}}` → AREA_ENUM
- `{{SECTION_*}}` → section names based on language
- `{{SOURCE_PREFIX}}` → based on backend:
  - A (Notion): `notion`
  - B (files): `repo`
  - C (other): the prefix that makes most sense for the tool
  - D (none): `repo`
- `{{BACKEND_QUERY_RULE}}` → based on backend:
  - A: "If any node cites a Notion source (via `SOURCES.md`) AND the question depends on fresh content, resolve it via `mcp__notion__notion-fetch` using the URL from `SOURCES.md`. **DO NOT answer from the cached summary in `SOURCES.md`** — that is only a directory."
  - B: "If the question depends on the current content of a repo file, read it directly with Read. The wiki stores pointers, not copies."
  - C: "If the question depends on fresh content from [tool name], query it directly. The wiki stores pointers, not copies of the content."
  - D: "Answer exclusively from the wiki nodes. If the content is not in the wiki, say so explicitly."
- `{{BACKEND_SECTION_TITLE}}` → based on backend:
  - A: "Notion Integration"
  - B: "Repo Files Integration"
  - C: "Integration with [tool name]"
  - D: *(omit this entire section)*
- `{{BACKEND_SECTION_BODY}}` → based on backend:
  - A: "Notion is the project's source of truth. The nodes reference Notion pages with short IDs defined in `{{WIKI_DIR}}/SOURCES.md`. Claude **must** call `mcp__notion__notion-fetch` with the corresponding URL when the query depends on fresh content from a page. The cached summary in `SOURCES.md` is only used to decide whether it is worth fetching."
  - B: "The repo files are the source of truth. The nodes reference files with `repo:<path>` in the frontmatter. When a query depends on the current content of a file, Claude reads it directly with Read. The wiki does not duplicate file content — it stores pointers."
  - C: "[Integration instructions provided by the user]"
  - D: *(omit)*
- `{{LEGACY_NOTE}}` → based on HAS_LEGACY:
  - true: "**NEVER** touch the legacy session files marked as 'immutable historical' in `{{WIKI_DIR}}/SOURCES.md`. They are frozen history. New sessions go ONLY to `{{WIKI_DIR}}/sessions/`."
  - false: *(empty string)*
- `{{LANGUAGE_RULE}}` → based on language:
  - Spanish: "- Node content: **Spanish**.\n- Tags and slugs: **Spanish without accents, kebab-case**.\n- Frontmatter keys: **English** (technical convention)."
  - English: "- Node content: **English**.\n- Tags and slugs: **English, kebab-case**.\n- Frontmatter keys: **English**."
  - Other: "- Node content: **[chosen language]**.\n- Tags and slugs: kebab-case.\n- Frontmatter keys: **English** (technical convention)."

#### 2. `<WIKI_DIR>/INDEX.md`

Generate directly (there is no separate template):

```markdown
# <WIKI_DIR> — Index

> Node catalog. One one-liner per node. Organized by type and then by area.
> To operate this wiki, read `CLAUDE.md` in this directory.
> When querying, read this file first to decide which nodes to open.

**Last updated:** <TODAY'S DATE>
**Total nodes:** 0 sessions | 0 concepts | 0 ADRs

---

## Sessions

<one H3 section per area in AREA_ENUM, in alphabetical order>
### <area-1>

*(empty)*

### <area-2>

*(empty)*

[... one per area ...]

---

## Concepts

*(empty — will emerge organically via `/<COMMAND_NAME> lint` when a term appears in 3+ nodes)*

## ADRs

*(empty)*
```

If the language is Spanish, the headers should be in Spanish: "Sesiones", "Conceptos".

#### 3. `<WIKI_DIR>/LOG.md`

```markdown
# <WIKI_DIR> — Operations Log

> Append-only. Most recent entries at the **end**.
> Format: `## [YYYY-MM-DD HH:MM] <operation> | <slug-or-target>`
> Valid operations: `bootstrap`, `ingest`, `update`, `rename`, `merge`, `split`, `lint`, `deprecate`.

---

## [<TODAY'S DATE> <CURRENT TIME>] bootstrap | wiki-initialized
- Created: CLAUDE.md, INDEX.md, LOG.md, SOURCES.md, sessions/
- Areas: <AREA_ENUM>
- Backend: <BACKEND_TYPE description>
- Generated by: skill agents-brain v1
```

#### 4. `<WIKI_DIR>/SOURCES.md`

Use the corresponding template from `references/sources-templates.md`:
- Backend A (Notion) → TEMPLATE A
- Backend B (files) → TEMPLATE B
- Backend C (other) → TEMPLATE D (filling in `{{CUSTOM_BACKEND_NAME}}` and instructions)
- Backend D (none) → TEMPLATE C

Replace `{{WIKI_DIR}}` with WIKI_DIR.

#### 5. `<WIKI_DIR>/sessions/.gitkeep`

Empty file so git tracks the empty directory:
```
```
*(completely empty file)*

#### 6. `.claude/commands/<COMMAND_NAME>.md`

Take the template from `references/brain-command.md` and replace:
- `{{WIKI_DIR}}` → WIKI_DIR
- `{{COMMAND_NAME}}` → COMMAND_NAME
- `{{PROJECT_NAME}}` → PROJECT_NAME
- `{{SECTION_DECISIONS}}`, `{{SECTION_OUTPUT}}`, `{{SECTION_PENDING}}`, `{{SECTION_CROSS_REFS}}` → section names based on language
- `{{BACKEND_QUERY_STEP}}` → same rule as `{{BACKEND_QUERY_RULE}}` in CLAUDE.md
- `{{LEGACY_INGEST_NOTE}}` → based on HAS_LEGACY:
  - true: "**NEVER** touch the legacy session files marked as immutable. They are frozen history."
  - false: *(empty string)*

#### 7. `.vscode/settings.json`

This step requires merge logic:

**If `.vscode/settings.json` does NOT exist:**
```json
{
  "foam.files.ignore": [
    "**/.claude/**",
    "**/node_modules/**"
  ]
}
```
(If HAS_LEGACY=true, also add the paths for the legacy folders the user mentioned.)

**If `.vscode/settings.json` ALREADY exists:**
1. Read the current file with Read
2. Check if `foam.files.ignore` exists in the JSON
   - If it exists: add the new paths to the existing array (without overwriting the ones already there)
   - If it does not exist: add the `foam.files.ignore` key with the new paths
3. Use Edit to make the surgical change (DO NOT overwrite the entire file)

Paths to add to the array:
- `**/.claude/**`
- `**/node_modules/**`
- If HAS_LEGACY=true, for each legacy sessions folder the user mentions: `**/<folder>/sessions/**`

---

## Phase 3: Setup summary

After creating all files, show this summary to the user:

```
## agents-brain installed

Wiki at: <WIKI_DIR>/
Command: /<COMMAND_NAME> (ingest | query | lint)

### Generated files
<WIKI_DIR>/CLAUDE.md          ← open this to see the wiki rules
<WIKI_DIR>/INDEX.md           ← node catalog (empty for now)
<WIKI_DIR>/LOG.md             ← operations log
<WIKI_DIR>/SOURCES.md         ← external sources registry
<WIKI_DIR>/sessions/          ← this is where nodes will live
.claude/commands/<COMMAND_NAME>.md  ← slash command installed
.vscode/settings.json         ← Foam configuration (created or updated)

### Next steps
1. Install the Foam extension in VSCode (if you don't have it):
   Marketplace: "Foam" by Foam team, or search foam-vscode
<IF BACKEND=A>
2. Fill in brain/SOURCES.md with the URLs of your Notion pages
3. Make sure you have the Notion MCP configured in .mcp.json
</IF BACKEND=A>
<IF HAS_LEGACY=true>
2. You can migrate historical sessions manually: copy the content, add
   the YAML frontmatter, and save to <WIKI_DIR>/sessions/YYYY-MM-DD-slug.md
</IF HAS_LEGACY>
4. At the end of your first work session, run `/<COMMAND_NAME> ingest`
```

---

## Notes for the LLM

- The templates in `references/` use double-brace placeholders `{{PLACEHOLDER}}`. Replace all of them before writing the files.
- Do not leave any placeholder unreplaced in the generated files.
- If the user gives names in uppercase or with spaces for the areas, convert them to lowercase kebab-case for the frontmatter enum (e.g.: "Digital Marketing" → `digital-marketing`), but show the original name in the index.
- The `.vscode/settings.json` file is the only one that requires merge logic — all others are created fresh.
- If `.claude/commands/` does not exist, create it as well.
- Verify before creating that `<WIKI_DIR>/` does not already exist to avoid overwriting an existing wiki. If it exists, notify the user and ask if they want to continue.
