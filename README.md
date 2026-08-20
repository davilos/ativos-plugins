# Ativos Plugins

Marketplace de plugins do Claude Code para a Ativos Precatórios.

## Plugins disponíveis

| Plugin  | Descrição                          | Skills                |
|---------|--------------------------------------|------------------------|
| `radar` | Skills específicas do projeto Radar | `bulk-refactor`, `create-pr`, `pr-quality-gate` (mais serão adicionadas) |

## Como usar em outro projeto

Este repositório é **público**, então não é preciso nenhuma credencial git
para acessá-lo.

Dentro do Claude Code, no projeto onde você quer usar o plugin:

```
/plugin marketplace add davilos/ativos-plugins
/plugin install radar@ativos
```

Depois disso as skills `bulk-refactor`, `create-pr` e `pr-quality-gate` ficam disponíveis nesse projeto.

Para atualizar depois que novos plugins forem adicionados aqui:

```
/plugin marketplace update ativos
```

## Estrutura do repositório

```
.claude-plugin/
  marketplace.json       ← catálogo do marketplace (nome, owner, lista de plugins)
plugins/
  radar-team/
    .claude-plugin/
      plugin.json         ← metadados do plugin `radar`
    skills/
      bulk-refactor/
        SKILL.md          ← definição da skill
      create-pr/
        SKILL.md          ← gera título/descrição de PR e cria via `gh pr create`
      pr-quality-gate/
        SKILL.md          ← verifica e corrige os quality gates de uma PR até merge-ready
      <outras-skills>/    ← novas skills do projeto Radar entram aqui
        SKILL.md
```

## Como adicionar uma nova skill a um plugin existente

Criar `plugins/<plugin>/skills/<nome-da-skill>/SKILL.md` não basta: também é
preciso dar bump no `version` de `plugin.json` do plugin. Quem já instalou o
plugin só recebe a skill nova ao rodar `/plugin marketplace update ativos` se
a versão declarada mudou — a detecção de atualização compara essa versão, não
o SHA do git. Sem o bump, o cache local do plugin fica parado na skill antiga
mesmo depois do `update` (visto na prática em `20d4df8` e `b56f916`). Não é
preciso mexer em `marketplace.json` para isso.

## Como adicionar um novo plugin

1. Criar `plugins/<nome-do-plugin>/.claude-plugin/plugin.json` com `name`,
   `description`, `version` e `author`.
2. Adicionar as skills/comandos/agents do plugin dentro da mesma pasta
   (`skills/`, `commands/`, `agents/`, conforme o caso).
3. Registrar o plugin em `.claude-plugin/marketplace.json`, no array `plugins`,
   apontando `source` para a pasta criada.
4. Commitar e dar push — quem já tiver o marketplace adicionado só precisa
   rodar `/plugin marketplace update ativos` para ver o novo plugin.
