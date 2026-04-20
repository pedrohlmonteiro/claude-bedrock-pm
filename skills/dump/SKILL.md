---
name: dump
description: >
  Process daily dumps and quick notes. Reads a fleeting note (or free-form text),
  classifies each item, resolves person aliases, routes to the correct entity
  (person 1:1, person log, team decision, project update, etc.) via /bedrock:preserve,
  marks the fleeting as promoted, and shows a summary.
  Use when: "bedrock dump", "bedrock-dump", "/bedrock:dump", "processar o daily",
  "processar", "processa", "dump", "processar o quick dump",
  or when the user sends a quick inline note like "cobrar X do Fulano".
user_invocable: true
allowed-tools: Bash, Read, Write, Edit, Glob, Grep, Skill, Agent
---

# /bedrock:dump — Daily Dump Processor

## Plugin Paths

Entity definitions and templates are in the plugin directory, not the vault root.
Use the "Base directory for this skill" provided at invocation to resolve paths:

- Entity definitions: `<base_dir>/../../entities/`
- Templates: `<base_dir>/../../templates/{type}/_template.md`
- Plugin CLAUDE.md: `<base_dir>/../../CLAUDE.md` (auto-injected into context)

Where `<base_dir>` is the path shown in "Base directory for this skill".

---

## Vault Resolution

Resolve which vault to operate on. This skill can be invoked from any directory.

**Step 1 — Parse `--vault` flag:**
Check if the input arguments include `--vault <name>`. If found, extract the vault name and remove it from the arguments.

**Step 2 — Resolve vault path:**

1. **If `--vault <name>` was provided:**
   Read the vault registry at `<base_dir>/../../vaults.json`. Find the entry matching the name.
   If not found: error — "Vault `<name>` is not registered. Run `/bedrock:vaults` to see available vaults."
   If found: set `VAULT_PATH` to the entry's `path` value.

2. **If no `--vault` flag — CWD detection:**
   Read `<base_dir>/../../vaults.json`. Check if the current working directory is inside any registered vault path.
   If found: set `VAULT_PATH` to the matching vault's `path`.

3. **If CWD detection fails — default vault:**
   From the registry, find the vault with `"default": true`.
   If found: set `VAULT_PATH` to the default vault's `path`.

4. **If no vault resolved:** error — "No vault found. Run `/bedrock:setup` to initialize one."

---

## Overview

This skill processes daily dumps — raw text captures that need to be classified and
routed to the correct entities in the vault. It also handles quick inline notes
(e.g., "cobrar X do Fulano") without requiring a formal fleeting note.

**You are a classification and routing agent.** Follow the phases below in order.

---

## Phase 0 — Detect Input Mode

Parse the user's input to determine the mode:

| Input pattern | Mode | Description |
|---|---|---|
| Path to a fleeting note (e.g., `fleeting/2026-04-14-daily.md`) | **file** | Process a specific fleeting note |
| "processar o daily", "processa", "dump" (no specific file) | **latest** | Find the most recent unprocessed fleeting note |
| "processar o quick dump" | **quickdump** | Process a designated quick dump file |
| Free-form text (e.g., "cobrar X do Fulano") | **inline** | Route a single item directly |

---

## Phase 1 — Load Content

### Mode: file
Read the specified fleeting note from `<VAULT_PATH>/fleeting/<filename>`.

### Mode: latest
Find the most recent fleeting note with `status: raw` in frontmatter:

```bash
# List fleeting notes sorted by date (newest first)
ls -t <VAULT_PATH>/fleeting/*.md 2>/dev/null
```

Read each file's frontmatter until finding one with `status: raw` (or `status: reviewing`).
If none found: respond "No unprocessed fleeting notes found." and stop.

### Mode: quickdump
Read `<VAULT_PATH>/fleeting/quick-dump.md` (or the configured quick dump path).
After processing, **remove** each processed item from the file (quick dump is permanent, never archived).

### Mode: inline
Use the user's text directly as a single item. Skip to Phase 2.

---

## Phase 2 — Load Context

Load the vault context needed for classification and alias resolution:

1. **People with management_role:** Glob `<VAULT_PATH>/people/*.md`, read frontmatter of each.
   Build a lookup table: `{filename, name, aliases[], management_role, team}`.

2. **Teams:** Glob `<VAULT_PATH>/teams/*.md`, read frontmatter.
   Build: `{filename, name, aliases[]}`.

