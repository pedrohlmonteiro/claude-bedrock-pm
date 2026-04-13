---
type: person
name: ""
aliases: []  # ["Full Name Capitalized", "Nickname"] — min 1 alias
role: ""
team: "[[squad-name]]"
focal_points: []
email: ""  # email corporativo completo (ex: iury.krieger@stone.com.br)
github: ""  # opcional — login do GitHub, quando aplicável
slack: ""  # opcional — arroba do Slack (ex: @iury.krieger)
jira: ""
sources: []  # [{url: "https://...", type: "confluence|gdoc|github-repo|csv|markdown|manual", synced_at: YYYY-MM-DD}]
updated_at: YYYY-MM-DD
updated_by: ""
tags: [type/person]  # + domain/* opcional
---

<!-- Papel Zettelkasten: permanent note -->
<!-- Links no corpo devem ter contexto: "lidera a migracao do [[autobahn]] para [[payment-card-api]]" -->

<!-- Filename convention: prefixo do email corporativo, pontos → hífens.
     Ex: iury.krieger@stone.com.br → iury-krieger.md
     Quando email desconhecido: first-last.md baseado no nome completo. -->

# First Last

> Breve descrição (2-3 linhas) sobre a atribuição atual da pessoa na StoneCo — cargo, área de atuação, e contexto relevante.

## Time

Membro de [[squad-name]].

## Pontos Focais

- [[repo-name]] — contexto de envolvimento
- [[repo-name]] — contexto de envolvimento

## Assuntos Ativos

- [[YYYY-MM-type-slug]] — breve descrição

---

## Expected Bidirectional Links

> This section is a reference for agents and can be removed in real pages.

| From | To | Field |
|---|---|---|
| Person → Team | `[[squad-name]]` | `team` in frontmatter |
| Person → Actor | `[[repo-name]]` | "Pontos Focais" section |
| Person → Topic | `[[YYYY-MM-type-slug]]` | "Assuntos Ativos" section |
| Team → Person | `[[first-last]]` | `members` in Team frontmatter |
| Topic → Person | `[[first-last]]` | `people` in Topic frontmatter |
