# SPEC-038 — Cobertura do túnel do agente

**Estado:** proposta
**Data:** 19/09/2026
**Card:** navyr-io/navyr-deploy#41
**ADR:** 0001 (agent tunnel), 0008 (autorização fail-closed)
**Relacionada:** SPEC-036 (comportamento do caminho Helm), navyr-deploy#39 (rota unificada), navyr-deploy#37 (Registry em memória)

## 1. Problema

```
$ grep -cE 'agent|tunnel|/agent/tunnel' tests/e2e/test_e2e_full_stack.sh
0
```

Zero. O túnel do agente — o WebSocket por onde **toda ação em cluster** acontece — não é exercitado por suíte nenhuma: nem e2e, nem functional, nem caos.

### 1.1 O que fica sem cobertura

O agente disca `{orchestratorURL}/api/v1/clusters/{id}/agent/tunnel` (`navyr-agent/cmd/executor/main.go:181`). A partir dali, é por esse canal que passam:

- listagem e inspeção de workloads, pods, rede e RBAC;
- `exec` interativo em pod;
- stream de log;
- escala e demais ações de escrita;
- o próprio estado "cluster conectado" que a interface mostra.

Cinco handlers devolvem `agent not connected` quando o canal não está no processo que recebeu a requisição.

### 1.2 Por que a ausência é mais perigosa aqui

O túnel é **bidirecional, de vida longa e sensível a intermediário**. Só de análise estática, em dois dias, ele produziu três achados:

| Achado | Card |
|---|---|
| o `Connection ""` do nginx impedia o upgrade, e por isso o compose ia direto ao orchestrator | `#39` |
| `idle_timeout` padrão de 60s no ALB derrubaria o túnel em laço | `#17` |
| `tunnel.Registry` em memória impede segunda réplica — e **2 é o default do chart** | `#37` |

Nenhum dos três apareceria num teste de requisição HTTP comum. E nenhum é pego por catraca estrutural: o contrato não descreve comportamento de conexão.

## 2. O que existe para construir sobre

A SPEC-036 entregou o que faltava de infraestrutura: `tests/k8s/subir-em-kind.sh` sobe o chart num cluster descartável, e as suítes deixaram de saber contra qual empacotamento rodam.

A superfície de API necessária existe e foi conferida em 19/09:

| Rota | Uso no teste |
|---|---|
| `POST /auth/register` · `POST /auth/login` | sessão, como o e2e já faz |
| `POST /api/v1/clusters` | registra o cluster |
| `POST /api/v1/clusters/{id}/agent/token` | token do agente |
| `GET /api/v1/clusters/{id}/agent/manifest?token=…` | manifesto para `kubectl apply` — rota pública, autenticada pelo token |
| `GET /api/v1/clusters/{id}/agent/session` | estado da sessão do agente |
| `WS /api/v1/clusters/{id}/agent/tunnel` | o túnel |

E `tunnel.Registry` expõe `IsConnected` e `ConnectedIDs`.

## 3. Regras e critérios de aceitação

Um teste em `tests/k8s/tunel-do-agente.sh`, rodando contra a plataforma que o harness sobe, com **dois clusters kind**: um hospeda a plataforma, outro recebe o agente. Dois e não um, porque o caso real é o agente estar em cluster que não é o da plataforma — e é isso que o caminho ECS vende.

### 3.1 Os cinco itens, em ordem de custo

1. **o túnel conecta** e a plataforma reporta o cluster como conectado;
2. **uma ação de leitura atravessa** o túnel e devolve dado real do cluster do agente;
3. **`exec` interativo** troca bytes nos dois sentidos;
4. **stream de log** entrega linhas por mais tempo que o `idle_timeout` da borda — é o item que pegaria a regressão dos 60s;
5. **derrubar e resubir o orchestrator** faz o agente reconectar sozinho.

O item 5 é o que dá base à decisão de `desired_count = 1` no ECS: aquela escolha se apoia em o agente reconectar, e isso nunca foi verificado.

## 4. Fora de escopo

**Dublê de agente.** O teste não deve usar um. O agente real é o que exercita a reconexão, o formato de quadro e o comportamento sob intermediário — que é o que produziu os três achados da § 1.2.

**Corrigir os três achados que o teste vai expor.** `navyr-deploy#39` (rota unificada), `navyr-deploy#37` (Registry em memória) e o `idle_timeout` do ALB (`navyr-deploy#17`) têm cards próprios. Esta spec entrega a **medição**; a correção é de cada card. Em particular, o item 2 com `replicaCount.orchestrator: 2` deve **reprovar** — é assim que o `#37` deixa de ser argumento e vira número.

**Cobertura do túnel no caminho ECS.** O harness da SPEC-036 sobe kind. ECS é `navyr-deploy#17`.

**Substituir o e2e existente.** Esta suíte acrescenta; não reescreve `tests/e2e/`.

## 5. Riscos

