---
name: bulk-refactor
description: Playbook para refactors mecânicos de larga escala que tocam dezenas de arquivos (limpar todos os erros de lint/tsc, apertar a tipagem de uma camada de API compartilhada, absorver um upgrade de dependência que quebrou tipos em cascata, aplicar um codemod). Estabelece os idiomas uma vez, então distribui agentes paralelos por domínio com posse exclusiva de arquivos, verifica continuamente, e fecha com UAT ao vivo + commits atômicos. Use quando um único `lint`/`tsc`/build reportar erros espalhados por dezenas de arquivos — não para correções isoladas em 1-2 arquivos. Aciona em "corrigir todos os erros de lint", "fix all lint errors", "refactor sistêmico", "codemod", "upgrade quebrou os tipos em todo lugar".
metadata:
  version: 1.0.0
  origin: extraído da sessão de fix-lint-errors (front-end/radar, 2026-08-17) — 515 problemas de lint → 0, ~90 arquivos, 12 commits atômicos
---

# Bulk Refactor

Corrigir um erro em 90 arquivos não é 90 correções — é uma correção repetida 90 vezes.
O trabalho real é achar o padrão, escrevê-lo uma vez, e fazer com que se repita
consistentemente sem 90 rodadas de revisão manual.

```
┌─────────┐   ┌───────────────┐   ┌──────────────┐   ┌────────────┐   ┌─────────┐
│ TRIAGEM │ → │ FUNDAÇÃO/base │ → │  IDIOMAS  1x │ → │ FAN-OUT    │ → │ FECHAR  │
└─────────┘   │  compartilhada│   │ (você mesmo) │   │ por domínio│   │ (UAT +  │
              └───────────────┘   └──────────────┘   └────────────┘   │ commits)│
                                                                       └─────────┘
```

## Quando usar

Um único comando (`npm run lint`, `tsc --noEmit`, um codemod, um `npm outdated`)
retorna uma lista de problemas espalhada por **dezenas de arquivos**, e boa parte
deles compartilha a mesma causa raiz ou o mesmo padrão de correção. Se o problema
está isolado em 1-2 arquivos, isso não é bulk refactor — é só uma correção normal.

Não use para: decisões de arquitetura, features novas, ou qualquer coisa onde a
"correção" exige julgamento de produto arquivo a arquivo (isso é o trabalho do
`tlc-spec-driven`, não deste skill).

## Fase 0 — Triagem: meça antes de cortar

Rode a ferramenta real (`npm run lint`, `tsc --noEmit`, etc.) e capture a saída
completa num arquivo no scratchpad — não confie em amostras. Depois:

1. Conte ocorrências por regra/tipo de erro (`grep -oE '<regra>' | sort | uniq -c | sort -rn`).
   Isso te diz onde está o volume real antes de você gastar tempo em um erro raro.
2. Liste os arquivos afetados. Esse é o universo que você vai particionar na Fase 3.
3. Estime: a regra mais comum é mecânica (renomear, adicionar tipo) ou exige
   entender o domínio (ex.: `no-explicit-any` numa camada de API)? A segunda
   categoria é onde a Fase 1 importa.

## Fase 1 — Fundação compartilhada primeiro, com terreno real (não fabricado)

Se os erros nascem de uma fronteira compartilhada (um cliente de API tipado frouxo,
um schema, um contrato entre front e back), **conserte essa fronteira primeiro**,
usando a verdade do outro lado — não invente tipos para fazer o compilador calar a
boca.

- Leia o schema/DTO/contrato real (Prisma schema, DTOs do NestJS, OpenAPI, o que
  existir) em vez de inferir só pelos usos no lado que você está arrumando.
- Se ler o outro lado é demorado, dispare um agente de pesquisa em paralelo
  (`Explore`/`general-purpose`) enquanto você mesmo already começa a consertar o
  que já sabe — não espere ocioso.
- **Espere a cascata.** Apertar a fundação vai revelar tipos locais duplicados e
  divergentes nos arquivos consumidores (é esperado, não é um erro seu). Rode o
  type-checker de novo depois da fundação para ver o tamanho real da cascata antes
  de decidir a profundidade do resto do trabalho — se a cascata for grande e mudar
  o escopo do que foi pedido, pare e confirme com o usuário antes de prosseguir
  (ver "Achados que mudam de escopo" abaixo).

## Fase 2 — Estabeleça os idiomas você mesmo, antes de paralelizar

Antes de despachar qualquer agente, conserte 2-3 arquivos representativos com as
suas próprias mãos. Isso não é desperdício — é onde os idiomas realmente aparecem:
o padrão de tratamento de erro que se repete, a regra de lint nova que pega todo
efeito de "buscar dados ao montar", o helper que vale a pena extrair.

Escreva os idiomas como **código literal**, não descrição vaga — um agente que lê
"trate erros direito" improvisa 8 jeitos diferentes; um agente que lê o snippet
exato replica.

Exemplo real desta sessão (TypeScript/React, mas o princípio vale para qualquer
stack): a regra nova `react-hooks/set-state-in-effect` (do eslint-plugin-react-hooks
v7, parte do conjunto de regras do React Compiler) passou a marcar como **erro**
qualquer `setState` síncrono dentro de um efeito — inclusive indireto, via função
chamada. Isso pegou o padrão onipresente "buscar ao montar":

