# SPEC-016 — `manifests/apply` valida e não aplica

**Card:** `navyr-io/navyr-orchestrator#19` · **Severidade:** Crítico
**Descoberto:** varredura de interface ponta a ponta, Fase 4 (28/08/2026)

## O defeito

`POST /api/v1/manifests/apply` valida o manifesto, responde `202 accepted` com
`mode: "queued"` e a mensagem `"manifest accepted for safe apply pipeline"`, e
retorna. É a última linha da função.

`internal/handler/manifest_handler.go:78-83`.

Não existe fila. Varredura por `manifestQueue`, `applyQueue`, `pendingManifest` e
`safeApply` em todo o `navyr-orchestrator`: zero ocorrências. A string
*"safe apply pipeline"* só aparece nessa mensagem.

### Medido

```
POST /api/v1/manifests/apply  {"manifest":"…kind: Namespace… criado-pela-varredura"}
→ 202 {"accepted":true,"mode":"queued","message":"manifest accepted for safe apply pipeline"}

$ kubectl get ns criado-pela-varredura      (após 60s)
não existe

Pela tela de Namespaces, botão "+ Create":
  namespaces antes:  11
  namespaces depois: 11
  formulário fechou: sim     apareceu erro: não
```

### Quatro funcionalidades da interface dependem deste endpoint

| Tela | Arquivo | O que promete |
|---|---|---|
| Namespaces | `NamespacesPage.tsx:61` | criar namespace |
| Workloads | `WorkloadsPage.tsx:396` | implantar imagem/manifesto |
| Workload → YAML | `WorkloadDetailPage.tsx:216` | botão **Apply** do editor |
| Estado vazio de métricas | `MetricsServerEmptyState.tsx:44` | instalar metrics-server |

Todas relatam sucesso. Nenhuma altera o cluster.

## O que já está correto, e não muda

A camada de autorização **não é o defeito** e é reaproveitada inteira:

- O gateway classifica `manifests/apply` como feature `rollout`
  (`navyr-gateway/cmd/server/rotas.go:293`) — o gate de papel já se aplica.
- O gateway exige `X-Action-Reason` para esta rota
  (`despacho.go:715`), tratando-a como ação crítica de mutação.
- O ClusterRole do agente concede `get,list,watch` — e **só isso** — em
  `roles`, `rolebindings`, `clusterroles` e `clusterrolebindings`
  (`internal/agentmanifest/manifest.go:95`). Um manifesto não consegue escalar
  privilégio criando binding, porque o agente não pode criar binding. Essa é a
  fronteira final e ela permanece de pé.

O caminho de escrita no cluster também já existe: `CreateGenericResource`,
`UpdateGenericResource` e `DeleteGenericResource` usam o cliente dinâmico sobre o
túnel do agente (`internal/service/kubernetes.go:1703`). O `Apply` simplesmente
não o usa.

## Decisões

**D1 — Como aplicar: Server-Side Apply.**
`dyn.Resource(gvr).Apply(...)` com `FieldManager: "navyr-orchestrator"` e
`Force: false`. Idempotente, sem leitura-modificação-escrita (que abre corrida), e
um campo de posse de outro controlador produz **conflito explícito** em vez de ser
roubado em silêncio. `Force: false` é a escolha segura: o operador recebe um erro
acionável e decide, em vez de a plataforma decidir por ele.

**D2 — Kind → GVR: RESTMapper de descoberta.**
`restmapper.GetAPIGroupResources` sobre o cliente de descoberta do próprio túnel.
Custa uma ida de descoberta por aplicação; aceitável agora, memoizável depois.
Não há tabela cravada de kinds — CRDs do cliente funcionam como os tipos nativos.

**D3 — Multi-documento: aceito.**
Manifesto com vários documentos separados por `---` é o caso normal. Cada
documento é validado e aplicado em ordem. A aplicação **para no primeiro erro**, e
o que já foi aplicado **é reportado como aplicado**. Aplicação parcial é um risco
real e será dita, não escondida.

