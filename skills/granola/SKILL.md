---
name: granola
description: >
  Ingest meeting transcriptions from Granola via MCP. Fetches unprocessed meetings,
  classifies each (1:1 with direct report vs group meeting vs unknown), routes action
  items to people and projects via /bedrock:preserve, creates discussion entities for
  group meetings, and tracks processed IDs to avoid duplication.
  Use when: "bedrock granola", "bedrock-granola", "/bedrock:granola",
  "processar as transcrições do granola", "processar granola", "processe o granola",
  or when the user asks to process meeting transcriptions.
user_invocable: true
allowed-tools: Bash, Read, Write, Edit, Glob, Grep, Skill, Agent, ToolSearch, mcp__granola__*
---

# /bedrock:granola — Granola Meeting Ingestion

## Plugin Paths

Entity definitions and templates are in the plugin directory, not the vault root.
Use the "Base directory for this skill" provided at invocation to resolve paths:

- Entity definitions: `<base_dir>/../../entities/`
- Templates: `<base_dir>/../../templates/{type}/_template.md`
- Plugin CLAUDE.md: `<base_dir>/../../CLAUDE.md` (auto-injected into context)

Where `<base_dir>` is the path shown in "Base directory for this skill".

---

## Vault Resolution

Resolve which vault to operate on. Same resolution chain as other skills:

1. `--vault <name>` flag → registry lookup
2. CWD detection → registered vault path match
3. Default vault from registry
4. Error if no vault found

---

## Overview

This skill ingests meeting notes from the Granola app via its MCP server. It fetches
meetings not yet processed, classifies each one (1:1 with managed person vs group
meeting vs unknown), routes observations and action items to the correct entities,
and creates discussion entities for group meetings.

**You are a meeting ingestion agent.** Follow the phases below in order.

---

## Phase 0 — Check MCP Availability

Verify that the Granola MCP tools are available:

```
ToolSearch(query: "select:mcp__granola__list_meetings,mcp__granola__get_meetings", max_results: 3)
```

**If tools are NOT found:** respond with a graceful error:

> "Granola MCP server is not available. Make sure the Granola MCP is connected
> (check with `/mcp`). Skipping meeting ingestion."

Stop execution. Do NOT attempt to call Granola tools if they are not available.

**If tools ARE found:** proceed to Phase 1.

---

## Phase 1 — Ensure Registry Exists

Make sure the processed-IDs registry file exists. **Do NOT load its contents
into context.** Membership checks happen later via `grep` (Phase 2.2).

```bash
REGISTRY="<VAULT_PATH>/granola-processados.md"
if [ ! -f "$REGISTRY" ]; then
  cat > "$REGISTRY" <<'HEAD'
# Granola — Reuniões Processadas

Lista de IDs de reuniões do Granola já processadas no vault.
Um ID por linha. Não editar manualmente.

HEAD
fi
```

**Why grep instead of cat-into-context:** the file grows unbounded over time
(potentially thousands of IDs after a year). Loading it into the LLM context
inflates token usage with no benefit, and slicing the read with `head`/`tail`
risks missing IDs that happen to land outside the slice (this caused a bug
on 2026-05-05). `grep -qF` is O(file size) on disk but O(1) on context.

---

## Phase 2 — Fetch Meetings

### 2.1 List recent meetings

Call the Granola MCP to list meetings:

```
mcp__granola__list_meetings(time_range: "this_week")
```

If `this_week` returns 0 meetings, also try:

```
mcp__granola__list_meetings(time_range: "last_week")
```

### 2.2 Filter unprocessed (membership via grep)

For each meeting ID returned by `list_meetings`, run a fixed-string grep
against the registry. **Do not parse the file into the LLM context.** The
shell does the membership check; only the result (processed/unprocessed) is
exposed to the model.

```bash
for id in <id1> <id2> ... <idN>; do
  if grep -qF "$id" "$REGISTRY"; then
    echo "skip $id"
  else
    echo "process $id"
  fi
done
```

Build the list `UNPROCESSED_IDS` from the IDs that printed `process`.

If `UNPROCESSED_IDS` is empty: respond "Nenhuma reunião nova para processar." and stop.

### 2.3 Fetch details

For the unprocessed meetings (max 10 at a time):

```
mcp__granola__get_meetings(meeting_ids: [<id1>, <id2>, ...])
```

