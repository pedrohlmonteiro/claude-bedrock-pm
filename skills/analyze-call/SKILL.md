---
name: analyze-call
description: >
  Behavioral analysis of a single meeting/call. Extracts conversational dynamics
  (turn-taking, interruptions, % of speech), linguistic patterns per speaker,
  observable competence signals, and suggested next-1:1 follow-ups. Output is
  curated observations appended to the Log of the relevant person notes — never
  raw transcript. Default mode is interactive (always confirms before persisting).
  Use when: "bedrock analyze-call", "/bedrock:analyze-call",
  "analisa a call de hoje com X", "analisa essa reunião", "análise comportamental",
  or when the user wants to extract behavioral patterns from a specific meeting.
user_invocable: true
allowed-tools: Bash, Read, Write, Edit, Glob, Grep, Skill, Agent, ToolSearch, WebFetch, mcp__granola__*
---

# /bedrock:analyze-call — Behavioral Analysis of a Single Call

## Plugin Paths

Same convention as other Bedrock skills. `<base_dir>` is provided at invocation.

- Entity definitions: `<base_dir>/../../entities/`
- Templates: `<base_dir>/../../templates/{type}/_template.md`
- Plugin CLAUDE.md: `<base_dir>/../../CLAUDE.md` (auto-injected)

---

## Vault Resolution

Same chain as other Bedrock skills:

1. `--vault <name>` flag → registry lookup
2. CWD detection → registered vault path match
3. Default vault from registry
4. Error if no vault found

From this point: `VAULT_PATH` is the resolved vault root.

---

## Overview

This skill analyzes ONE specific call/meeting for **behavioral coaching purposes** —
not for ingestion. It runs alongside `/bedrock:granola` (which handles fluxo padrão
of meetings) and complements it.

**You are a coaching analysis agent.** Follow the phases below in order.

**Hard rules (read these first):**

1. Output is **observations**, not judgments. Never write "X is arrogant". Always anchor:
   "On 2026-05-04 in call X, interrupted Y three times during the explanation of Z".
2. **Never persist literal transcript** in the vault. Only curated observations.
3. Default mode is **interactive** — always show analysis to user and ask for
   confirmation before persisting. Never write to vault silently.
4. Output is **private to the vault owner**. Mark every entry with
   `<!-- private: coaching -->` on the first content line.
5. Transcripts are noisy. Treat conclusions as preliminary. Avoid quoting literal
   phrases (they often contain transcription errors). Describe patterns instead.
6. If uncertain about a pattern (single occurrence, low signal), say so explicitly.

---

## Phase 0 — Parse Arguments

Skill is invoked as:

```
/bedrock:analyze-call [SOURCE] [REFERENCE] [--focus <person-slug>] [--draft] [--no-self]
```

Or in natural language ("analisa a call de hoje com o Cannuto"). Parse intent:

- **SOURCE** ∈ {`granola`, `gdoc`, `last`, auto-detect from natural language}
- **REFERENCE** depends on source:
  - granola: meeting UUID OR title fragment (skill will search via list_meetings)
  - gdoc: full URL of a Google Doc containing the Meet transcript
  - last: no reference; fetch most recent meeting from Granola today
- **--focus**: person slug to focus the analysis on. Default: all managed people in the call.
- **--draft**: do not persist; only show analysis output.
- **--no-self**: skip auto-analysis of the vault owner. Default: include.

If args ambiguous, ask one grouped clarification question before proceeding.

---

## Phase 1 — Fetch Transcript via Adapter

### 1.1 Granola adapter

Verify MCP availability:

```
ToolSearch(query: "select:mcp__granola__list_meetings,mcp__granola__get_meeting_transcript,mcp__granola__get_meetings", max_results: 3)
```

If MCP unavailable: graceful error and stop.

**Resolve meeting:**
- If UUID provided → use directly
- If title fragment → `mcp__granola__list_meetings(time_range: "this_week")`, fuzzy match
- If `last` → list this_week, take most recent (sort by date desc)
- If multiple matches → ask user to disambiguate

**Fetch transcript and metadata:**

```
transcript = mcp__granola__get_meeting_transcript(meeting_id: <uuid>)
details = mcp__granola__get_meetings(meeting_ids: [<uuid>])
```

`details` provides participants, title, date. Save these.

### 1.2 GDoc adapter

If source is `gdoc`:

Delegate to `/bedrock:gdoc-to-markdown` skill (or call it as a sub-skill) with the URL.
Returns markdown of the transcript. Then parse: identify speaker labels (Meet uses
"Speaker Name HH:MM:SS" format).

