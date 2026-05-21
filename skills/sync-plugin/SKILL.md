---
name: sync-plugin
description: Importa concepts novos de um Claude Code plugin instalado para o vault Bedrock. Diff one-way (plugin → vault) por basename — copia arquivos que existem no cache do plugin mas ainda não estão em `<vault>/concepts/`, adicionando `sources:` no frontmatter pra rastreabilidade. Concepts existentes em ambos são skipados por default (modo `--diff` mostra divergências sem tocar nos arquivos). Use após `claude plugin update <plugin>` quando quiser que o Bedrock e o Obsidian conheçam o conhecimento que outros contribuidores adicionaram ao plugin. Não sobrescreve concepts existentes; não desinstala nada. Trigger: "sync plugin", "bedrock sync-plugin", "/bedrock:sync-plugin", "puxar atualizações do plugin", "trazer concepts do plugin pro vault".
user_invocable: true
allowed-tools: Bash, Read, Write, Glob, Skill
---

# /bedrock:sync-plugin — Importar concepts de um Claude Code plugin

One-way sync: cache do plugin → vault. Importa apenas concepts que ainda não existem no vault. Não sobrescreve, não desinstala.

## Quando usar

- Após `claude plugin update <plugin>` quando há suspeita de concepts novos
- Pra trazer pro vault contribuições que terceiros mergearam no plugin
- Pra que `/bedrock:ask` e o Obsidian (graph, wikilinks) conheçam concepts do plugin

## Plugin Paths

Skill segue convenção do `/bedrock:sync` pra resolver paths internos:
- Entity definitions: `<base_dir>/../../entities/`
- Templates: `<base_dir>/../../templates/{type}/_template.md`

Use a "Base directory for this skill" provided at invocation pra resolver paths.

## Argumentos

```
/bedrock:sync-plugin <plugin-name>                   # required, ex: prod-data-kb
/bedrock:sync-plugin <plugin-name> --diff            # mostra diff dos existentes (read-only)
/bedrock:sync-plugin <plugin-name> --dry-run         # só relata, não escreve
/bedrock:sync-plugin <plugin-name> --vault <name>    # vault target (default: detectado por CWD ou default)
```

## Vault Resolution

Igual ao `/bedrock:sync`:

1. **Se `--vault <name>` provided:** lê `<base_dir>/../../vaults.json`, acha entrada matching. Se ausente, erro "Vault `<name>` not registered. Run `/bedrock:vaults`."
2. **Se sem flag — CWD detection:** lê o registry e checa se CWD está dentro de algum vault registrado. Match mais específico vence.
3. **Fallback default vault:** entry com `"default": true` no registry.
4. **Se nada resolve:** erro listando vaults disponíveis.

Set `VAULT_PATH` e `VAULT_NAME`. Validar `test -d "$VAULT_PATH"`. Ler `<VAULT_PATH>/.bedrock/config.json` pra campos relevantes (`language`, `git.strategy`).

A partir desse ponto, **toda operação de vault file usa `<VAULT_PATH>`**.

## Fluxo

### Passo 1 — Resolver cache do plugin

```bash
PLUGIN_NAME="<plugin-name>"
PLUGIN_BASE="$HOME/.claude/plugins/cache"

# O cache segue o padrão: <cache>/<marketplace-name>/<plugin-name>/<version>/
# Geralmente o marketplace tem o mesmo nome do plugin (single-plugin marketplace).
# Buscamos primeiro pelo padrão direto, depois fallback pra qualquer marketplace
# que contenha o plugin.

CANDIDATE="$PLUGIN_BASE/$PLUGIN_NAME/$PLUGIN_NAME"
if [ -d "$CANDIDATE" ]; then
  PLUGIN_DIR="$CANDIDATE"
else
  # Procurar em todos marketplaces
  PLUGIN_DIR=$(find "$PLUGIN_BASE" -maxdepth 2 -type d -name "$PLUGIN_NAME" 2>/dev/null | head -1)
fi

if [ -z "$PLUGIN_DIR" ] || [ ! -d "$PLUGIN_DIR" ]; then
  echo "Plugin '$PLUGIN_NAME' não está instalado."
  echo "Rode: claude plugin install $PLUGIN_NAME@$PLUGIN_NAME"
  exit 1
fi

# Versão mais recente (lexicograficamente — semver-friendly até v9.x.x)
LATEST_VERSION=$(ls -1 "$PLUGIN_DIR" | sort -V | tail -1)
PLUGIN_VERSION_DIR="$PLUGIN_DIR/$LATEST_VERSION"

PLUGIN_CONCEPTS="$PLUGIN_VERSION_DIR/concepts"
if [ ! -d "$PLUGIN_CONCEPTS" ]; then
  echo "Plugin '$PLUGIN_NAME' v$LATEST_VERSION não tem pasta concepts/."
  echo "Esse plugin é uma KB? Verifique."
  exit 1
fi
```

### Passo 2 — Listar concepts no plugin

```bash
# Lista todos os .md em concepts/ recursivamente, excluindo template e README
PLUGIN_FILES=$(find "$PLUGIN_CONCEPTS" -name "*.md" \
  -not -name "_TEMPLATE.md" -not -name "_template.md" \
  -not -name "README.md" -not -name "readme.md" 2>/dev/null)
```

Conte e armazene em estrutura `{basename, relative_path, absolute_path}` por arquivo.

### Passo 3 — Listar concepts no vault