**Tempo.** Dois clusters kind mais a plataforma passa de dez minutos. Mitigação: os itens 1 e 2 num job, os 3 a 5 num job separado, disparável sob demanda.

**Flakiness.** É teste de rede de vida longa; o `#35` mostra o custo de um gate instável. Mitigação: esperar por condição observável — `IsConnected` — em vez de `sleep`, e falhar com o estado do cluster impresso.

## 6. Plano de execução

| Ciclo | Entrega | Critério de pronto |
| --- | --- | --- |
| 1 | Harness de dois clusters kind: um hospeda a plataforma pelo `subir-em-kind.sh` da SPEC-036, o outro recebe o agente | `kubectl --context kind-navyr-plat get pods -n navyr` e `kubectl --context kind-navyr-agente get ns` respondem, com a saída colada |
| 2 | Itens 1 e 2 — o túnel conecta, e uma leitura atravessa | `tunel-do-agente.sh 1 2` sai 0; `GET /api/v1/clusters/{id}/agent/session` reporta conectado; a listagem devolve um namespace que só existe no cluster do agente |
| 3 | Mordida dos itens 1 e 2 | com `Connection ""` no nginx, o item 1 reprova; com `replicaCount.orchestrator: 2`, o item 2 reprova de forma intermitente — ambas as saídas coladas |
| 4 | Itens 3 e 4 — `exec` interativo e stream de log além do `idle_timeout` | `tunel-do-agente.sh 3 4` sai 0, com o log correndo por mais de 60s |
| 5 | Mordida do item 4 | `idle_timeout` curto na borda reprova o item 4, com a saída colada |
| 6 | Item 5 — derrubar e resubir o orchestrator, o agente reconecta sozinho | `tunel-do-agente.sh 5` sai 0; `IsConnected` volta a `true` sem intervenção |
| 7 | Dois jobs de CI: itens 1–2 no gate, itens 3–5 em `workflow_dispatch` | run verde no SHA da `main`, e o segundo job disparado à mão com saída |

Cada ciclo termina em commit semântico com a suíte verde e nota em
`038-ciclos/cycle-NN.md`, com a **saída real** colada.

### 6.1 A mordida de cada item

Os cinco itens passando não bastam. **Cada um precisa reprovar com um defeito plantado:**

| Item | Defeito que deve reprová-lo |
|---|---|
| 1 | `Connection ""` no bloco do túnel do nginx |
| 2 | `replicaCount.orchestrator: 2` — o `agent not connected` intermitente do `#37` |
| 4 | `idle_timeout` curto na borda |
| 5 | — |

O item 2 com duas réplicas é o teste que transforma o `navyr-deploy#37` de argumento em medição. O item 5 não tem defeito plantado: a falha dele é a ausência de reconexão, que se observa desligando o orchestrator — já é o próprio teste.

## 7. Pipeline de entrega — o que se aplica

| Etapa | Status | Observação |
| --- | --- | --- |
| 1. Versionamento | ✅ | commit semântico por ciclo |
| 2. Testes unitários | ✅ | suíte Go do `navyr-agent` e do `navyr-orchestrator` |
| 3. Build de imagem | 🔁 substituído | o harness usa as imagens já publicadas no GHCR; esta spec não muda código de aplicação |
| 4. Scan de imagem | ⛔ N/A | sem imagem nova |
| 5. Push para registry | ⛔ N/A | sem imagem nova |
| 6. Teste funcional | ✅ | **é o produto desta spec** — os cinco itens contra a plataforma real |
| 7. Teste E2E | 🔁 substituído | `tunel-do-agente.sh` cobre o fluxo ponta a ponta do túnel; o `tests/e2e/` existente segue como está |

O CI da org executa o job dos itens 1–2. Os itens 3–5 passam de dez minutos com dois clusters kind e ficam em `workflow_dispatch`, pela razão da § 5.

## 8. Decisões em aberto (nenhuma bloqueia o início)

1. **Dois clusters kind na mesma máquina cabem no runner do CI?** Assumo que sim para os itens 1–2, e que os itens 3–5 ficam fora do gate. — Se não couber, o job do gate vira `workflow_dispatch` também, e a cobertura do túnel deixa de ser bloqueante até haver runner maior.
2. **O agente reconecta com backoff de quanto?** Assumo o que o `DialContext` já faz hoje, sem tocar nele. — Se o item 5 reprovar por tempo de espera e não por defeito, o critério vira "reconecta em até N segundos", com N medido e escrito.
3. **O item 4 mede contra o `idle_timeout` de qual borda?** Assumo o nginx do compose, que é o que o harness sobe. — O ALB do ECS tem 60s e é `navyr-deploy#17`; se o número divergir, o teste ganha o valor como parâmetro em vez de constante.
4. **`ConnectedIDs` é estável o bastante para asserção direta?** Assumo que sim, esperando por condição e não por `sleep`. — Se oscilar, a asserção passa a ser sobre a rota de sessão, que é contrato público.
