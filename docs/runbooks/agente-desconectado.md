# Agente desconectado (cluster registrado mas não responde)

O orchestrator nunca conecta ao cluster do cliente: o agente abre uma conexão
de saída por WebSocket e ela fica viva. `tunnel.Registry` mapeia
`clusterID → *Conn`. Ver [ADR 0001](../adr/0001-agent-tunnel.md).

Consequência direta: **operação no cluster depende da conexão estar viva.** Se
ela morre, o cluster aparece registrado e nenhuma operação funciona.

## 1. O orchestrator considera o cluster conectado?

```bash
NS=navyr
kubectl -n "$NS" logs -l app=navyr-orchestrator --tail=100 | grep -iE 'tunnel|cluster_id|register'
```

Procure o `cluster_id` do cluster em questão. `Unregister` recente indica queda
detectada; **ausência de qualquer registro** indica que o agente nunca chegou.

## 2. O agente está de pé no cluster do cliente

```bash
kubectl --context <cluster-do-cliente> -n navyr-agent get pods
kubectl --context <cluster-do-cliente> -n navyr-agent logs -l app=navyr-agent --tail=50
```

Causas mais comuns, em ordem de frequência:

- **Token de registro expirado ou já usado.** Gere outro e reinstale o
  manifesto do agente.
- **Egress bloqueado.** O agente precisa alcançar o orchestrator na porta do
  WebSocket. Proxy corporativo que não fala WebSocket derruba o upgrade.
- **URL com esquema errado.** `ws://` onde deveria ser `wss://` já vazou para
  chamada HTTP — foi defeito real e está corrigido, mas configuração manual
  pode reintroduzir.

## 3. Se o agente diz estar conectado e o orchestrator não concorda

Este é o caso da **morte silenciosa**: a conexão TCP não recebeu FIN nem RST —
por exemplo um NAT que expirou a tradução — e as duas pontas acham que ainda
existe.

Detectar isso é justamente o que `SetReadDeadline` e o ping periódico fazem:

| Parâmetro | Valor |
|---|---|
| Prazo de leitura | 60s |
| Intervalo de ping | 20s |

Com esses valores, uma conexão morta é detectada em no máximo 60 segundos. Se
o orchestrator considera conectado por **mais de um minuto** enquanto o agente
não recebe nada, isso é defeito e merece issue — não é o comportamento
esperado.

**Mitigação imediata:** reinicie o agente. Ele reabre a conexão, e o
`Register` sobrescreve a entrada antiga no registry.

```bash
kubectl --context <cluster-do-cliente> -n navyr-agent rollout restart deploy/navyr-agent
```

## 4. Se vários clusters caem ao mesmo tempo

Aí a suspeita inverte: não são os agentes, é o orchestrator. Reinício do pod
derruba todas as conexões que ele mantinha — elas são **stateful e vivem no
processo**. Os agentes reconectam sozinhos, mas há uma janela de
indisponibilidade.

```bash
kubectl -n "$NS" get pods -l app=navyr-orchestrator   # RESTARTS recentes?
```

É a consequência registrada no [ADR 0001](../adr/0001-agent-tunnel.md): o
túnel não sobrevive à troca de pod. Com o HPA do orchestrator ativo, a
reconexão em massa passa a acontecer também em escala automática — e é o ponto
a observar quando isso for ligado.

## O que não fazer

**Não presuma que "registrado" significa "alcançável".** O registry guarda a
conexão, e a conexão pode estar morta sem que ninguém tenha avisado. A
verificação honesta é uma operação real contra o cluster, não a presença no
registry.
