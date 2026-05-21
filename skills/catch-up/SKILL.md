---
name: catch-up
description: >
  Orchestrates "process the day" by discovering and processing unprocessed items
  across all input sources: daily fleeting note, Drive transcripts (Gemini Meet
  captures saved by /transcricoes), and Granola meetings (via MCP). Dedupes
  Drive vs Granola for the same meeting (Drive wins), and routes everything via
  /bedrock:preserve in a single coordinated run.
  Use when: "bedrock catch-up", "bedrock-catch-up", "/bedrock:catch-up",
  "processa o dia", "processar o dia", "catch-up", "processar tudo do dia".
user_invocable: true
allowed-tools: Bash, Read, Write, Edit, Glob, Grep, Skill, Agent, ToolSearch, mcp__granola__*
---

# /bedrock:catch-up — Day Catch-up Orchestrator

## Plugin Paths

Entity definitions and templates are in the plugin directory, not the vault root.
Use the "Base directory for this skill" provided at invocation to resolve paths:

- Entity definitions: `<base_dir>/../../entities/`
- Templates: `<base_dir>/../../templates/{type}/_template.md`
- Plugin CLAUDE.md: `<base_dir>/../../CLAUDE.md` (auto-injected into context)

---

## Vault Resolution

Same resolution chain as other skills:

1. `--vault <name>` flag → registry lookup at `<base_dir>/../../vaults.json`
2. CWD detection → registered vault path match
3. Default vault from registry
4. Error if no vault found

Set `VAULT_PATH` accordingly.

---

## Overview

This skill is a **discovery + orchestration** layer. It does NOT reimplement
classification or routing — it discovers unprocessed sources for a given date,
presents a plan, and delegates each source to the appropriate sub-skill or to
`/bedrock:preserve` directly.

**You are an orchestration agent.** Follow phases below in order. Do NOT skip
the planning phase: always show the user what was found before executing.

---

## Phase 0 — Parse Input

Parse the user's input to determine the target date:

| Input | Resolution |
|---|---|
| (empty) or "hoje" | Today's date |
| "ontem" | Yesterday |
| `YYYY-MM-DD` | Use literally |
| Relative like "anteontem" or "-N" | Compute from today |

Set `DATE` to resolved value in `YYYY-MM-DD` format.

---

## Phase 1 — Discovery

Run all three discovery steps in parallel (they are independent).

### 1.1 — Daily fleeting

Check whether `<VAULT_PATH>/fleeting/<DATE>-daily.md` exists.

If yes, read its frontmatter:
- `status: raw` (or `reviewing`) → mark `daily.state = "raw"`, capture path
- `status: promoted` → mark `daily.state = "already_processed"`
- Other / no status → treat as raw

If file does not exist → `daily.state = "missing"`.

### 1.2 — Drive transcripts

Read vault config:
```bash
cat <VAULT_PATH>/.bedrock/config.json
```

Field `transcripts_dir`. If absent or empty → skip Drive source entirely
(`drive.state = "not_configured"`).

If configured, scan that directory for `*.md` files. For each file with
frontmatter `data: <DATE>`, check the `processed_by_dump` field:

```bash
# Cheap membership check via shell (do NOT cat files into context)
for f in "<transcripts_dir>"/*.md; do
  data=$(grep -m1 '^data:' "$f" | awk '{print $2}')
  if [ "$data" = "<DATE>" ]; then
    if grep -q '^processed_by_dump: true' "$f"; then
      echo "PROCESSED|$f"
    elif grep -q '^skip: true' "$f"; then
      echo "SKIP|$f"
    else
      echo "PENDING|$f"
    fi
  fi
done
```

Build `drive_pending`: list of `{path, hora_inicio, doc_id, titulo}` for each
PENDING file. Parse `hora-inicio` from frontmatter and the original meeting
title from the body's first heading.

### 1.3 — Granola meetings

Check Granola MCP availability:

```
ToolSearch(query: "select:mcp__granola__list_meetings,mcp__granola__get_meetings", max_results: 3)
```

If tools NOT found → `granola.state = "not_available"`.

If found, call:
```
mcp__granola__list_meetings(time_range: "this_week")
```

Filter results to meetings whose start date matches `<DATE>`.

Cross-check against `<VAULT_PATH>/granola-processados.md` using shell grep
(never load the file into context):

```bash
# For each candidate ID
grep -qF "$ID" <VAULT_PATH>/granola-processados.md && echo "ALREADY" || echo "PENDING"
```

