# claude-bedrock

Claude Code plugin for Obsidian vault automation — entity management, ingestion, compression, and sync.

Bedrock turns any Obsidian vault into a structured Second Brain with 7 entity types (actors, people, teams, topics, discussions, projects, fleeting) managed by AI agents via Claude Code skills.

## Installation

```bash
claude plugins add iurykrieger/claude-bedrock
```

Or for local development/testing:

```bash
claude --plugin-dir ./claude-bedrock
```

## Skills

| Skill | Purpose |
|---|---|
| `/bedrock:query` | Smart vault reader — answers questions by searching and cross-referencing entities |
| `/bedrock:teach` | Ingest external sources (Confluence, GDocs, GitHub, CSV) — extracts entities |
| `/bedrock:preserve` | Single write point — entity detection, matching, create/update, bidirectional links |
| `/bedrock:compress` | Deduplication and vault health — broken links, orphan entities, stale content |
| `/bedrock:sync` | Re-sync entities with external sources |

## Vault Structure

After installing the plugin, your vault should follow this directory layout:

```
your-vault/
├── actors/          # Systems, services, APIs (permanent notes)
├── people/          # Contributors, team members (permanent notes)
├── teams/           # Squads, organizational units (permanent notes)
├── topics/          # Cross-cutting subjects with lifecycle (bridge notes)
├── discussions/     # Meeting notes, conversations (bridge notes)
├── projects/        # Initiatives with scope and deadline (index notes)
└── fleeting/        # Raw ideas, unstructured captures (fleeting notes)
```

Each directory should contain a `_template.md` defining the frontmatter schema. Templates are available in this plugin's `templates/` directory.

## Entity Definitions

The `entities/` directory contains semantic definitions for each entity type — used by skills to classify and route content correctly. These define:

- What each entity type represents
- When to create vs. not create
- Required frontmatter fields
- Zettelkasten role and linking rules
- Completeness criteria

## Writing Rules

All vault content follows these conventions (enforced via the plugin's CLAUDE.md):

- **Language:** Portuguese (pt-BR), technical terms in English
- **Frontmatter:** Keys in English, values in pt-BR
- **Wikilinks:** Bare names only (`[[name]]`, never `[[dir/name]]`)
- **Tags:** Hierarchical (`type/actor`, `status/active`, `domain/acquiring`)
- **Aliases:** Minimum 1 per entity
- **Git:** Trunk-based, commit convention `vault(<type>): <verb> <name> [fonte: <source>]`

See [CLAUDE.md](CLAUDE.md) for the complete set of rules.

## License

MIT
