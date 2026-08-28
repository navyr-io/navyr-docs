# SPEC-008 — "All namespaces" devolve só o `default`

**Estado:** parcialmente implementada — critérios 1 a 6 verificados em 27/08
**Card:** navyr-io/navyr-orchestrator#11

## Problema

Duas telas do produto se contradizem sobre o mesmo cluster, no mesmo instante.

| Onde | Mostra |
|---|---|
| Cockpit | **10 pods**, health 100% |
| Tela de Pods, "All namespaces" (padrão) | **"No pods"**, contadores em zero |

`kubectl get pods -A` confirma 10. A tela de Pods é a errada.

## Causa

`internal/handler/kubernetes_pods.go:29-32`:

```go
namespace := r.URL.Query().Get("namespace")
if namespace == "" {
    namespace = "default"
}
```

A UI manda `?namespace=` para dizer "todos" — a convenção do `client-go`, onde
`Pods("").List()` percorre o cluster. O handler troca por `default`, que no kind
tem 0 pods.

Medido, mesma sessão:
```
/api/v1/pods?namespace=              → null
/api/v1/pods?namespace=kube-system   → [coredns…, etcd…, …]
```

**A substituição aparece 80 vezes em 13 arquivos.** Não é só pods: deployments,
services, ingresses, configmaps, PVCs — e `security_scan.go`, em dois pontos.

Um scan de segurança que varre só o `default` e reporta como se tivesse varrido o
cluster é a versão cara deste defeito.

## Três camadas com três opiniões sobre o que vazio significa

1. **O handler** troca por `default`.
2. **`serviceForRequest`** (`kubernetes_rede.go:298`) trata vazio como caso
   legítimo e pula a validação.
3. **`validateNamespace`** no cliente (`internal/service/kubernetes.go:52`)
   rejeita vazio com `"namespace is required"`.

Como a camada 1 age primeiro, as outras duas nunca veem vazio.

## A descoberta que define a decisão

**As duas guardas de autorização liberam quando o namespace é vazio.**

`kubernetes_cluster.go:579` e `:592`:

```go
func namespaceAllowed(cluster *models.Cluster, namespace string) bool {
    if cluster == nil || namespace == "" || len(cluster.AllowedNamespaces) == 0 {
        return true
    }
```
```go
func namespaceInRequestScope(r *http.Request, namespace string) bool {
    if namespace == "" {
        return true
    }
```

Elas rodam em `serviceForRequest`, **depois** de o handler ter substituído por
`default`. Hoje recebem `"default"` e checam corretamente.

Se vazio passar a significar "todos", as duas passam a liberar exatamente a
requisição que lê o cluster inteiro. Um cluster restrito a `["staging"]`
devolveria todos os namespaces.

**A substituição por `default` é, por acidente, o que sustenta a autorização.**

### Quanto disso é risco hoje

Nenhum, e é importante ser preciso: as duas guardas já estão inertes.

- `allowed_namespaces` é `[]` em todos os clusters — medido no banco. Com lista
  vazia, `namespaceAllowed` devolve `true` de qualquer forma.
- `X-Scope-Namespaces` **nunca é emitido**. Varredura nos 8 repositórios: o
  orquestrador lê o cabeçalho, e ninguém o escreve. Com `raw == ""`,
  `namespaceInRequestScope` devolve `true`.

Ou seja, o buraco não se abre hoje — ele se abre no dia em que alguém povoar a
allowlist ou ligar o escopo por usuário, e nesse dia não haverá nada indicando que
a leitura cluster-wide escapa das duas.

## Segundo defeito, independente

`kubernetes_pods.go:44`:
```go
var podList []models.Pod        // nil quando não há pods
json.NewEncoder(w).Encode(pods) // nil slice → `null`
```

O contrato de uma coleção vazia é `[]`. `null` obriga todo consumidor a tratar dois
casos, e quem não trata renderiza vazio em silêncio. Vale para todos os handlers
que acumulam em slice declarado com `var`.

## Decisões tomadas

**D1 — Vazio significa "todos os namespaces", e as guardas param de liberar.**
*(27/08)* Vira R1 a R4.

A alternativa de recusar vazio com 400 foi descartada por quebrar todo chamador
atual — inclusive a própria interface, em várias telas — sem ganho de segurança
sobre a opção escolhida, já que as guardas seriam corrigidas de qualquer forma.

**D2 — Toda lista devolve `[]`.** *(27/08)* Vira R5.

## Os 80 pontos não são uma coisa só

Classificados pelo que o handler faz, e não pelo nome:

| Natureza | Quantas | O que vazio deve significar |
|---|---|---|
| **Escopo** — lista ou analisa sobre um conjunto | **33** | todos os namespaces permitidos |
| **Objeto nomeado** — lê, altera ou apaga um recurso específico | **46** | segue `default` |