```bash
# Vault usa estrutura plana em <vault>/concepts/
VAULT_FILES=$(find "$VAULT_PATH/concepts" -maxdepth 1 -name "*.md" \
  -not -name "_template.md" 2>/dev/null)
```

Armazene basenames pro diff.

> **Nota de design:** o plugin organiza concepts por domínio (`concepts/finance/`, `concepts/transacional/`); o vault Bedrock usa estrutura plana (`concepts/<slug>.md`). O diff é por **basename**, não path. Quando um concept entra no vault, perde a estrutura de domínio (a tag `domain/<x>` no frontmatter preserva a categorização).

### Passo 4 — Diff por basename

Para cada arquivo no plugin:
- `basename=<slug.md>`
- Se `basename ∈ VAULT_FILES`: categoria **existente**
- Se `basename ∉ VAULT_FILES`: categoria **novo**

Resultado: duas listas (`novos`, `existentes`).

### Passo 5 — Mostrar resumo (sempre, antes de qualquer escrita)

```
Vault: <VAULT_NAME> (<VAULT_PATH>)
Plugin: <PLUGIN_NAME> v<LATEST_VERSION>

Plugin tem <N> concepts. Vault tem <M> concepts.

Novos (vão ser importados):
  - <relative-path-no-plugin> → <vault>/concepts/<basename>
  - ...

Existentes (skipados — use --diff pra comparar):
  - <basename> (no plugin: <relative-path>)
  - ...
```

Se `--dry-run`, parar aqui — não escrever nada.

### Passo 6 — Importar novos

Pra cada concept em `novos`:

1. **Read** o source no plugin (`absolute_path`).
2. **Parse frontmatter** (YAML entre `---`).
3. **Atualizar frontmatter**:
   - Adicionar entry em `sources:` (criar a key se não existir):
     ```yaml
     sources:
       - url: "claude-plugin://<PLUGIN_NAME>/<relative-path>"
         type: "markdown"
         synced_at: "<YYYY-MM-DD hoje>"
     ```
     **Não remover entries existentes em `sources:`.** Apenas adicionar.
   - Setar `updated_at: <hoje>`.
   - Setar `updated_by: "sync-plugin@agent"`.
   - **Não alterar** `name`, `aliases`, `description`, `tags`, conteúdo do body.
4. **Write** em `<VAULT_PATH>/concepts/<basename>`.

### Passo 7 — Lint (best-effort)

Se a skill `/lint-concept` do plugin estiver disponível (verifique via `command -v` ou olhe se existe `$PLUGIN_VERSION_DIR/skills/lint-concept/`):
- Invoque `/lint-concept <vault>/concepts/<basename>` pra cada novo importado
- Reporte warnings sem bloquear
- Se erros (exit 1), reporte com destaque mas não reverta a escrita — autor humano decide

### Passo 8 — Diff dos existentes (só se `--diff`)

Para cada existente:
- Read source no plugin
- Read source no vault
- Compare via `diff -u` (bash) ou comparação textual
- Reporte: "concepts/<basename>: <N> linhas diferentes — investigar manualmente se quiser atualizar"
- **NÃO** mexe no arquivo do vault — diff é read-only

### Passo 9 — Reporte final

```
✓ Importados (<N>):
  - <basename>
  - ...
↺ Skipados (<M> existentes)
⚠ Warnings de lint:
  - <basename>: <warning resumido>
  ...

Vault git: <N> arquivos novos em <VAULT_PATH>/concepts/.
Considere commit (git.strategy do vault: <config.json strategy>).
```

## Limitações conhecidas (v1)

- **Wikilinks órfãos:** concepts importados podem referenciar outros que ainda estão só no plugin. Rode `/bedrock:sync-plugin` de novo após plugin update se houver dependências em série.
- **Frontmatter incompatível:** se um concept do plugin tem fields não-Bedrock (raro), preservados mesmo assim — Bedrock tolera frontmatter extra.
- **Atualização de existentes:** **NÃO** faz upgrade automático de concepts já no vault. Por design: vault local pode ter versão mais rica (wikilinks pessoais, log). Use `--diff` pra investigar e edite manualmente se quiser.
- **Sem rollback:** se a escrita falhar no meio, arquivos parcialmente importados ficam no vault. Vault git (config `strategy: commit-only`) permite revert manual.
- **Concept não é entity Bedrock de 1ª classe:** essa skill escreve direto em `<VAULT_PATH>/concepts/<basename>` sem delegar pra `/bedrock:preserve` ou `/bedrock:teach`. Se concepts virarem entity formal no Bedrock no futuro, refatorar pra delegar.

## O que NÃO fazer

- **Não sobrescrever** concept existente no vault sem `--diff` + confirmação humana.
- **Não desinstalar** concept do vault porque sumiu do plugin.
- **Não modificar** concept no plugin (read-only do lado do plugin).
- **Não rodar `/bedrock:teach`** sobre concepts do plugin — eles já estão estruturados; passar pela extração reinterpreta conteúdo.
- **Não criar entities Bedrock** (actors, topics) a partir de wikilinks órfãos — usuário decide se cria.

## Casos de erro

| Situação | Ação |
|---|---|
| Plugin não instalado | Mensagem com `claude plugin install` |
| Plugin sem `concepts/` | Mensagem; possivelmente plugin não é KB |
| Vault não resolvido | Listar vaults registrados |
| Frontmatter quebrado num concept | Skip esse arquivo, reporte, continue |
| `concepts/` do vault não existe | Criar via `mkdir -p` (zero risco) |
| Permissão negada de escrita | Erro com path; não retentar |
