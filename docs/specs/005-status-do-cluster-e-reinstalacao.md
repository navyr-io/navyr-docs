# SPEC-005 — Duas noções de status com o mesmo nome, e a reinstalação bloqueada

**Estado:** aprovada — decisões D1 e D2 tomadas em 27/08
**Data:** 27/08/2026
**Card:** navyr-io/navyr-orchestrator#13
**Bloqueia:** R5 e R6 da [SPEC-004](004-probes-do-agente.md)

## Problema

Pedir reinstalação do agente responde 409 justamente nos clusters que precisam
reinstalar.

```
POST /api/v1/clusters/{id}/registration-token
409 — {"code":"conflict","message":"cluster is already connected — no reinstall needed"}
```

`internal/handler/registration_token_handler.go:140`:
```go
if cluster.Status == "ready" {
    http.Error(w, "cluster is already connected — no reinstall needed", http.StatusConflict)
```

## O sistema tem duas noções de status chamadas `status`

**A derivada**, calculada na hora a partir do heartbeat. `cluster_service.go:88-94`:

```go
if c.Status != models.ClusterStatusRevoked {
    if c.LastAgentSeenAt == nil || now.Sub(*c.LastAgentSeenAt) > 2*time.Minute {
        c.Status = models.ClusterStatusUnreachable
    } else {
        c.Status = models.ClusterStatusReady
    }
}
```

O comentário logo acima explica a razão de existir, e está certo:

> `"pending"` right after registration; that lies to the UI.
> `"revoked"` is admin-driven and not heartbeat-related, so we preserve it.

**A persistida**, coluna `orchestrator_clusters.status`. Recebe `revoked` e
`pending` por ação administrativa (`cluster_service.go:170` e `:180`) e `ready`
por `UpdateValidation`. **Nada a devolve para `unreachable`** — varredura no
repositório confirma que a string só aparece no enum, num contador local e num
nome de evento.

**A derivação só é aplicada em `List()`.** `Get()` e `Authorize()` devolvem o valor
cru do banco. A guarda usa `Authorize()`.

## Medido

Agente escalado para 0 réplicas, mesmo cluster, mesmo instante:

| Fonte | Valor |
|---|---|
| `GET /api/v1/clusters` (derivado) | `unreachable` |
| `psql orchestrator_clusters.status` (persistido) | `ready` |
| `POST …/registration-token` (lê o persistido) | 409 "already connected" |

A interface diz que o cluster está inalcançável e a API recusa a reinstalação
alegando que está conectado.

## Consequência

Depois da primeira conexão bem-sucedida, a coluna vai para `ready` e permanece.
A guarda passa a recusar **toda** reinstalação daquele cluster, para sempre.

O comentário do próprio handler, na linha 112, declara a intenção contrária:

```go
// Issues a fresh token for an existing unreachable/pending cluster.
```

O caminho que ele descreve é inalcançável, porque a coluna não chega em
`unreachable`.

## O que já está certo e não deve ser refeito

A regra de derivação — janela de 2 minutos sobre `last_agent_seen_at`, preservando
`revoked` — está implementada e documentada. Esta spec **não** propõe outra regra;
propõe aplicar a que existe onde ela falta.

## Regras

**R1 — A derivação vira função única, usada por `List`, `Get` e `Authorize`.**
Hoje ela mora embutida no laço do `List`. Extraída, deixa de existir caminho que
devolve o status cru por acidente.

**R2 — `revoked` continua tendo precedência sobre o heartbeat.**
É estado administrativo. Um cluster revogado cujo agente ainda responde não é
`ready` — é revogado com um agente que ainda não foi removido. Já é o comportamento
atual e vira regra explícita para não se perder na extração.

**R3 — A guarda de reinstalação passa a usar o status derivado.**
Consequência direta de R1: cluster com heartbeat velho volta a poder reinstalar,
que é o que o comentário do handler sempre prometeu.

**R4 — Reinstalar cluster saudável exige confirmação explícita.** *(D1)*
Emissão de token para cluster derivado como `ready` só acontece com um parâmetro
deliberado. Sem ele, a recusa permanece — mas passa a ser sobre o status derivado,
não sobre a coluna morta.

A recusa muda de mensagem: dizer "no reinstall needed" era afirmar algo que a
plataforma não sabe. Ela sabe que o agente responde; não sabe se o manifesto está
correto — foi exatamente essa a suposição que escondeu o navyr-orchestrator#12 por
1429 reinícios.

**R5 — Toda emissão para cluster saudável é auditada com o motivo.**
O ganho de exigir confirmação some se o registro não disser quem pediu e por quê.

**R6 — A coluna persistida passa a guardar só ciclo de vida.** *(D2)*
`pending` e `revoked` — os estados que vêm de ação administrativa e precisam
sobreviver a reinicialização. `ready` e `unreachable` deixam de ser gravados; são
sempre derivados. Migração normaliza as linhas existentes e aperta o CHECK, para
que a categoria de defeito "li a coluna errada" deixe de ser possível.

## Critérios de aceitação

1. Com o agente parado por mais de 2 minutos, `POST …/registration-token` emite
   token — hoje responde 409.
2. `GET /api/v1/clusters` e `GET /api/v1/clusters/{id}` concordam sobre o status do
   mesmo cluster no mesmo instante. Hoje divergem.
3. Cluster `revoked` com heartbeat fresco continua reportado como `revoked`.
4. Cluster `revoked` continua recusando emissão de token.
5. Existe teste que reprova se algum caminho de leitura devolver o status cru.
6. Emissão para cluster derivado `ready` sem confirmação continua recusada, e com
   confirmação emite e registra na auditoria.
7. Depois da migração, nenhuma linha em `orchestrator_clusters` tem `ready` ou
   `unreachable`, e o CHECK recusa esses valores.

## Decisões tomadas

**D1 — Sim, com confirmação explícita.** *(27/08)* Vira R4 e R5.

A alternativa de liberar só quando a plataforma detectasse motivo foi recusada
porque acoplaria esta correção à R5 da SPEC-004 e deixaria sem saída os motivos que
a plataforma não sabe detectar — RBAC alterado à mão, Secret rotacionado, imagem
corrompida no nó. Quem opera tem contexto que a plataforma não tem.

**D2 — A coluna passa a guardar só ciclo de vida.** *(27/08)* Vira R6.

Manter a coluna e passar a atualizá-la foi recusado: seriam duas fontes que
precisam concordar para sempre, que é exatamente a estrutura que produziu este
defeito.

## Fora de escopo

- A janela de 2 minutos (R2 preserva a atual).
- O R5/R6 da SPEC-004, que consome esta correção.
