# Ativos Plugins

Marketplace de plugins do Claude Code para a Ativos Precatórios.

## Plugins disponíveis

| Plugin  | Descrição                          | Skills                |
|---------|--------------------------------------|------------------------|
| `radar` | Skills específicas do projeto Radar | `bulk-refactor`, `create-pr` (mais serão adicionadas) |

## Como usar em outro projeto

Este repositório é **privado**, então é preciso ter acesso git normal a ele
(SSH key cadastrada no GitHub, ou `gh auth login`) na máquina onde o Claude
Code vai rodar.

Dentro do Claude Code, no projeto onde você quer usar o plugin:

```
/plugin marketplace add davilos/ativos-plugins
/plugin install radar@ativos
```

Depois disso as skills `bulk-refactor` e `create-pr` ficam disponíveis nesse projeto.

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
      <outras-skills>/    ← novas skills do projeto Radar entram aqui
        SKILL.md
```

## Como adicionar uma nova skill a um plugin existente

Basta criar `plugins/<plugin>/skills/<nome-da-skill>/SKILL.md`. Não precisa
mexer em `plugin.json` nem em `marketplace.json` — quem já instalou o plugin
recebe a skill nova ao rodar `/plugin marketplace update ativos`.

## Como adicionar um novo plugin

1. Criar `plugins/<nome-do-plugin>/.claude-plugin/plugin.json` com `name`,
   `description`, `version` e `author`.
2. Adicionar as skills/comandos/agents do plugin dentro da mesma pasta
   (`skills/`, `commands/`, `agents/`, conforme o caso).
3. Registrar o plugin em `.claude-plugin/marketplace.json`, no array `plugins`,
   apontando `source` para a pasta criada.
4. Commitar e dar push — quem já tiver o marketplace adicionado só precisa
   rodar `/plugin marketplace update ativos` para ver o novo plugin.
