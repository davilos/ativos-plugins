---
name: create-pr
description: Analisa os commits da branch atual (desde que ela diverge da branch base, normalmente main) e o nome da branch para gerar o título e a descrição de um Pull Request, então cria o PR automaticamente via `gh pr create`. O tipo do PR (feat/fix/refactor/chore/...) é inferido do prefixo do nome da branch, com fallback para o tipo dominante nos commits (formato Conventional Commits já usado neste repo). A descrição lista as tarefas realizadas, extraídas das mensagens de commit. Invocar a skill já é o pedido explícito de publicar — ela push a branch e abre o PR, não é um dry-run. Aciona em "criar PR", "abrir pull request", "gera a descrição do PR", "abre um PR pra essa branch".
metadata:
  version: 1.0.0
  origin: criada em 2026-08-18 durante a sessão da branch fix-lint-errors (radar)
---

# Create PR

Transformar uma branch terminada em PR é duas coisas: reconstruir a história do que
foi feito a partir dos commits, e publicar isso — não é escrever uma descrição
genérica tipo "várias correções".

## Quando usar

Invocada explicitamente pelo usuário quando o trabalho na branch atual está pronto
para virar PR. A skill push a branch e chama `gh pr create` como parte normal do
fluxo — se o usuário quiser só um rascunho sem publicar, isso precisa ser dito
explicitamente antes de invocar (nesse caso, pare na Fase 4 e mostre o
título/corpo sem executar a Fase 5).

## Fase 1 — Determinar branch atual e branch base

- `git branch --show-current` para a branch de origem do PR.
- Branch base: normalmente `main` — confirme com `git remote show origin | grep
  'HEAD branch'` se o projeto não deixar isso óbvio, em vez de assumir `main` às
  ciegas em repositórios que você não conhece.
- `git merge-base <base> HEAD` define o ponto de divergência; os commits
  relevantes são `git log <base>..HEAD`.

## Fase 2 — Inferir o tipo do PR a partir do nome da branch

- Extraia o primeiro token do nome da branch, separando por `/` ou `-`:
  `fix-lint-errors` → `fix`; `feature/consulta-siconfi-aportes` → `feature`;
  `refactor-radar` → `refactor`.
- Normalize sinônimos (`feature` → `feat`).
- Tipos conhecidos: `feat`, `fix`, `refactor`, `chore`, `style`, `docs`, `perf`,
  `test`.
- Se o token não é um tipo conhecido (ex.: `update-dependencies`), não adivinhe
  pelo nome — olhe a distribuição real dos tipos nos commits do range
  (`git log <base>..HEAD --pretty=%s`, extraindo o prefixo `tipo(scope):`) e use
  o mais frequente. Se ainda ficar ambíguo, pergunte ao usuário antes de titular.

## Fase 3 — Coletar e sintetizar os commits em tarefas

- `git log <base>..HEAD --no-merges --pretty=format:'%s'`, em ordem cronológica
  (mais antigo primeiro) — o PR conta a história na ordem em que a mudança foi
  construída, não invertida.
- Para cada commit, remova o prefixo `tipo(scope):` e use o restante como uma
  linha de tarefa. Agrupe por scope só se isso deixar a lista mais legível (muitos
  commits do mesmo scope); não force agrupamento numa lista que já é plana e
  coerente.
- Não copie mensagens de commit temporárias/internas ao pé da letra (`wip`, `fix
  typo`, commits de merge) — funda esses casos na tarefa vizinha que descreve a
  mudança real. Se restar dúvida sobre o que um commit fez, `git show <sha>
  --stat` antes de descrever.

## Fase 4 — Montar título e corpo

- Título: `tipo: resumo curto e específico do resultado` (ex.: `fix: elimina
  no-explicit-any e no-unused-vars do back-end`). O resumo não é o nome da
  branch traduzido ao pé da letra — é uma frase informada pelos commits reais.
- Corpo em Markdown:
  ```
  ## Summary
  - <tarefa 1>
  - <tarefa 2>

  ## Test plan
  - [ ] <verificação 1>
  ```
- Test plan: derive do que é verificável dado o tipo de mudança — suíte de testes
  existente, ou passos manuais. Se a branch só tocou lint/tipos, `npm run lint` /
  `tsc --noEmit` limpos já são o test plan.

## Fase 5 — Push e criar o PR

- Antes de pushar, confira `git status`: se houver mudanças não commitadas, pare
  e avise o usuário — decidir o que entra em qual commit não é papel desta skill.
- `git push -u origin <branch>` (nunca `--force`) se a branch ainda não tem
  upstream ou está atrás/à frente do remoto.
- `gh pr create --title "<título>" --body "$(cat <<'EOF' ... EOF)"`, usando
  heredoc para preservar a formatação do corpo.
- Devolva a URL do PR ao usuário ao final.

## O que a skill não faz

- Não decide se o conteúdo do PR está correto — revisão final é do usuário.
- Não faz merge, não aprova review, não fecha issues — só cria o PR.
- Não depende de MCP: usa só `gh` via Bash, que já é a ferramenta padrão deste
  ambiente para operações de PR.
