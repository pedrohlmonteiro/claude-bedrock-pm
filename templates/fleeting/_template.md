---
type: fleeting
title: ""
aliases: []  # ["Short Title"] — min 1 alias se titulo for longo
source: ""  # session | teach | manual
captured_at: YYYY-MM-DD
status: "raw"  # raw | reviewing | promoted | archived
promoted_to: ""  # "[[nota-destino]]" quando promovida
sources: []  # [{url: "https://...", type: "confluence|gdoc|github-repo|csv|markdown|manual", synced_at: YYYY-MM-DD}]
updated_at: YYYY-MM-DD
updated_by: ""
tags: [type/fleeting, status/raw]  # + domain/* opcional
---

<!-- Papel Zettelkasten: fleeting note -->
<!-- Conteudo em maturacao — links exploratórios permitidos sem contexto textual completo -->

# Fleeting Title

> Captura bruta de informação. Fonte: `source`.

## Conteudo

Informação capturada — ideias, fragmentos, menções. Não precisa estar completa ou bem estruturada.

## Conexoes Possiveis

Wikilinks exploratórios para entidades que parecem relacionadas:
- [[entity-name]] — motivo da possível conexão

## Contexto de Captura

De onde veio, quando, e qualquer contexto adicional que ajude na futura promoção.

---

## Expected Bidirectional Links

> This section is a reference for agents and can be removed in real pages.

| From | To | Field |
|---|---|---|
| Fleeting -> Permanent/Bridge | `[[entity-name]]` | `promoted_to` in frontmatter (when promoted) |
