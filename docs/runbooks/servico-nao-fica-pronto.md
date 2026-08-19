# Serviço não fica pronto (pod em 0/1)

`Running` mas `0/1` significa que o processo subiu e a **readiness** está
reprovando. É diferente de `CrashLoopBackOff`, onde o processo morre.

Isso é comportamento correto, não defeito: o pod sai do Service e para de
receber tráfego que não conseguiria atender.

## 1. Qual probe está reprovando

```bash
NS=navyr
kubectl -n "$NS" describe pod <pod> | grep -A3 -E 'Readiness|Liveness'
kubectl -n "$NS" get events --sort-by=.lastTimestamp | tail -20
```

## 2. Se for o `/ready`, é dependência

`/health` responde 200 incondicionalmente e serve para liveness — uma queda de
banco **não deve** reiniciar o processo. `/ready` reflete a dependência.

| Serviço | O que `/ready` verifica |
|---|---|
| auth, billing, orchestrator, community | `Ping` no pool do Postgres, timeout 2s |
| gateway | Alcança auth, billing, community e orchestrator |
| frontend | `/healthz` do nginx, estático |

Então: **gateway em 0/1 quase nunca é problema do gateway.** É um dos quatro
abaixo dele. Olhe os outros primeiro.

```bash
kubectl -n "$NS" get pods    # quem mais está 0/1?
```

## 3. Se for banco

```bash
kubectl -n "$NS" logs -l app=navyr-auth --tail=20 | grep -i 'database\|migration\|connect'
kubectl -n "$NS" get pod -l app=navyr-postgres
```

**`i/o timeout` resolvendo `navyr-postgres` é DNS, não banco.** Causa provável:
NetworkPolicy sem egress de DNS. Confirme:

```bash
kubectl -n "$NS" get networkpolicy
kubectl -n "$NS" get networkpolicy navyr-backend-allow -o yaml | grep -A6 'port: 53'
```

Se não houver regra de porta 53, a policy está incompleta — foi um defeito
real, corrigido em 19/08. Veja [achados-abertos.md](../achados-abertos.md).

> Um detalhe que custa tempo: **NetworkPolicy exige as duas pontas.** Não basta
> a origem ter egress liberado; o destino precisa aceitar ingress. Uma conexão
> que falha com egress aparentemente correto costuma ser ingress faltando do
> outro lado.

## 4. Se nada disso, teste de dentro

```bash
kubectl -n "$NS" run diag --rm -it --image=busybox:1.36 \
  --labels="app=navyr-auth" --restart=Never -- sh
# dentro:
nslookup navyr-postgres
nc -zv navyr-postgres 5432
```

O `--labels` importa: sem ele o pod não é selecionado pelas mesmas
NetworkPolicies e o teste não reproduz a condição real.

## O que não fazer

**Não desligue a NetworkPolicy para "destravar".** Se a policy é a causa, o
defeito está nela e volta no próximo ambiente. E `networkPolicy.enabled=false`
já deixou duas policies aplicadas por um `{{- end }}` fora de lugar — desligar
pode não desligar.

**Não aumente o `failureThreshold` da readiness** para o pod "ficar verde". Ele
ficaria verde recebendo tráfego que não consegue atender, que é exatamente o
que a probe existe para evitar.
