---
name: pr-quality-gate
description: Verifica todos os quality gates de uma PR do radar e corrige o que estiver falhando. Use ao guiar uma PR até o estado de merge-ready.
---

# PR Quality Gate

Você está conduzindo uma pull request do radar até o estado de merge-ready. Faça exatamente uma passada.

## Determinar o número da PR

- Se um número de PR foi informado explicitamente ao invocar a skill (`$ARGUMENTS`), use-o.
- Caso contrário, descubra a PR associada à branch atual: `gh pr view --json number --jq .number`.
  Se não houver PR aberta para a branch atual e nenhum número foi informado, pare e peça o número ao usuário.

## Escopo: descubra o que a PR toca

O radar é um monorepo (`front-end/` com Next.js + Vitest + ESLint, `back-end/` com NestJS + Jest + ESLint). Rode:

- `gh pr diff <número-da-pr> --name-only` para saber se a PR toca `front-end/`, `back-end/`, ou ambos.
- Rode os gates abaixo SOMENTE nos workspaces tocados pela PR.

## Sensores: levante o estado atual primeiro

- `gh pr checks <número-da-pr>` — status dos checks do workflow `pr-checks.yml` (lint, type-check, testes, cobertura).
- `gh pr view <número-da-pr> --json statusCheckRollup,mergeable` — rollup geral.
- Se algum check não estiver disponível ou você precisar depurar localmente, reproduza sem `--fix` (para sensoriar sem alterar código):
  - Front-end: `cd front-end && npx eslint . && npx tsc --noEmit && npm run test:unit`
  - Back-end: `cd back-end && npx eslint "{src,apps,libs,test}/**/*.ts" && npx tsc --noEmit -p tsconfig.build.json && npm run test:cov`

## Critério de sucesso (o objetivo)

A PR está pronta para merge SOMENTE quando TODOS os itens abaixo forem verdadeiros, para cada workspace tocado:

- Todo check de CI do workflow `pr-checks.yml` está passando.
- ESLint reporta zero erros e zero warnings.
- `tsc --noEmit` não reporta nenhum erro de tipo.
- Toda suíte de testes existente passa (Vitest no front-end, Jest no back-end).
- Cobertura é medida e reportada pelo CI; ela hoje não bloqueia a PR (sem baseline suficiente para travar um threshold) — apenas confira que o número não é uma queda óbvia em relação ao histórico recente.

Se todos forem verdadeiros, poste um comentário `✅ Todos os gates verdes — pronta para merge` e PARE. Não faça merge da PR você mesmo.

## Ação: corrija exatamente o que está falhando

Para cada gate falhando, faça a menor mudança correta que resolva a causa raiz, então commit com mensagem no padrão conventional-commit e push para a branch da PR.

## Guardrails — inegociáveis

- NUNCA faça merge, aprove ou feche a PR.
- NUNCA force-push ou reescreva o histórico.
- NUNCA desabilite, pule ou apague um teste para fazer um gate passar.
- NUNCA toque em arquivos fora do diff dessa PR.
- Se uma falha for ambígua ou você não tiver confiança na correção, poste um comentário explicando o motivo e PARE para um humano.
