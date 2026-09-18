# SPEC-035 — Entrega por serviço, e o contexto interno que não era verificado

**Estado:** aceita — executada em 18/09/2026
**Data:** 18/09/2026
**Card:** navyr-io/navyr-deploy#38
**ADR:** 0006 (empacotamentos que divergem), 0008 (autorização fail-closed)
**Relacionada:** navyr-helm#6 (a catraca), SPEC-034 (contrato de empacotamento), navyr-deploy#40 (teste comportamental)

## 1. Problema

A catraca do chart conferia **"a variável é entregue em algum lugar"**, não "chega ao serviço que a lê":

```python
entregues |= {e["name"] for e in c.get("env", [])}   # união de TODOS os contêineres
...
if var not in entregues:                              # conferência global
```

Isso aprova o caso em que a variável chega ao serviço **errado**. E aprovava um defeito de segurança real.

### 1.1 O defeito que passava verde

O chart **não entrega `INTERNAL_CONTEXT_SIGNING_SECRET` ao orchestrator**. O segredo existe em `navyr-secrets`; o deployment do orchestrator referencia outros três (`CLUSTER_CREDENTIAL_ENCRYPTION_KEY`, `DATABASE_URL`, `WS_EXEC_TICKET_SECRET`). Como gateway, auth e community o recebem, a conferência global passava.

Cadeia medida no código do orchestrator:

```go
// main.go:100
internalCtxSecret := strings.TrimSpace(os.Getenv("INTERNAL_CONTEXT_SIGNING_SECRET"))
// main.go:194
... internalContextAuthMiddleware(bodyLimitMiddleware(mux), internalCtxSecret) ...

// middleware.go:128
if secret == "" {
    return next          // passa direto, sem verificar assinatura
}
```

O chart tem `appEnv: staging` por default, então `validateProductionSecurity` (`main.go:98`, que faz `log.Fatal`) não dispara.

**Dois modos de falha, conforme o `appEnv`:**

| `appEnv` | Comportamento |
|---|---|
| `staging` (default do chart) | o middleware é **desviado**: `X-User-Role` e `X-Cluster-ID` são acreditados sem assinatura |
| `production` | o orchestrator **não sobe** — `log.Fatal` na validação |

O segundo é a mesma classe que o `navyr-helm#6` achou para o `navyr-auth`, e que o `values.yaml:136` do chart documenta: *"instalar com global.appEnv=production punha o auth em CrashLoop"*. O padrão sobreviveu no orchestrator.

### 1.2 O que segurava

NetworkPolicy: `navyr-default-deny` com `podSelector: {}`, e `navyr-backend-allow` permitindo entrada no orchestrator **só** de `navyr-gateway` e `navyr-collector`.

A rede segura, e não deveria ser a única coisa a segurar:

- o `navyr-collector` alcança o orchestrator e não precisa de privilégio administrativo para a função dele — comprometê-lo escalava a `org_admin` em qualquer cluster;
- NetworkPolicy só vale com CNI que a aplique. Sem enforcement, a política é decorativa e a assinatura era a única barreira.

A assinatura existe justamente para que posição de rede não baste (ADR 0008).

### 1.3 Exposição, medida

Nenhuma. Verificado em 18/09:

| Sinal | Resultado |
|---|---|
| Releases `navyr-platform` em clusters alcançáveis | nenhum |
| `kind-navyr-demo` | só o agente |
| `v0.1.0` | publicada 16–17/09 |
| `navyr-install` (entrada pública) | criado 17/09 |
| Forks · stars · watchers · **visitas** | 0 · 0 · 0 · **0** |
| Clones do `navyr-install` | 25/15 únicos, **todos no dia da criação**, com 0 visitas — padrão de bot |

Sem instalação de terceiro, não houve incidente. O prazo é a próxima instalação, não uma já feita.

## 2. Decisão

### 2.1 A catraca passa a conferir por serviço

`test_empacotamentos_cobrem_o_codigo.py` devolve **mapa serviço → variáveis entregues** em vez de conjunto plano, e exige que **cada serviço que LÊ a variável** a receba. O contrato já diz quem lê o quê, em `variaveis`.

Serviço que existe no contrato e não tem workload no chart recebe mensagem própria — é divergência de topologia, e culpar a variável esconderia isso.

### 2.2 Exclusões passam a ser por par (variável, serviço), com motivo

A exclusão global não serve mais: `REDIS_URL` deve chegar ao gateway e legitimamente não ao collector. Cada par excluído carrega motivo, e o **contrapeso** continua — par excluído que ESTÁ sendo entregue reprova.

### 2.3 As cinco lacunas, decididas uma a uma