Get participants list from the doc header if present, or infer from speaker labels.

### 1.3 Length sanity check

If transcript is shorter than ~10 minutes of content (approx <2000 words):
warn user that confidence is low. Offer to abort.

---

## Phase 2 — Load Vault Context

Same as `/bedrock:granola` Phase 3, but only what's needed for behavioral analysis:

1. **Vault owner identity:** read `<VAULT_PATH>/people/` to find the entry where
   `aliases` includes the user's name from config, OR `management_role: "self"`.
   Save as `OWNER_SLUG`.
2. **Managed people:** Glob `<VAULT_PATH>/people/*.md` filtered by
   `management_role` ∈ {`direct-report`, `indirect-report`}. Build alias lookup.
3. **Reforge competency definitions** (optional context for signal mapping):
   read from vault if present in `Desenvolvimento/` folder.

---

## Phase 3 — Normalize Speakers

Map each transcript turn to a known person slug.

### 3.1 Granola normalization

Granola transcripts use `Me:` (vault owner) and `Them:`.

- **1:1 (exactly 2 known participants):** "Me" → `OWNER_SLUG`. "Them" → resolve
  the OTHER participant via alias map. **Proceed.**
- **Group meeting (3+ known participants):** "Them" is ambiguous — multiple
  people share the label. **STOP and ask user:**
  > "Granola não distingue speakers em reuniões de grupo (todos viram 'Them').
  > Posso fazer uma análise da dinâmica geral (% Pedro x % grupo, padrões seus
  > apenas), mas não consigo atribuir comportamentos individuais a cada
  > participante. Prosseguir mesmo assim, ou prefere usar GDoc/Meet transcript?"

### 3.2 GDoc/Meet normalization

Meet uses real names. For each speaker label:

1. Match against the alias map of vault people.
2. If match → use the person slug.
3. If no match → mark as `external:<Name>` (no entity created, no persistence
   for this speaker).

---

## Phase 4 — Run Analysis

For each known speaker (vault owner + focused person(s)), compute:

### 4.1 Conversational dynamics (across all speakers)

- Approximate share of speech (word count per speaker as % of total).
- Number of speaker turns.
- Interruption count and direction (X→Y means X cut Y mid-thought).
- Question count per speaker (turns ending in `?` or starting with question
  patterns: "você", "como", "por que", "será que", etc.).

### 4.2 Linguistic patterns per speaker

- **Hedging frequency:** "acho", "talvez", "sei lá", "tipo", "meio que".
- **Assertiveness markers:** declarative sentences without hedging vs hedged statements.
- **Justification before request:** does the speaker explain the rationale before
  asking for input? (pattern of "blindagem")
- **Topic-shift initiation:** who starts new topics?
- **Validation seeking:** "faz sentido?", "concorda?", "está ok?".

### 4.3 Behavioral signals (only if clearly observable)

Map to Reforge Product Competency Model dimensions when there is **explicit
evidence**:

- Communication / Fluency
- Strategic thinking
- Execution
- Discovery
- Quality

If signal is weak or single-occurrence, do **not** report it.

**Reforge as canonical source for vocabulary.** When mapping a signal to a
sub-competency or to a leadership pattern (e.g., "directive vs supportive",
"managing up", "stakeholder calibration"), validate the term against the
Reforge material in
`<VAULT_PATH>/Desenvolvimento/reforge-product-leadership.md`.

**How to consult (do NOT load the file into context):** the file is ~115k
words. Use targeted grep instead:

```bash
grep -n -B 2 -A 8 "<term>" <VAULT_PATH>/Desenvolvimento/reforge-product-leadership.md
```

Useful pivot terms: "directive leadership", "supportive leadership",
"feedback", "coaching conversation", "performance management", "team
archetype", "Product Leader Canyon", "managing up", "stakeholder".

When citing a Reforge concept in the analysis output, reference it in passing
("padrão típico de directive leadership leaning"); do not copy the source
material into the Log entry.

### 4.4 Friction / repetition

- Topics where speakers disagreed and didn't reconcile.
- Topics where the vault owner had to repeat themselves to be understood.
- Topics that the focused person evaded or deflected.

### 4.5 Suggested next-1:1 items

- Items that came up in the call but weren't fully resolved.
- Behavioral feedback that the owner might want to deliver.
- Things the focused person offered (help, opinion) that the owner accepted/declined.

---

## Phase 4.6 — Pattern Check Across Previous Analyses