As 33 de escopo incluem `ListPods`, `ListDeployments`, `ListSecrets`,
`GetSecurityConfigRisk`, `GetTopologyGraph` e `WatchEvents`.

**As 46 de objeto nomeado ficam como estão, deliberadamente.** `GetPod` sem
namespace não tem resposta melhor que `default`, e é o mesmo comportamento do
`kubectl` sem `-n`. Mudar isso seria decisão própria, não consequência desta.

## Regras

**R1 — Nas 33 funções de escopo, vazio deixa de virar `default`.**
O valor segue vazio até a camada que sabe resolvê-lo.

**R2 — `validateNamespace` do cliente aceita vazio.**
Hoje rejeita com `"namespace is required"`, o que tornaria o caminho inalcançável.
Vazio passa a ser válido; qualquer valor não-vazio continua submetido ao regex.

**R3 — As guardas param de liberar com vazio, e passam a resolver o escopo.**
`namespaceAllowed` e `namespaceInRequestScope` recebem vazio hoje como "sem
restrição". Passam a devolver o **conjunto efetivo**: sem allowlist e sem escopo,
o conjunto é "tudo" e a leitura segue cluster-wide; com allowlist ou escopo, a
leitura é feita namespace a namespace sobre o conjunto permitido, e os resultados
são unidos.

É a diferença entre "você não pediu namespace, então pode tudo" e "você não pediu
namespace, então pode o que lhe cabe".

**R4 — Existe teste negativo com allowlist povoada.**
A allowlist é `[]` em todos os clusters hoje e o escopo por usuário nunca é
emitido — as duas guardas estão inertes. Sem um teste que popule a allowlist, R3
não teria como reprovar, e a regressão passaria despercebida exatamente como
passaria em produção.

**R5 — Toda lista serializa `[]`, nunca `null`.**
Handlers que acumulam em slice declarado com `var` devolvem `null` quando vazio.
Isso obriga cada consumidor a tratar dois casos, e quem não trata renderiza vazio
em silêncio — que é parte do sintoma que originou esta spec.

## Critérios de aceitação

1. `/api/v1/pods?namespace=` devolve os pods de todos os namespaces. Hoje devolve `null`.
2. `/api/v1/pods?namespace=kube-system` continua devolvendo só aquele namespace.
3. Cockpit e tela de Pods concordam na contagem, na mesma sessão.
4. Com `allowed_namespaces` povoado, a leitura cluster-wide devolve **só** os
   namespaces da lista — verificável nos dois sentidos.
5. Lista vazia serializa `[]` em todos os endpoints de coleção.
6. As 46 funções de objeto nomeado seguem com `default`, sem alteração.

## O que foi entregue

17 das 33 funções de escopo, escolhidas pelo alcance: `ListPods`, os 6 de
workloads, `ListServices`/`ListEndpoints`/`ListIngresses`/`ListGateways`,
`ListConfigMaps`/`ListSecrets`, os 2 de storage, `ListNetworkPolicies` e
`GetSecurityConfigRisk`.

O efeito medido no scan de configuração, que motivou a prioridade deste item:

| Escopo | Achados | Namespaces | Danger |
|---|---|---|---|
| cluster-wide (vazio) | **10** | kube-system, local-path-storage, navyr-system | **3** |
| `default` — o que ele varria antes | **0** | — | 0 |

Um operador lia "nenhum achado" e concluía que a postura estava limpa.

**16 funções de escopo seguem sem conversão**, rastreadas em card próprio:

- `WatchEvents` e `WatchPods` — streams. Escopo restrito exigiria multiplexar N
  watches numa saída só, com o encerramento de cada um. Trabalho próprio.
- `ClusterApply` e `CreateSecuritySectionResource` — escritas. "Aplicar em todos
  os namespaces" não é o mesmo problema que "ler de todos".
- `GetWorkloadContext`, `GetWorkloadLogs`, `GetWorkloadHints`, `GetWorkloadTimeline`,
  `GetWorkloadEvents` — recebem o nome do workload por query, não por path, então
  o classificador as pôs em escopo; na prática são de objeto nomeado.
- `ListEvents`, `ListCorrelatedEvents`, `ListObservabilityAlerts`,
  `GetObservabilityUnified`, `GenerateIncidentCopilotPlan`, `ListSecuritySection`,
  `GetTopologyGraph` — formatos que não batem o canônico e precisam de conversão
  individual.

## Fora de escopo

- Ligar o escopo por usuário (`X-Scope-Namespaces`) de ponta a ponta. É
  funcionalidade, não correção — mas esta spec não pode deixá-la mais difícil,
  e R3 existe para isso.
- As 46 funções de objeto nomeado.