Build `granola_pending`: list of `{id, hora_inicio, titulo}` for each PENDING meeting.

### 1.4 — Dedupe Drive ↔ Granola

For each meeting in `granola_pending`, check if `drive_pending` has an entry
with the same `data` and `hora_inicio` within ±5 minutes.

If yes → remove from `granola_pending` (mark as `deduped_by_drive`).

**Drive wins.** Granola is fallback only for meetings without a Drive capture.

---

## Phase 2 — Present Plan

Show the user a summary BEFORE executing:

```
Catch-up para <DATE>:

  Daily fleeting:
    <state and path, or "missing" or "already processed">

  Drive transcripts (transcripts_dir):
    <N pending> | <M already processed> | <K skip>
    [list pending with hora and title, if any]

  Granola meetings:
    <N pending> | <M deduped by Drive> | <K already processed>
    [list pending with hora and title, if any]
    [list deduped entries for transparency]

Vou processar nesta ordem: Drive → Granola (resto) → Daily.

Confirmar? (sim / pular X / cancelar)
```

Wait for confirmation. The user may:
- "sim" → proceed
- "pular drive" / "pular daily" / "pular granola" → exclude a source
- "cancelar" → stop

---

## Phase 3 — Load Vault Context (once, shared across sub-processes)

Load the alias map and entity lookups once. They will be reused by all sub-steps.

1. **People with management_role:** Glob `<VAULT_PATH>/people/*.md`, read frontmatter, build:
   `{filename, name, aliases[], management_role, team}`.
2. **Teams:** Glob `<VAULT_PATH>/teams/*.md`, read frontmatter.
3. **Active projects:** Glob `<VAULT_PATH>/projects/*.md` where `status != "completed"`.
4. **Alias map:** case-insensitive: alias → person_filename.

---

## Phase 4 — Execute

### 4.1 — Drive transcripts

For each file in `drive_pending` (chronological order by `hora-inicio`):

1. **Read the file** — full body has `# {original title}` then plain-text export
   (Observações + Transcrição sections).

2. **Detect 1:1 vs group** — same heuristics as `/bedrock:granola` Phase 4:
   - Parse participants from the Observações section (Gemini lists them) and
     from the title (e.g., "Fulano / Monteiro" → 2 participants).
   - If exactly 2 known participants and the other is a managed person
     (`direct-report` / `indirect-report`) OR title pattern `1:1`,
     `X <> Y`, `X / Y` → **1:1**.
   - Otherwise → **group**.

3. **Classify content** — extract from Observações (Gemini already produces a
   summary + action items):
   - Action items for managed people → `proximo_1_1`
   - Action items for the vault owner → owner's to-do
   - Observations about people (1:1 only) → person `log`
   - Team decisions → team `decisao`
   - Project updates → project body

4. **Build /bedrock:preserve input:**
   - For 1:1: `person` updates (log + proximo_1_1)
   - For group: `discussion` create entity + routed people/team/project updates
   - Discussion slug: lowercase, no accents, spaces→hyphens, from title (max 50 chars)

5. **Delegate to `/bedrock:preserve`** via Skill tool.

6. **Mark file as processed** — append to its frontmatter (or edit in place):
   ```yaml
   processed_by_dump: true
   processed_by_dump_at: <today YYYY-MM-DD>
   ```

   Use `Edit` to flip `processed_by_dump: false` → `processed_by_dump: true`
   and add `processed_by_dump_at`. Never overwrite the body.

### 4.2 — Granola (only meetings NOT deduped)

If `granola_pending` is non-empty AND user did not skip Granola:

Invoke `/bedrock:granola` via Skill tool. The granola skill will fetch its own
unprocessed list (which already excludes IDs in `granola-processados.md`) and
process them. Our dedupe (Phase 1.4) acted only as a planning aid; the actual
re-processing protection lives in granola-processados.md.

**Important:** if Phase 1.4 marked specific IDs as `deduped_by_drive`, append
them to `granola-processados.md` BEFORE invoking `/bedrock:granola`, so that
skill skips them:

```bash
for ID in <deduped_ids>; do
  echo "$ID" >> <VAULT_PATH>/granola-processados.md
done
```

This is critical to avoid double-processing.

### 4.3 — Daily fleeting

If `daily.state == "raw"`:

Invoke `/bedrock:dump` via Skill tool with the resolved daily file path. The
dump skill handles its own classification, routing, and `status: promoted`
marking.

