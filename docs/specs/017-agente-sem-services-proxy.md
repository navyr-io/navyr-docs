# SPEC-017 — O agente não tem `services/proxy`

**Card:** `navyr-io/navyr-orchestrator#18` · **Severidade:** Alto
**Descoberto:** varredura de interface ponta a ponta, Fase 3 (28/08/2026)

## O defeito

Todo o módulo de observabilidade reporta "indisponível" num cluster onde
Prometheus está instalado e funcionando, e reporta como se a dependência não
estivesse lá.

### Medido

Prometheus responde no endereço exato que o produto procura:

```
$ kubectl run sonda --image=busybox -- wget -O- \
    http://prometheus-operated.monitoring.svc.cluster.local:9090/api/v1/query?query=up
{"status":"success","data":{"resultType":"vector","result":[…]}}
```

O produto responde:

```
GET /api/v1/clusters/{id}/observability/prometheus/query_range?query=up
{"available": false, "data": null}
```

### Causa

O produto alcança Prometheus e Loki pelo **proxy de Service da API do
Kubernetes** — `internal/service/kubernetes.go:101`:

```go
req := kc.clientset.CoreV1().RESTClient().Get().
    AbsPath("/api/v1/namespaces", namespace, "services", "http:"+serviceName+":"+portName, "proxy", target)
```

Isso exige o subrecurso `services/proxy`. O ClusterRole que o manifesto do agente
entrega concede `services` — e no Kubernetes **o subrecurso é um recurso
distinto, não coberto pelo pai**.

Confirmado personificando a ServiceAccount do agente:

```
$ kubectl get --raw ".../services/http:prometheus-operated:9090/proxy/api/v1/query?query=up" \
    --as=system:serviceaccount:navyr-system:navyr-agent
Error from server (Forbidden)
```

São 13 pontos de chamada afetados — todo o módulo.

## Decisões

**D1 — Conceder `services/proxy`, e só ele.**
Verbo `get` apenas. O produto só lê pelo proxy; não há caminho que escreva.
Conceder `create` em `services/proxy` permitiria POST arbitrário a qualquer
Service do cluster, o que é uma capacidade bem maior do que a necessária.

**D2 — Não mexer em `pods/proxy`.**
O produto não usa proxy de pod. Conceder por simetria seria ampliar superfície
sem caso de uso.

**D3 — O agente já instalado não ganha a permissão sozinho.**
O ClusterRole vive no cluster do cliente e só muda quando o manifesto é
reaplicado. A correção precisa ser dita na interface, não só no código: um
cluster cujo agente foi instalado antes desta versão continuará reportando
observabilidade indisponível até reaplicar o manifesto.

## Regras

- **R1** — O ClusterRole do agente concede `services/proxy` com verbo `get`.
- **R2** — Nenhum outro subrecurso de proxy é concedido.
- **R3** — Existe teste que falha se `services/proxy` sair do manifesto. O teste
  ignora linhas de comentário, para não passar por causa de uma menção em
  comentário.

## Critérios de aceitação

1. `kubectl get --raw` pelo proxy, personificando a ServiceAccount do agente,
   deixa de responder `Forbidden`.
2. `GET /api/v1/clusters/{id}/observability/prometheus/query_range?query=up`
   responde `available: true` com dado.
3. Prova negativa: removendo `services/proxy` do manifesto, o teste de R3 falha.
