# SPEC-037 — Cobertura do túnel do agente

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

## 3. Decisão

Um teste em `tests/k8s/tunel-do-agente.sh`, rodando contra a plataforma que o harness sobe, com **dois clusters kind**: um hospeda a plataforma, outro recebe o agente. Dois e não um, porque o caso real é o agente estar em cluster que não é o da plataforma — e é isso que o caminho ECS vende.

### 3.1 Os cinco itens, em ordem de custo

1. **o túnel conecta** e a plataforma reporta o cluster como conectado;
2. **uma ação de leitura atravessa** o túnel e devolve dado real do cluster do agente;
3. **`exec` interativo** troca bytes nos dois sentidos;
4. **stream de log** entrega linhas por mais tempo que o `idle_timeout` da borda — é o item que pegaria a regressão dos 60s;
5. **derrubar e resubir o orchestrator** faz o agente reconectar sozinho.

O item 5 é o que dá base à decisão de `desired_count = 1` no ECS: aquela escolha se apoia em o agente reconectar, e isso nunca foi verificado.

### 3.2 O que o teste NÃO deve fazer

Não deve usar um dublê de agente. O agente real é o que exercita a reconexão, o formato de quadro e o comportamento sob intermediário — que é o que produziu os três achados da seção 1.2.

## 4. Riscos

**Tempo.** Dois clusters kind mais a plataforma passa de dez minutos. Mitigação: os itens 1 e 2 num job, os 3 a 5 num job separado, disparável sob demanda.

**Flakiness.** É teste de rede de vida longa; o `#35` mostra o custo de um gate instável. Mitigação: esperar por condição observável — `IsConnected` — em vez de `sleep`, e falhar com o estado do cluster impresso.

## 5. Verificação

Os cinco itens passando, e **cada um reprovando com um defeito plantado**:

| Item | Defeito que deve reprová-lo |
|---|---|
| 1 | `Connection ""` no bloco do túnel do nginx |
| 2 | `replicaCount.orchestrator: 2` — o `agent not connected` intermitente do `#37` |
| 4 | `idle_timeout` curto na borda |
| 5 | — |

O item 2 com duas réplicas é o teste que transforma o `#37` de argumento em medição.