If `daily.state == "missing"` or `"already_processed"`: skip silently (mention
in summary).

---

## Phase 5 — Aggregate Summary

After all sub-steps complete, combine their summaries:

```
Catch-up <DATE> concluído.

Drive (N transcrições):
  - <titulo curto> (1:1 com Fulano) → log + 2 items próximo 1:1
  - <titulo curto> (grupo) → discussion + 3 routings

Granola (M reuniões processadas, K deduped pelo Drive):
  - <delegado pra /bedrock:granola, sumário abaixo>

Daily fleeting:
  - <delegado pra /bedrock:dump, sumário abaixo>

Distribuição total:
  - X itens → Próximo 1:1
  - Y observações → Log
  - Z decisões → Teams
  - W updates → Projects
  - ⚠️ N incertos
```

---

## Phase 6 — Offer Behavioral Analysis

After Phase 5 summary, identify Drive 1:1 transcripts processed in this run that
are candidates for `/bedrock:analyze-call`. Granola 1:1s are NOT considered here —
the granola skill has its own offer mechanism that fires after `/bedrock:granola`
completes; do not double-offer.

### 6.1 — Build candidate list

From Drive transcripts processed in Phase 4.1, select files classified as `1:1`.
For each, look up the OTHER participant (not the vault owner) in `<VAULT_PATH>/people/`:

| Other participant's `management_role` | Default selection | Show as candidate? |
|---|---|---|
| `direct-report` | **default-on** | yes |
| `indirect-report` | **default-on** | yes |
| `peer` | default-off | yes (user can opt-in) |
| `leader` | default-off | yes (analyzing being-coached dynamics) |
| `external`, `self`, `offboarded` | n/a | skip |

For each candidate, enrich with prior-analysis context (cheap shell grep, do NOT
load Log into context):

```bash
# Count of prior [análise] entries (≥0)
grep -c '^### .* — \[análise\]' <VAULT_PATH>/people/<slug>.md 2>/dev/null

# Date of most recent prior analysis (or "nenhuma")
grep -m1 '^### .* — \[análise\]' <VAULT_PATH>/people/<slug>.md | awk '{print $2}' 2>/dev/null
```

### 6.2 — Present batched offer

Show ONE consolidated question (never per-file). Format:

```
N transcrições processadas, M são 1:1 elegíveis pra análise comportamental:
  [x] <Nome> (<role>) — última análise: <YYYY-MM-DD ou "nenhuma">, <N> entries de [análise]
  [ ] <Nome> (peer) — última análise: nenhuma
  ...

Rodar /bedrock:analyze-call em quais? (todas / nenhuma / lista de números, ou "x N" pra alternar)
```

### 6.3 — Execute (if user confirms)

For each selected candidate, invoke `/bedrock:analyze-call` via Skill tool with the
transcript file path as input. analyze-call handles its own write (Log entry with
`<!-- private: coaching -->` marker) and the cross-call pattern detection (≥2
ocorrências → propor tema em `## Temas em Acompanhamento`).

If user replies "nenhuma" or empty: skip silently, no error.

### 6.4 — Group meetings: never auto-offer

Group meetings (3+ participants) are NEVER offered for analyze-call automatically.
analyze-call on groups has limited value (everyone but vault owner is "Them" in
Granola; in Drive transcripts behavioral signal dilutes across many speakers).
User can run `/bedrock:analyze-call` manually on a specific group transcript when
they explicitly want.

---

## Rules

1. **Discovery is read-only.** Phase 1 never writes — it only inventories.
2. **Plan before execute.** Always present Phase 2 and wait for confirmation,
   unless user explicitly bypassed via flag (not currently supported).
3. **Drive > Granola for dedupe.** ±5 min window on `data + hora-inicio`.
4. **Idempotent.** Re-running on the same day must not re-process anything:
   - Drive: respects `processed_by_dump: true` and `skip: true`
   - Granola: respects `granola-processados.md` (deduped IDs added before delegation)
   - Daily: respects `status: promoted`
5. **Missing source ≠ error.** If `transcripts_dir` is not configured, or no
   daily exists, or Granola MCP is offline — skip that source, continue with
   the rest. Report what was skipped and why.
6. **Membership checks via shell, not context.** Use `grep -qF` to test
   processed-IDs and frontmatter flags. Never `cat` registries into the prompt.
7. **Language** — use the vault's configured language for all output.