---

## Phase 3 — Load Vault Context

Same as `/bedrock:dump` Phase 2 — load people, teams, and projects for classification:

1. **People with management_role:** Glob `<VAULT_PATH>/people/*.md`, build lookup with aliases.
2. **Teams:** Glob `<VAULT_PATH>/teams/*.md`, build lookup.
3. **Active projects:** Glob `<VAULT_PATH>/projects/*.md` where `status != "completed"`.
4. **Alias resolution map:** From all people, case-insensitive.

---

## Phase 4 — Classify Each Meeting

For each unprocessed meeting, determine its type based on participants and title:

### 4.1 Detect 1:1 with managed person

A meeting is a **1:1 with a managed person** when:
- Exactly 2 known participants (vault owner + one other)
- The other participant matches a person with `management_role: "direct-report"` or `"indirect-report"`
- OR the title matches patterns like "1:1", "Fulano <> Owner", "Owner / Fulano", "Fulano / Owner"

**When detected as 1:1:**
- Do NOT create a discussion entity
- Route observations → person **Log** (with `### YYYY-MM-DD` header)
- Route action items for the person → person **Próximo 1:1**
- Route action items for the vault owner → owner's to-do system

### 4.2 Detect group meeting

A meeting is a **group meeting** when:
- 3+ participants, OR
- Not detected as 1:1

**When detected as group meeting:**
- Create a **discussion** entity in `<VAULT_PATH>/discussions/YYYY-MM-DD-<slug>.md`
- Extract: participants, conclusions, action_items (with owner resolution)
- Route action items to people's **Próximo 1:1** when the owner is a managed person
- Route project updates to the relevant project entity
- Route team decisions to the relevant team entity

### 4.3 Unknown meeting

If a meeting has participants not found in the vault AND the purpose is unclear:
- Collect for a grouped question (same pattern as `/bedrock:dump`)

---

## Phase 5 — Execute via /bedrock:preserve

### For 1:1 meetings:

Build structured input for `/bedrock:preserve`:

```yaml
entities:
  - type: person
    name: "<person name>"
    action: update
    route_to: log
    content: "<all observations from meeting summary>"
    date: "YYYY-MM-DD"
  - type: person
    name: "<person name>"
    action: update
    route_to: proximo_1_1
    content: "<action item description>"
    date: "YYYY-MM-DD"
```

### For group meetings:

Build structured input including a new discussion entity + routing:

```yaml
entities:
  - type: discussion
    name: "YYYY-MM-DD-<slug>"
    action: create
    content:
      title: "<meeting title>"
      date: "YYYY-MM-DD"
      summary: "<meeting summary>"
      participants: ["[[person1]]", "[[person2]]"]
      conclusions: ["<conclusion 1>", "<conclusion 2>"]
      action_items:
        - description: "<action>"
          owner: "[[person-slug]]"
          status: "todo"
          deadline: ""
          routed_to: "[[person-slug]]"
      related_topics: []
      related_actors: []
      related_projects: ["[[project-slug]]"]
      related_teams: ["[[team-slug]]"]
  - type: person
    name: "<managed person>"
    action: update
    route_to: proximo_1_1
    content: "<action item for this person>"
    date: "YYYY-MM-DD"
  - type: team
    name: "<team name>"
    action: update
    route_to: decisao
    content:
      decision: "<decision text>"
      context: "<context>"
    date: "YYYY-MM-DD"
  - type: project
    name: "<project name>"
    action: update
    content: "<project update>"
    date: "YYYY-MM-DD"
```

Delegate to `/bedrock:preserve`.

---

## Phase 5.5 — Offer Behavioral Analysis (analyze-call hook)

After ingestion completes (Phase 5), identify which processed meetings qualify
for behavioral analysis via `/bedrock:analyze-call`. The goal is coaching:
extract patterns from 1:1s with managed people that the standard summary
misses.

### 5.5.1 Eligibility filter

For each processed meeting, classify into one of three buckets:

