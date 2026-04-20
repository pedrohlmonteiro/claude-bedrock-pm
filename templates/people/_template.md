---
type: person
name: ""
aliases: []  # ["Full Name Capitalized", "Nickname"] — min 1 alias
role: ""
team: "[[squad-name]]"
focal_points: []
email: ""  # full corporate email (e.g.: alice.smith@company.com)
github: ""  # optional — GitHub login, when applicable
slack: ""  # optional — Slack handle (e.g.: @alice.smith)
jira: ""
# --- Management fields (optional — only when management_role is set) ---
management_role: ""  # direct-report | indirect-report | peer | leader | external — leave empty for non-managed persons
manager: ""  # "[[manager-slug]]" — wikilink to this person's manager
# --- Competencies: Reforge Product Competency Model (optional, scale 0-5) ---
# Only populate when management_role is set. Leader assessment + self-assessment.
product_sense: 0
product_sense_self: 0
analytical: 0
analytical_self: 0
execution: 0
execution_self: 0
strategic_thinking: 0
strategic_thinking_self: 0
fluency: 0
fluency_self: 0
discovery: 0
discovery_self: 0
growth: 0
growth_self: 0
go_to_market: 0
go_to_market_self: 0
quality: 0
quality_self: 0
delivery: 0
delivery_self: 0
user_insight: 0
user_insight_self: 0
data_intuition: 0
data_intuition_self: 0
# --- End management fields ---
sources: []  # [{url: "https://...", type: "confluence|gdoc|github-repo|csv|markdown|manual", synced_at: YYYY-MM-DD}]
updated_at: YYYY-MM-DD
updated_by: ""
tags: [type/person]  # + domain/* optional
---

<!-- Zettelkasten role: permanent note -->
<!-- Links in the body must have context: "leads the migration of [[legacy-gateway]] to [[billing-api]]" -->

<!-- Filename convention: corporate email prefix, dots → hyphens.
     E.g.: alice.smith@company.com → alice-smith.md
     When email is unknown: first-last.md based on full name. -->

# First Last

> Brief description (2-3 lines) about the person's current role in the organization — position, area of expertise, and relevant context.

## Team

Member of [[squad-name]].

## Focal Points

- [[repo-name]] — context of involvement
- [[repo-name]] — context of involvement

## Active Topics

- [[YYYY-MM-type-slug]] — brief description

<!-- ============================================================
     MANAGEMENT SECTIONS — Include ONLY when management_role is set.
     Remove this entire block for non-managed persons.
     ============================================================ -->

---

## Próximo 1:1

> [!todo] Pauta acumulada para o próximo 1:1
> Itens adicionados automaticamente via daily dump, reuniões ou notas rápidas.

- [ ] %%vazio — novos itens serão adicionados aqui%%

---

## Temas em Acompanhamento

%%Temas ativos que precisam de follow-up recorrente%%

---

## Desenvolvimento / PDI

---

## Log

%%Registro corrido de observações, feedbacks e contexto relevante. Mais recente no topo.%%

<!-- ============================================================
     END MANAGEMENT SECTIONS
     ============================================================ -->

---

## Expected Bidirectional Links

> This section is a reference for agents and can be removed in real pages.

| From | To | Field |
|---|---|---|
| Person → Team | `[[squad-name]]` | `team` in frontmatter |
| Person → Actor | `[[repo-name]]` | "Focal Points" section |
| Person → Topic | `[[YYYY-MM-type-slug]]` | "Active Topics" section |
| Person → Manager | `[[manager-slug]]` | `manager` in frontmatter (management only) |
| Team → Person | `[[first-last]]` | `members` in Team frontmatter |
| Topic → Person | `[[first-last]]` | `people` in Topic frontmatter |
