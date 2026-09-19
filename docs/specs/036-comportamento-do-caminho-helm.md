# SPEC-036 — Comportamento do caminho Helm, não só a estrutura dele

**Estado:** aceita — em execução em 19/09/2026
**Data:** 19/09/2026
**Card:** navyr-io/navyr-deploy#40
**ADR:** 0006 (empacotamentos que divergem)
**Relacionada:** SPEC-034 (contrato), SPEC-035 (entrega por serviço), navyr-deploy#41 (túnel sem cobertura), navyr-deploy#39 (rota do túnel)

## 1. Problema

Medido em 18/09, o alvo de cada suíte do `navyr-deploy`:

| Suíte | Alvo |
|---|---|
| `tests/e2e/test_e2e_full_stack.sh` | `localhost:8080`, `:8081`, `:8082` |
| `tests/smoke/smoke_local_stack.sh` | `localhost:80xx` |
| `tests/chaos/run_chaos_suite.sh` | `docker compose` |
| `tests/functional/*.sh` | `localhost:80xx` |

Ou seja:

| Caminho | Estrutural | Comportamental |
|---|---|---|
| Compose | ✅ contrato | ✅ e2e, smoke, chaos, functional |
| Helm | ✅ contrato | **nenhuma** |
| ECS | ✅ contrato | **nenhuma**, e nunca aplicado |

**O efeito é pior do que "um caminho sem teste": o caminho mais testado é o menos parecido com produção.** Cliente nenhum instala por compose.

### 1.1 Por que o contrato não fecha isto

O contrato descreve quais serviços existem, portas, imagens, caminho de saúde, `depends_on`, quais variáveis cada serviço lê e os defaults de imagem. Ele **não descreve, e não tem como**:

- postura de segurança dos datastores — o módulo ECS cifra o Redis com AUTH token, o chart o deixa em texto claro dentro do cluster, e nenhuma catraca compara;
- a rota do túnel do agente, que difere em cada empacotamento (`#39`);
- multiplicidade de datastore — dois Redis no chart, um no compose e no ECS;
- HPA e PDB, que o chart tem e o ECS não;
- NetworkPolicy contra security group — 9 políticas renderizadas contra 4 grupos.

A prova de que o vão é real veio da SPEC-035: o chart rodava com o orchestrator **sem verificar a assinatura do contexto interno**, e nenhuma verificação estrutural via isso. Foi preciso subir o chart e mandar uma requisição forjada.

## 2. Decisão

### 2.1 As suítes deixam de saber contra o que rodam

`tests/lib/alvo.sh` define quatro URLs de serviço, com os defaults do compose:

```bash
NAVYR_GATEWAY_URL="${NAVYR_GATEWAY_URL:-http://localhost:8080}"
NAVYR_AUTH_URL="${NAVYR_AUTH_URL:-http://localhost:8081}"
NAVYR_BILLING_URL="${NAVYR_BILLING_URL:-http://localhost:8082}"
NAVYR_ORCHESTRATOR_URL="${NAVYR_ORCHESTRATOR_URL:-http://localhost:8083}"
```

São quatro e não uma base única de propósito: no compose cada serviço escuta porta própria, no Kubernetes o roteamento é por caminho num ingress. Base única esconderia a diferença em vez de a acomodar.

As 193 ocorrências de `http://localhost:80xx` nas 7 suítes foram substituídas. **Verificado que é no-op para o compose**: expandindo as variáveis de volta, os 7 arquivos são idênticos byte a byte aos de antes.

### 2.2 Subir o alvo deixa de ser responsabilidade da suíte

`NAVYR_ALVO_EXTERNO=1` torna `compose_up` e `compose_down` no-ops. Não é um modo de teste mais fraco — as verificações são as mesmas, só o dono de levantar o alvo muda.

`run_sql` ganha a mesma indireção por `NAVYR_PSQL_CMD`. As suítes consultam o banco para verificar efeito colateral — outbox de e-mail, eventos de uso, auditoria —, e isso é insubstituível por resposta HTTP: "a API devolveu 200" não prova que o evento foi gravado. O **como** alcançar o banco, porém, é do empacotamento: `docker compose exec` no compose, `kubectl exec` no chart.

### 2.3 Um harness, e um job de CI

`tests/k8s/subir-em-kind.sh` cria o cluster, instala o chart, faz port-forward dos quatro serviços **nas mesmas portas do compose**, e roda as suítes pedidas. A porta igual é deliberada: a suíte não sabe — nem precisa saber — contra qual empacotamento está rodando.

Os segredos são gerados pelo harness e passados **ao chart e às suítes**. Precisam bater: o teste do gateway assina token com `NAVYR_JWT_SECRET`, e valor divergente recusaria a requisição por motivo que não tem a ver com o que se testa.

Job `helm-em-kind` no `ci.yml`, rodando `smoke`, `gateway` e `billing`. Roda em `kind` dentro do runner, então não custa nuvem. O caminho ECS fica fora de propósito: exige conta AWS com recurso ligado, que é decisão de custo separada (`#17`).

## 3. Uma correção de desenho, achada testando o próprio teste

A primeira versão do harness alcançava os serviços por `kubectl port-forward svc/...`. **Isso não atravessa o Service.**

Descoberto em 19/09 plantando `targetPort: 9999` no Service do gateway para provar que o job vale algo: o cluster ficou com o defeito, confirmado por

```
kubectl -n navyr get svc navyr-gateway -o jsonpath='{.spec.ports[*]}'
{"port":8080,"protocol":"TCP","targetPort":9999}
```

e a suíte smoke **passou igual**, porque o port-forward fala com o pod.

O harness verificava, então, os contêineres — não a camada de rede do chart. E Service, DNS interno e NetworkPolicy são justamente o que **só existe no caminho Kubernetes**, e o que o contrato não descreve. Um harness que não os exercita deixa de fora a categoria de defeito mais específica do empacotamento que ele existe para testar.

**Correção:** uma sonda que roda como pod **dentro** do cluster, curlando `http://navyr-<serviço>:<porta>/health`. Ela atravessa resolução de DNS do Service, mapeamento `port`/`targetPort` e as NetworkPolicies.

A sonda leva o rótulo `app=navyr-gateway` porque a `navyr-backend-allow` só admite entrada de gateway e collector — sonda sem rótulo seria barrada, e o teste falharia por motivo errado.

Fica a divisão: **port-forward verifica os contêineres, sonda verifica a rede do chart.** As duas, porque nenhuma cobre a outra.

## 4. O que esta spec NÃO cobre

**O Ingress.** As suítes falam com cada serviço na porta dele; o roteamento por caminho da borda — `/api` → gateway, `/auth` → auth, `/` → frontend — é verificação própria. É onde o caminho ECS mais se parece com o Helm e menos com o compose.

**O túnel do agente.** `#41`, e é a função central do produto. Depende deste card para ter onde rodar.

**A suíte de caos.** Ela manipula contêineres por nome de projeto do compose; adaptá-la a pods é trabalho próprio.

## 5. Verificação

1. Expandindo as variáveis, as 7 suítes são idênticas às de antes — **feito**;
2. o harness sobe o chart em `kind` e a suíte smoke passa contra ele;
3. o job de CI sai verde;
4. **um defeito introduzido só no caminho Helm reprova o job novo e não o do compose** — é isto que prova que o job vale algo.