```ts
// ANTES — dispara react-hooks/immutability (hoisting) E set-state-in-effect
useEffect(() => { loadX(); }, [dep]);
const loadX = async () => { setLoading(true); ...; setLoading(false); };

// DEPOIS — dois idiomas distintos, escolhidos por caso:

// (a) é sincronização legítima com sistema externo (API, localStorage) →
//     useCallback ANTES do efeito + disable comment justificado
const loadX = useCallback(async () => { ... }, [dep]);
useEffect(() => {
  // eslint-disable-next-line react-hooks/set-state-in-effect
  loadX();
}, [loadX]);

// (b) é estado derivado que só precisa resetar quando outro valor muda →
//     ajuste de estado DURANTE a renderização, sem efeito nenhum
const [prevDep, setPrevDep] = useState(dep);
if (dep !== prevDep) {
  setPrevDep(dep);
  setEstadoDerivado(valorInicial);
}
```

A escolha entre (a) e (b) é o julgamento que você faz nos 2-3 arquivos manuais —
os agentes recebem os dois padrões prontos e aplicam caso a caso, com a regra:
"quando em dúvida, prefira (a) — é mais seguro, preserva o comportamento exato".

Outro idioma típico: um helper único para o padrão de erro repetido
(`catch (error: any) { toast(error.message) }` → uma função `getErrorMessage(error, fallback)`
central), em vez de cada agente reinventar o `instanceof Error` na mão.

## Fase 3 — Fan-out com posse exclusiva e fronteiras explícitas

Particione os arquivos do universo (Fase 0) em grupos coerentes — por domínio de
negócio, não por tipo de arquivo. Cada grupo vira um agente (`Agent`, tipo
`general-purpose`), rodando em paralelo, com um prompt que inclui:

1. **O que já foi descoberto** — os idiomas da Fase 2 como código literal, e
   qualquer verdade de terreno já levantada na Fase 1 (contratos reais, tipos
   corretos) para que o agente não precise re-descobrir.
2. **Posse exclusiva de arquivos** — liste os arquivos exatos que o agente pode
   editar. Se o agente precisar mudar um arquivo compartilhado (a fundação da
   Fase 1, um tipo usado por outro grupo), ele **relata, não edita** — evita dois
   agentes paralelos pisando no mesmo arquivo.
3. **Disciplina de descoberta** (ver abaixo) — o que fazer se a correção revelar
   um bug de negócio pré-existente.
4. **Contrato de verificação** — o agente roda a ferramenta (lint/tsc) escopada
   aos seus arquivos antes de encerrar, e reporta limpo ou lista o que sobrou.
5. **Formato do relatório final** — peça um resumo curto (200-300 palavras):
   o que foi corrigido, e qualquer achado que precise da sua decisão.

Dispare todos os agentes do grupo em uma única mensagem (chamadas de tool em
paralelo) para que rodem concorrentemente. Enquanto esperam, você pode seguir
consertando arquivos que não couberam em nenhum grupo, ou revisando o que já
voltou.

## Disciplina de descoberta: não conserte bugs de negócio escondidos no meio

Um refactor mecânico frequentemente revela comportamento pré-existente que estava
mascarado (um `any` que escondia um campo que a API nunca retorna de verdade, um
`setState` que nunca era chamado, uma condição morta). A regra:

- **Preserve o comportamento atual** ao máximo, mesmo que pareça "errado" —
  a correção do bug de negócio é uma decisão do usuário, não sua, dentro do
  escopo de um refactor de lint/tipos.
- **Documente o achado** no relatório final (seu e de cada agente) em vez de
  silenciosamente misturar o fix de negócio com o fix de lint — isso deixa a
  decisão explícita e revisável.
- Exceção: se o achado for grande o suficiente para mudar a estimativa de esforço
  do trabalho todo (ex.: a fundação da Fase 1 exige reconciliar 10 tipos
  divergentes, não 1), pare e apresente a escolha ao usuário (fix raso e rápido
  vs. fix completo e correto) antes de despachar a Fase 3 — não decida sozinho a
  profundidade.

## Fase 4 — Fechar: verificação estática + build + UAT ao vivo

Depois que todos os agentes voltarem:

1. Rode a ferramenta original (lint/tsc) no projeto inteiro — não por arquivo.
   Agentes paralelos editando arquivos relacionados podem deixar arestas soltas
   que só aparecem na visão de projeto inteiro.
2. Rode o build de produção se existir (`next build`, `tsc -b`, etc.) — pega
   erros que lint/tsc isolado não pega (ex.: geração de páginas estáticas).
3. **UAT ao vivo, não só estático.** Um refactor grande o suficiente para tocar
   90 arquivos é grande o suficiente para ter quebrado algo em runtime que os
   checkers estáticos não veem (um efeito reescrito que para de disparar, uma
   condição de corrida). Suba a aplicação de verdade (dev server do front +
   back, se aplicável) e navegue pelos fluxos que os arquivos mais alterados
   afetam, checando o console do browser por erros — não declare pronto sem
   isso quando o projeto não tem um runner de componente/E2E automatizado.
4. Só então reporte "pronto" ao usuário, com a lista de achados de negócio
   preservados (não corrigidos) em destaque.

## Fase 5 — Commits atômicos

Não junte 90 arquivos num commit. Divida na mesma partição da Fase 1/3 (fundação
primeiro, depois um commit por grupo de domínio), na ordem em que as dependências
exigem (a fundação tem que vir antes dos consumidores). Antes de cada commit:

- `git add` só os arquivos daquele grupo — nunca `-A`/`.`.
- `git diff --cached --stat` para conferir que só entrou o que devia — arquivos
  parcialmente stageados antes da sessão (por um editor, por um hook) podem grudar
  no primeiro `git add` de um arquivo não relacionado; se aparecer algo que não é
  desse grupo, `git restore --staged <arquivo>` e deixe para o commit certo.
- Mensagem no padrão do repo (confira `git log --oneline -10` antes do primeiro
  commit), descrevendo o quê + por quê do grupo, não a lista de arquivos.