**Goal:** Detect whether observations from the current call repeat patterns from
prior `[análise]` entries in the same person's Log. Surfaces recurring behavior
that warrants a Tema em Acompanhamento.

### 4.6.1 Fetch prior analyses

For each person being analyzed (focused person + owner):

```bash
# Read the person's Log section and extract all [análise] entries
grep -n '^### [0-9]\{4\}-[0-9]\{2\}-[0-9]\{2\} — \[análise\]' \
  <VAULT_PATH>/people/<person-slug>.md
```

For each match found, extract the entry block (from header until next `### `
header or end of `## Log` section). Sort by date desc, take the **last 5 entries
maximum**. Skip the current call (if already written, by source ID).

If fewer than **2 prior entries** exist for the person: pattern check is
**skipped** (not enough signal). State this in the output: "Pattern check
pulado: histórico insuficiente (<2 análises prévias)."

### 4.6.2 Semantic comparison

For each observation in the current analysis (Phase 4.2 to 4.4), compare against
observations in prior entries. **Use semantic equivalence, not string match**:

- "justifica antes de pedir input" ≈ "blindagem com 4 argumentos antes da pergunta"
- "recusou ajuda sem destrinchar" ≈ "fechou oferta do Mac antes de explorar"
- "interrompeu durante explicação técnica" ≈ "cortou no meio do raciocínio"

Two observations are the same pattern if they describe the same underlying
behavior, even if phrased differently. The LLM doing the comparison should err
toward **conservative matching** — when in doubt, don't merge.

### 4.6.3 Threshold

A pattern is **flagged as recurring** when it appears in ≥2 distinct analyses
(including the current one). Strong patterns: ≥3 occurrences.

### 4.6.4 Output structure

Add a new section to the rendered analysis (Phase 5) for each person:

```markdown
**Padrões recorrentes detectados:**
- [3x] {pattern description} ({date1}, {date2}, {date3})
- [2x] {pattern description} ({date1}, {date2})
```