| Par | Decisão | Motivo |
|---|---|---|
| `INTERNAL_CONTEXT_SIGNING_SECRET` → orchestrator | **entregar** | é o defeito de segurança |
| `JWT_SECRET` → orchestrator | excluir | é fallback de `WS_EXEC_TICKET_SECRET` (`ws_ticket_store.go:40`), que o chart entrega. Acrescentar segredo não usado amplia exposição sem ganho |
| `REDIS_URL` → orchestrator | excluir | ligaria `runCollector` em processo (`main.go:168`) **além** do serviço `navyr-collector`, que já faz coleta. Dois laços escrevendo no mesmo Redis é decisão de arquitetura, não de empacotamento |
| `COLLECTOR_INTERVAL` → orchestrator | excluir | só tem efeito se o laço interno rodar |
| `REDIS_URL` → collector | excluir | o chart usa `REDIS_ADDR`, que o collector suporta e prefere apenas quando `REDIS_URL` está definida (navyr-collector `eb0692a`) |

## 3. Um segundo defeito, achado ao verificar o primeiro

Subindo o chart em `kind` para testar, o `helm upgrade` com `appEnv=production` e segredos novos saiu **"deployed"** — e os pods não rolaram:

| | Valor |
|---|---|
| ConfigMap no cluster | `APP_ENV=production` |
| Secret no cluster | 48 caracteres |
| **Pod rodando** | **`APP_ENV=staging`, segredo de 23** |

Os pod templates não tinham `checksum/config` nem `checksum/secret`. Sem elas, mudar ConfigMap ou Secret não reinicia nada.

**A consequência que decide: rotação de segredo não surtia efeito.** Quem rotaciona uma credencial vazada e vê o `helm upgrade` dar certo acredita que a rotação aconteceu, e o valor vazado segue em uso até algo reiniciar o pod por outro motivo.

Corrigido: as duas annotations nos 8 pod templates dos Deployments e no StatefulSet do Postgres — 17 no render. Com ressalva registrada no template do Postgres: reiniciar o pod faz ele ver o valor novo, mas `POSTGRES_PASSWORD` só vale na primeira inicialização do volume, então rotação de verdade exige `ALTER ROLE`.

## 4. O que esta spec NÃO resolve

**A duplicação entre o `runCollector` em processo e o serviço `navyr-collector`.** A exclusão de `REDIS_URL` acima é a escolha conservadora — não ligar um segundo coletor —, mas deixa código que nenhum empacotamento exercita. Card próprio.

**A verificação comportamental.** Ligar uma verificação que nunca rodou pode recusar requisições se a assinatura do gateway e a conferência do orchestrator divergirem em formato ou em `maxAge`. Esta spec exige teste em cluster real antes do merge, e é o primeiro pedaço do `navyr-deploy#40`.

## 5. Verificação — executada em 18/09

Tudo abaixo foi medido, não projetado.

**1 a 3, a catraca mordendo.** Três perturbações, três reprovações:

```
INTERNAL_CONTEXT_SIGNING_SECRET → navyr-gateway: não chega a este serviço
REDIS_URL → navyr-collector: decida — ou entrega, ou exclui com motivo
COLLECTOR_INTERVAL → navyr-orchestrator          (motivo curto)
```

Estado verde: **51 pares variável/serviço** conferidos, 2 variáveis e 4 pares fora por decisão registrada. Antes eram 34 conferências por variável.

**4, a assinatura em cluster real.** Chart instalado em `kind` (14 pods prontos, `helm install --wait` rc=0). Requisição assinada com o algoritmo do gateway, replicado do código dele (`main.go:1787`), contra o orchestrator:

| Requisição | Resultado |
|---|---|
| assinada corretamente | **HTTP 200** |
| assinatura inválida | **401** `invalid internal context signature` |
| sem assinatura, `X-User-Role: org_admin` forjado | **401** `missing internal context signature` |
| `/health` | 200 — segue aberto, como deve |

As duas implementações de `buildInternalContextPayload` são idênticas — mesmo payload `METHOD|path|org|cluster|user|request|ts`, mesmos nomes de cabeçalho, HMAC-SHA256 hex, `maxAge` 60s. **Ligar a verificação não quebra o caminho.**

**5, produção.** Após as annotations de checksum, `helm upgrade --set global.appEnv=production` rolou os pods e o orchestrator subiu: `APP_ENV=production`, segredo de 48 caracteres, `1/1 Running`. Assinatura reconferida nesse modo: 200 para assinada, 401 para sem assinatura.

Instalação removida ao fim; o cluster voltou ao estado anterior.

### O que a verificação achou de passagem

`replicaCount.orchestrator: 2` é o **default do chart**, sem aviso. O `navyr-deploy#37` descreve `tunnel.Registry` como mapa em memória que impede segunda réplica — e a instalação padrão já roda duas. Deixa de ser risco teórico e passa a ser a configuração entregue. Registrado no card.