| Bucket | Criteria |
|---|---|
| **Auto-eligible** (default: include) | 1:1 with a person whose `management_role` is `direct-report` or `indirect-report` |
| **Optional** (default: exclude, ask) | 1:1 with a person whose `management_role` is `peer` (e.g., Mac) |
| **Skip** | Group meetings (Granola can't distinguish speakers in groups), 1:1s with external people, meetings with no transcript or summary too short (<2000 words estimated) |

### 5.5.2 Present batch question

If at least one meeting is auto-eligible OR optional, ask the user **once**:

```
Análise comportamental disponível para {N} reunião(ões):

  Auto-elegíveis (1:1 com liderado):
    - "{título}" → {pessoa} ({direct-report|indirect-report})
    - ...

  Opcionais (1:1 com peer):
    - "{título}" → {pessoa} (peer)

Rodar análise comportamental?
  [s] Sim — todas auto-elegíveis (default)
  [t] Sim — todas, incluindo peers
  [n] Não — pular análise nesta rodada
  [c] Customizar — escolher uma a uma
```

If zero meetings are auto-eligible AND zero are optional: skip Phase 5.5
silently (don't ask anything, just proceed to Phase 6).

### 5.5.3 Invoke analyze-call per selected meeting

For each meeting selected by the user, invoke `/bedrock:analyze-call`:

```
/bedrock:analyze-call granola <meeting_uuid>
```

Each invocation runs in **interactive mode** (its own confirmation flow per
person/log entry). Do NOT bypass `/bedrock:analyze-call`'s own guard-rails —
the user still confirms each Log entry. The Phase 5.5 batch question is only
about *whether to disparar* the analysis, not about silently persisting.

After all selected analyses complete (or are declined), proceed to Phase 6.

### 5.5.4 Hard rules

1. **Never auto-disparar in batch without the user's explicit confirmation.**
   Even with all-auto-eligible default selected, the question must be asked.
2. **Group meetings are always skipped** — Granola "Me/Them" attribution
   collapses in groups. The user can run `/bedrock:analyze-call gdoc <url>`
   manually with a Meet transcript if needed.
3. **Privacy continues:** behavioral analysis output goes to the person's Log
   with `[análise]` prefix and `<!-- private: coaching -->` marker. The
   `/bedrock:analyze-call` skill enforces this; granola only invokes.
4. **Cost awareness:** if more than 5 meetings are eligible at once, warn the
   user about token cost before proceeding ("Isso vai ler {N} transcripts
   completos. Confirmar?").

---

## Phase 6 — Register Processed IDs

After successfully processing each meeting, append its ID to the registry —
**but only if not already there** (defensive against re-processing scenarios):

```bash
id="<meeting_uuid>"
grep -qF "$id" "$REGISTRY" || echo "$id" >> "$REGISTRY"
```

One ID per line. Append only — never remove or rewrite the file. The
`grep -qF || echo` pattern keeps the file deduplicated without rewriting it.

---

## Phase 7 — Summary

Present a summary:

```
Processado! X reuniões do Granola:

Reuniões processadas:
- "Meeting Title 1" (1:1 com Fulano) → Log + 2 items Próximo 1:1
- "Meeting Title 2" (grupo, 5 participantes) → discussion + 3 items roteados
- "Meeting Title 3" (desconhecida) → ⚠️ perguntei acima

Distribuição:
- N itens → Próximo 1:1 (Pessoa1: X, Pessoa2: Y)
- N observações → Log
- N discussions criadas
- N decisões → Teams
- N updates → Projetos

Análise comportamental:
- N reunião(ões) elegível(is) → analisada(s) via /bedrock:analyze-call
- N pulada(s) (grupo / peer / sem transcript)
```

---

## Rules

1. **1:1 meetings do NOT create discussions** — they route directly to the person's Log and Próximo 1:1.
2. **Alias resolution is mandatory** — resolve all participant names before routing.
3. **Never reprocess** — always check `granola-processados.md` before processing.
4. **Group all uncertainties** into one question (unknown participants, unclear purpose).
5. **Graceful degradation** — if Granola MCP is unavailable, inform the user and stop. Never error out.
6. **Language** — use the vault's configured language for all output and entity content.
7. **Discussion slug** — derive from meeting title: lowercase, no accents, spaces→hyphens, max 50 chars.
8. **Behavioral analysis hook (Phase 5.5)** — after ingestion, offer to run
   `/bedrock:analyze-call` on eligible 1:1s (direct-report / indirect-report
   default-on; peer default-off). Group meetings are always skipped
   (Granola speaker attribution collapses). Always ask before dispatching;
   never silently auto-disparar.