If no pattern was flagged: omit the section entirely (don't show empty).

### 4.6.5 Limits and guards

- **Max 5 entries** read per person (window size). Don't try to compare against
  the entire history — costs grow and signal degrades.
- **Conservative clustering.** If you're uncertain whether two observations
  describe the same pattern, treat them as distinct.
- **Don't claim trends from 2 occurrences alone.** Mark as "[2x] possível padrão".
  Reserve "[3x]" or higher for confident calls.
- **Don't infer causation.** "Cannuto cortou Carol em 3 calls" is observation,
  not "Cannuto desrespeita Carol".
- **Vocabulary drift:** if the user describes the same pattern with different
  words across analyses, the LLM should still cluster them. Document the
  consolidated phrasing in the output.

---

## Phase 5 — Render Output

For each person to be written about (focused person(s) + optionally owner),
render a Log entry in this exact format:

```markdown
### YYYY-MM-DD — [análise] {meeting title} ({source}:{id})
<!-- private: coaching -->

**Dinâmica:** {OWNER_NAME} ~X% / {OTHER_NAME} ~Y% da fala, N interrupções

**Observações sobre {OTHER_NAME}:**
- {pattern 1, anchored in evidence}
- {pattern 2, anchored in evidence}

**Padrões meus na call:** *(only in OWNER's note)*
- {self-observation 1}
- {self-observation 2}

**Padrões recorrentes detectados:** *(omit section if none)*
- [3x] {pattern description} ({date1}, {date2}, {date3})
- [2x] {pattern description} ({date1}, {date2})

**Sinais de competências (preliminar, não atualizar score):**
- {OTHER_NAME}: {dimension} {score-suggestion} — {evidence}

**Pra próximo 1:1:**
- [ ] {follow-up item} *(YYYY-MM-DD)*
```

Notes:

- Header format is **fixed** for parseability: `### YYYY-MM-DD — [análise] <title> (<source>:<id>)`.
- The `[análise]` prefix is the marker that future aggregation skills will use to filter.
- The `<source>:<id>` enables traceability and dedup (`granola:c82110d9-...` or `gdoc:1AbCd...`).
- Suggested 1:1 items use the same format as `/bedrock:dump` so they merge naturally.

---

## Phase 6 — Interactive Confirmation

Default flow (always interactive unless `--draft` is set):

1. Show the rendered analysis to the user.
2. Ask explicit questions in this order:
   - **"Gravar essas observações no Log de {pessoa}? (sim / não / editar)"**
   - **"Adicionar item ao Próximo 1:1 de {pessoa}? (sim / não / editar)"**
   - **For each recurring pattern detected (Phase 4.6):**
     **"Padrão '{descrição}' apareceu em {N} análises. Adicionar em Temas em Acompanhamento de {pessoa}? (sim / não / editar)"**
   - **"Gravar análise sobre você em `pedro-monteiro.md`? (sim / não)"** *(only if --no-self was not passed)*
3. If user says "editar", let them dictate the changes inline before persisting.
4. If user says "não" to all → respond "OK, nada foi gravado." and stop.

---

## Phase 7 — Persist via /bedrock:preserve

Build structured input for `/bedrock:preserve`. Each block goes to the **Log**
section (route_to: log). The header date is the meeting date; the marker is
included in the content body.

```yaml
entities:
  - type: person
    name: "{focused person name}"
    action: update
    route_to: log
    content: |
      [análise] {meeting title} ({source}:{id})
      <!-- private: coaching -->

      **Dinâmica:** ...
      **Observações:** ...
      **Sinais de competências (preliminar):** ...
      **Pra próximo 1:1:** ...
    date: "YYYY-MM-DD"

  - type: person
    name: "{focused person name}"
    action: update
    route_to: proximo_1_1
    content: "{follow-up item from analysis}"
    date: "YYYY-MM-DD"

  - type: person
    name: "{focused person name}"  # only when user confirmed adding pattern to Temas em Acompanhamento
    action: update
    route_to: temas_em_acompanhamento
    content: |
      ### Coaching: {pattern description}
      <!-- private: coaching -->
      Padrão observado em {N} análises ({date1}, {date2}, ...). {brief context}.
    date: "YYYY-MM-DD"

  - type: person
    name: "Pedro Monteiro"  # only if --no-self was not passed and self-section has content
    action: update
    route_to: log
    content: |
      [análise] {meeting title} ({source}:{id})
      <!-- private: coaching -->

      **Padrões meus:** ...
    date: "YYYY-MM-DD"
```

Delegate to `/bedrock:preserve`.

**Important:** the Log entry header MUST follow the format
`### YYYY-MM-DD — [análise] {title} ({source}:{id})`. This is enforced because
future aggregation skills depend on it. If `/bedrock:preserve` strips or alters
the format, fix the rendering before invoking.

---

## Phase 8 — Summary

Present a compact summary to the user:

```
Análise gravada:
- Log de [[rodrigo-cannuto]] → 1 entrada [análise]
- Próximo 1:1 de [[rodrigo-cannuto]] → 2 itens
- Temas em Acompanhamento de [[rodrigo-cannuto]] → 1 padrão recorrente registrado
- Log de [[pedro-monteiro]] → 1 entrada [análise] (auto-coaching)

Pattern check: {N entradas prévias lidas, M padrões recorrentes detectados}.

Confiança: {alta / média / baixa} — transcrição com {N} marcadores de ruído
detectados ({pequenos exemplos}).

Fonte: {granola:uuid | gdoc:url}
```

---

## Rules (recap)

1. **Default mode is interactive.** Never persist without user confirmation.
2. **Anchored observations only.** No absolute judgments. Each statement has
   evidence (interruption count, % of speech, repeated pattern).
3. **No literal transcript in the vault.** Only curated observations.
4. **Privacy marker on every entry.** `<!-- private: coaching -->` first content line.
5. **Header format is fixed** for aggregation: `### YYYY-MM-DD — [análise] {title} ({source}:{id})`.
6. **Group meetings via Granola → ask before proceeding.** Granola can't distinguish
   speakers in groups; offer to switch to Meet transcript via GDoc, or proceed with
   group-level dynamics only.
7. **Low-confidence signals are suppressed.** Only report a behavioral signal when
   it occurs more than once or is unambiguous.
8. **Language:** match vault config (Portuguese here). Frontmatter keys stay in English.
9. **No auto-update of competency scores.** Only suggestions, marked "preliminar".
10. **Graceful degradation** when MCP/GDoc adapter fails: inform user and stop.
11. **Pattern check window:** read ≤5 prior `[análise]` entries per person.
    Skip pattern check if <2 prior entries exist. Threshold for flagging: ≥2
    occurrences (including current call). "Possível padrão" for [2x],
    "padrão recorrente" for [3x]+.
12. **Patterns become Temas em Acompanhamento only after user confirmation.**
    Never auto-promote a recurring observation. The Tema entry includes the
    privacy marker and references source dates for traceability.