**D4 — Escopo de namespace: a regra que faltava.**
Hoje a validação olha só o campo `namespace` da requisição, e ignora o
`metadata.namespace` de dentro do YAML. Um manifesto pode declarar
`namespace: kube-system` enquanto a requisição diz `producao`, e escapar de
`allowed_namespaces`. **Este é o furo de segurança que a correção fecha.**

**D5 — Recursos de escopo de cluster.**
Permitidos apenas quando o cluster **não tem** `allowed_namespaces` definido. Num
cluster com allowlist, criar recurso fora de namespace é, por definição, criar
fora da allowlist — e é recusado com mensagem explícita.

**D6 — A resposta para de mentir.**
`mode: "queued"` e *"accepted for safe apply pipeline"* saem. Entra um relatório
por documento: kind, nome, namespace e operação (`created` / `configured` /
`unchanged`). Sucesso total responde `200`; falha de validação `422`; conflito de
posse `409`; recusa do servidor de API `403`, com a mensagem do Kubernetes
repassada literalmente.

## Regras

- **R1** — `Apply` aplica de fato, por Server-Side Apply, com field manager
  `navyr-orchestrator` e `Force: false`.
- **R2** — A autorização é a **mesma** de qualquer outra operação de cluster: o
  handler reusa `serviceForRequest`, e não reimplementa a guarda. Uma segunda
  implementação da guarda é uma segunda chance de errar.
- **R3** — Manifesto multi-documento é aplicado em ordem, parando no primeiro
  erro; o relatório distingue aplicado, falho e não tentado.
- **R4** — Para recurso com namespace: `metadata.namespace` vazio é preenchido com
  o namespace validado; `metadata.namespace` preenchido **precisa ser igual** ao
  namespace validado, ou o documento é recusado.
- **R5** — Recurso de escopo de cluster só é aceito quando
  `len(cluster.AllowedNamespaces) == 0`.
- **R6** — Erro do servidor de API (`Forbidden`, `Conflict`, `Invalid`) é
  repassado com a mensagem original. A plataforma não substitui o diagnóstico do
  Kubernetes por um erro genérico próprio — foi exatamente isso que produziu o
  `navyr-gateway#22`.
- **R7** — Nenhuma resposta afirma que algo foi aplicado sem que a aplicação tenha
  retornado sucesso do servidor de API.

## Consequência conhecida, e por que não é regressão

`MetricsServerEmptyState` instala o metrics-server, cujo manifesto inclui
`ClusterRole`, `ClusterRoleBinding` e `APIService`. O agente não pode criar
nenhum dos três. Depois desta correção essa tela passará a **falhar com
`403 Forbidden` e a mensagem do Kubernetes**, em vez de relatar sucesso.

Isso é melhora, não regressão: hoje ela mente. Ampliar o ClusterRole do agente
para cobrir esse caso daria a ele poder de criar binding de cluster — ou seja,
poder de se auto-promover — e é precisamente a fronteira que D-nenhum deve
atravessar. Fica cardado à parte.

## Critérios de aceitação

Verificados contra a stack viva, não contra mock.

1. Criar namespace pela tela de Namespaces **muda a contagem** de namespaces no
   cluster.
2. Manifesto cujo `metadata.namespace` diverge do namespace da requisição é
   **recusado**, com mensagem que nomeia os dois.
3. Manifesto de escopo de cluster em cluster com `allowed_namespaces` é recusado.
4. Aplicar o **mesmo** manifesto duas vezes responde `configured`/`unchanged` na
   segunda, e não erro — a idempotência do Server-Side Apply.
5. Manifesto multi-documento com o segundo documento inválido reporta o primeiro
   como aplicado e o segundo como falho.
6. Nenhuma resposta contém a string `safe apply pipeline`.
7. Prova negativa: com o `return` original restaurado, o teste de aceitação 1
   falha.
