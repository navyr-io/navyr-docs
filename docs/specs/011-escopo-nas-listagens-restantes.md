# SPEC-011 — As listagens de escopo que a SPEC-008 não converteu

**Estado:** implementada e verificada — 28/08
**Card:** navyr-io/navyr-orchestrator#15
**Continua:** [SPEC-008](008-namespace-vazio.md)

## Problema

A SPEC-008 fez `namespace=` vazio significar "todos os namespaces" e corrigiu a
fronteira de autorização, mas converteu 17 das 33 funções de escopo. As 16
restantes seguem trocando vazio por `"default"`.

O card as agrupou em quatro categorias por suposição. Verificadas uma a uma, a
divisão muda.

## O que a verificação mostrou

**5 supostamente mal classificadas — confirmadas como objeto nomeado.**
`GetWorkloadContext`, `GetWorkloadLogs`, `GetWorkloadHints`, `GetWorkloadTimeline`
e `GetWorkloadEvents` chamam `parseWorkloadPath`, que extrai `kind` e `name` do
caminho. O classificador da SPEC-008 não conhecia essa função e por isso as pôs em
escopo.

São de objeto nomeado, e `default` é defensável — é o comportamento do `kubectl`
sem `-n`, e o mesmo dado às outras 46. **Ficam como estão.**

**2 escritas — `default` é o comportamento convencional.**
`ClusterApply` aplica um manifesto; `CreateSecuritySectionResource` cria um
recurso. "Criar em todos os namespaces" não tem significado, e `kubectl apply` sem
`-n` também usa o namespace corrente. **Ficam como estão.**

**7 listagens — precisam de conversão individual.**

| Função | Chama |
|---|---|
| `ListEvents` | `ListEventsFiltered` |
| `ListCorrelatedEvents` | `ListCorrelatedEvents` |
| `ListObservabilityAlerts` | `ListEventsFiltered` |
| `GenerateIncidentCopilotPlan` | `ListCorrelatedEvents` |
| `ListSecuritySection` | `ListGenericResources` |
| `GetTopologyGraph` | `ListGenericResources` |
| `GetObservabilityUnified` | cinco listagens diferentes |

Não bateram o formato canônico porque os serviços recebem parâmetros além do
namespace. `listarNoEscopo` já resolve isso com closure — foi assim que
`GetSecurityConfigRisk` foi convertida na SPEC-008.

`GetTopologyGraph` é a de maior alcance: um grafo de topologia que cobre um
namespace e se apresenta como o do cluster.

**2 streams — decisão pendente.** `WatchEvents` e `WatchPods`.

## O detalhe que a conversão introduz

`ListEventsFiltered` e `ListCorrelatedEvents` recebem `limit`. Percorrendo N
namespaces, aplicar o limite a cada um devolve até N×limit resultados.

Só acontece quando há restrição — sem allowlist e sem escopo é uma chamada só, e o
limite vale uma vez. Ainda assim, o resultado precisa ser truncado ao limite
pedido depois da união, senão o contrato do parâmetro deixa de valer justamente
para os clusters restritos.

## Regras

**R1 — As 7 listagens passam a usar `listarNoEscopo`.**

**R2 — Onde há `limit`, o resultado unido é truncado ao limite pedido.**
Consequência direta de percorrer N namespaces. Sem isso, um cluster restrito a 5
namespaces devolveria até 5× o que foi pedido.

**R3 — As 5 de workload e as 2 escritas ficam como estão, e isso é registrado.**
Não é omissão: `default` é o comportamento correto para objeto nomeado e para
escrita. Registrar evita que a próxima varredura as trate como pendência.

**R4 — Existe teste do truncamento com escopo restrito.**
É o caso que só aparece com allowlist povoada, e a allowlist é `[]` em todo cluster
hoje — sem teste, R2 não teria como reprovar.

## Critérios de aceitação

1. `/api/v1/events?namespace=` devolve eventos de todos os namespaces.
2. `GetTopologyGraph` sem namespace cobre o cluster inteiro.
3. Com `allowed_namespaces` povoado, as 7 devolvem só os namespaces permitidos.
4. Com `limit=N` e escopo de vários namespaces, o resultado tem no máximo N itens.
5. As 5 de workload e as 2 escritas seguem com `default`, sem alteração.

## Decisão tomada

**D1 — Os streams viram card próprio.** *(28/08)*

Multiplexar N watches numa saída só é código concorrente com encerramento
coordenado e backpressure. Misturar isso com substituição de chamada no mesmo
commit é como se perde a revisão — e um vazamento de goroutine só aparece em
produção sob carga.

## Resultado

Das 16 do card: **7 convertidas**, 5 confirmadas como objeto nomeado, 2 escritas
mantidas, 2 streams em card próprio.

Nenhuma função de escopo restante troca vazio por `default`.

Medido na stack, com e sem allowlist:

| | cluster-wide | `default` | allowlist `[kube-system]` |
|---|---|---|---|
| topologia (nós) | 16 | 2 | 11 |
| pods | 10 | 0 | 8 |

Um grafo de topologia que cobria 2 nós e se apresentava como o do cluster passa a
cobrir 16 — e respeita a allowlist quando ela existe.