3. **Active projects:** Glob `<VAULT_PATH>/projects/*.md`, read frontmatter where `status != "completed"`.
   Build: `{filename, name, aliases[]}`.

4. **Alias resolution map:** From all people, build a case-insensitive map:
   `alias → person_filename`. Include: name, all aliases, filename stem.
   Example: "Duff" → "pedro-duff", "Carol" → "carolina-lindenberg", "Cannuto" → "rodrigo-cannuto".

---

## Phase 3 — Classify Each Item

For each item in the content (line, paragraph, or bullet point), classify it:

| Item pattern | Route | Target |
|---|---|---|
| "cobrar X do [pessoa]", "levar pro 1:1 de [pessoa]", "pedir [pessoa]" | `proximo_1_1` | Person → Próximo 1:1 |
| Observation about a person (behavior, feedback, context) | `log` | Person → Log |
| Recurring theme about a person | `tema` | Person → Temas em Acompanhamento |
| Development/career note about a person | `pdi` | Person → Desenvolvimento / PDI |
| Competency score (e.g., "Fulano: delivery 4, quality 3") | `competencia` | Person → frontmatter |
| Decision by a tribe/squad | `decisao` | Team → Decisões Importantes |
| Strategic theme for a tribe | `tema_estrategico` | Team → Temas Estratégicos |
| Project update | `project_update` | Project → body (append) |
| Personal action item for the vault owner | `todo` | Fleeting with tag `to-do` (or user's task system) |
| Meeting reference | `discussion` | Create discussion entity |
| **Cannot identify** | `uncertain` | Collect for grouped question |

### Alias Resolution

When a person is mentioned, resolve using the alias map from Phase 2:
- Case-insensitive matching
- Partial match for names (e.g., "Duff" matches "Pedro Duff")
- If multiple matches: prefer exact alias match over partial name match
- If no match: mark as `uncertain`

### Grouping Uncertainties

Collect ALL uncertain items. Do NOT ask about each one individually.
After classifying all items, present ONE grouped question:

```
Não consegui identificar o destino destes itens:
1. "item A" — pessoa desconhecida "Fulano"
2. "item B" — pode ser projeto ou decisão de tribo
3. "item C" — tribo/squad incerto

Para onde devem ir?
```

Wait for user response before proceeding. Then re-classify based on the answer.

---

## Phase 4 — Route via /bedrock:preserve

Build a structured input for `/bedrock:preserve` with all classified items:

```yaml
entities:
  - type: person
    name: "<resolved person name>"
    action: update
    route_to: proximo_1_1
    content: "<item text>"
    date: "YYYY-MM-DD"
  - type: person
    name: "<resolved person name>"
    action: update
    route_to: log
    content: "<observation text>"
    date: "YYYY-MM-DD"
  - type: team
    name: "<resolved team name>"
    action: update
    route_to: decisao
    content:
      decision: "<decision text>"
      context: "<context>"
    date: "YYYY-MM-DD"
  - type: project
    name: "<resolved project name>"
    action: update
    content: "<update text>"
    date: "YYYY-MM-DD"
```

Delegate to `/bedrock:preserve` with this structured input.

---

## Phase 5 — Mark as Processed

### Mode: file or latest
Update the fleeting note's frontmatter:
- Set `status: "promoted"`
- Set `updated_at` to today
- Set `updated_by` to `"dump@agent"`

### Mode: quickdump
Remove each processed item from the quick dump file. Leave unprocessed items intact.

### Mode: inline
No fleeting to update — skip.

---

## Phase 6 — Summary

Present a summary to the user:

```
Processado! Distribuí X itens:
- N itens → Próximo 1:1 (Pessoa1: X, Pessoa2: Y)
- N observações → Log (Pessoa1: X)
- N decisões → Teams (Team1: X)
- N updates → Projetos (Projeto1: X)
- N ações pessoais → To-do
- ⚠️ N itens não identificados (perguntei acima)
```

---

## Rules

1. **NEVER ask about each uncertain item individually** — always group into one question.
2. **Resolve aliases before routing** — "Duff" must become `[[pedro-duff]]`, not left as text.
3. **Date format:** always use the date from the fleeting note (or today for inline mode).
4. **Append-only:** never overwrite existing content in target entities.
5. **Language:** use the vault's configured language for all output.
6. **If the entire input is a single quick note** (e.g., "cobrar X do Duff"): skip the summary and just confirm the routing.
