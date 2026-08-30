# SPEC-025 — Suíte de testes contra a stack real

**Card:** `navyr-io/navyr-frontend#32` · **Severidade:** Médio
**Motivada por:** varredura de interface ponta a ponta (28–29/08/2026)

## Correção de duas afirmações do card

O card foi escrito a partir das notas da varredura e não reverificado. Medido
agora:

**❌ "4 dos 7 specs estão quebrados."** Falso — `npx playwright test`:
`22 passed (1.1m)`. Nenhum quebrado.

**❌ "o CI os roda com `continue-on-error`."** Falso. O `ci.yml` chama
`navyr-io/.github/.github/workflows/frontend.yml`, cujo job `e2e` roda
`npx playwright test` **sem** `continue-on-error`, e o comentário do próprio
workflow diz: *"Bloqueante desde 19/08 … Uma quebra aqui é regressão de verdade
e deve falhar o run."* Alguém já tinha corrigido isso.

**✅ "todos mockam a rede."** Verdadeiro, e é o que importa.

| Spec | `page.route` |
|---|---|
| `caminhos-infelizes` | 17 |
| `sprint-a-worker-flows` | 14 |
| `m6-flow` | 12 |
| `dashboard-incidents-flow` | 7 |
| `p3-enterprise-flows` | 6 |
| `sso-landing` | 6 |
| `monaco-sem-cdn` | 5 |

**67 interceptações, e nenhum spec fala com a stack real** — `grep` por
`localhost:5173` em `tests/e2e/`: zero.

Esses specs são bons no que fazem: caminhos infelizes, expiração de sessão,
pouso de SSO, ausência de CDN. Não é errado tê-los. Mas por construção não
detectam tela que promete o que o backend não entrega — a resposta vem do
próprio teste.

## O que esta spec entrega

Uma segunda suíte, **sem mock nenhum**, contra a stack de demonstração, que
guarda as onze correções desta rodada.

- `playwright.stack.config.ts` — separada da configuração de unidade. Sem
  `webServer`: a stack precisa estar de pé. `workers: 1`, porque os testes
  tocam o mesmo cluster. `retries: 0`, porque teste que só passa na segunda
  tentativa está escondendo instabilidade, e instabilidade contra stack real é
  achado, não ruído.
- `tests/stack/` — 14 testes cobrindo `orchestrator#19`, `#18`, `#17`,
  `frontend#24`, `#28`, `#25`, `#26`, `#27`, `#30` e `gateway#21`, `#22`.
- `npm run test:stack`.

Não entra no CI de unidade, que não tem stack. Endereços e credenciais vêm do
ambiente, com padrão apontando para o `navyr-deploy`.

## Três armadilhas encontradas ao construir, e o que ensinam

**1. O fixture `request` tem jar de cookie próprio.**
Autenticar com ele e navegar com `page` deixa a página deslogada. O sintoma é
obscuro: a rota redireciona para `/login` e o teste morre esperando por `main`.
Em teste que usa `page`, é `page.request`.

**2. `networkidle` nunca chega.**
A aplicação mantém stream aberto (SSE do feed ao vivo), então a rede jamais fica
ociosa e a espera consome o timeout inteiro. O sintoma foi **toda** medição de
tela falhando por tempo, com a página na verdade já renderizada — e a suíte
levando 7,3 min em vez de 46 s. Trocado por `domcontentloaded` mais uma espera
por conteúdo, com teto.

**3. Contexto de navegador novo não tem cluster selecionado.**
Telas de escopo de organização leem `navyr.cluster_id` do `localStorage`. Sem
semear, elas renderizam o estado de "nenhum cluster" e o teste procura um botão
que não está lá. `addInitScript` roda antes de qualquer script da página, que é
o único momento em que semear funciona.

## Regras

- **R1** — Nenhum `page.route` em `tests/stack/`. Um mock aqui derrota o
  propósito da suíte.
- **R2** — Sem `test.skip` condicional que se auto-pule por não achar elemento:
  um teste que se pula sozinho esconde regressão em silêncio. Se o elemento
  pode não existir, isso é a asserção.
- **R3** — Cada teste nomeia, no comentário, o card que ele guarda e o que foi
  medido antes da correção.
- **R4** — A suíte é provada reintroduzindo um defeito real e confirmando que
  ela falha.

## Prova negativa — a suíte pega o crítico

Reintroduzido o `navyr-orchestrator#19` (o `apply` volta a validar e retornar
`202 queued` sem aplicar), imagem reconstruída e stack recriada:

```
✘ manifests/apply aplica de verdade, e o recurso volta na leitura
✘ manifesto que declara outro namespace é recusado nomeando os dois
✓ eventos sem filtro devolvem o cluster inteiro
✓ os limites de plano da tela vêm do backend
✓ observabilidade alcança o Prometheus pelo túnel do agente
✓ o risk_score cresce com a gravidade
✓ webhook com URL que não resolve devolve o erro de validação
2 failed | 5 passed
```

Os dois que deviam falhar falharam; os cinco sem relação passaram. **A suíte com
mock não teria notado nada** — ela não chama o endpoint.

Restaurado: `14 passed (46.0s)`.

## Critérios de aceitação

1. `npm run test:stack` roda 14 testes verdes contra a stack de demonstração.
2. Nenhum `page.route` em `tests/stack/`.
3. Nenhum teste se pula sozinho.
4. Prova negativa acima.
