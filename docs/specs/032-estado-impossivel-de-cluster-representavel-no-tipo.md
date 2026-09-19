# SPEC-032 — O estado impossível de cluster ainda é representável no tipo

**Estado:** proposta
**Data:** 16/09/2026
**Card:** navyr-io/navyr-deploy#19
**Relacionada:** SPEC-005 R6 (migration `000030`, que estreitou o schema), SPEC-030 (gate do gosec — card irmão)

## 1. Problema

Com o CI de volta desde 01/09, três passos reprovam além do `gosec`. Dois são
triviais; o terceiro esconde um defeito de desenho.

### 1.1 Testes de integração do orchestrator — o achado real

```
--- FAIL: TestAtualizacaoDeStatusPersiste (0.01s)
integration_test.go:177: UpdateStatus: ERROR: new row for relation
  "orchestrator_clusters" violates check constraint
  "chk_orchestrator_clusters_status" (SQLSTATE 23514)
FAIL	github.com/navyr-io/navyr-orchestrator/internal/repository	18.517s
```

A migration `000030_status_do_cluster_so_ciclo_de_vida` (SPEC-005 R6) estreitou
a coluna de `('pending','ready','unreachable','revoked')` para
`('pending','revoked')`. A justificativa, no próprio arquivo:

> A coluna status guardava duas coisas com o mesmo nome: ciclo de vida
> ('pending', 'revoked'), que vem de ação administrativa e precisa sobreviver a
> reinicialização, e liveness ('ready', 'unreachable'), que só o heartbeat sabe.
> A metade de liveness nunca foi mantida: nada no código devolvia a coluna para
> 'unreachable'. Depois da primeira conexão ela ficava em 'ready' para sempre, e
> a guarda de reinstalação — que lia essa coluna — passava a recusar todo pedido
> daquele cluster.

A decisão foi certa. O que ficou pela metade é que **só o schema foi
estreitado**:

| Camada | Estados que admite |
|---|---|
| Coluna `status` (após `000030`) | `pending`, `revoked` |
| Tipo `models.ClusterStatus` | `pending`, `ready`, `unreachable`, `revoked` |
| `PostgresClusterRepository.UpdateStatus` | **qualquer um** — grava `SET status = $3` sem validar |

Medido em `internal/repository/cluster_repository.go:155-165`: a função recebe
`status models.ClusterStatus` e o escreve direto no `UPDATE`. Nenhuma validação.

Em produção o código nunca passa `ready` — ele é **derivado** de
`last_agent_seen_at` por `StatusDerivado`, e os 14 pontos que leem
`ClusterStatusReady` leem o derivado (`cluster_service.go:118-120`,
`aiops_service.go:806`, `workers.go:176,321`, entre outros). Nenhum grava.

Então o defeito não morde hoje. Mas a assinatura **convida** a mordida: qualquer
chamador novo que passe `ClusterStatusReady` compila, passa no code review por
parecer natural, e falha só no banco em runtime — com `SQLSTATE 23514`, que não
diz ao leitor que o estado foi deliberadamente abolido. É exatamente o tipo de
armadilha que a SPEC-005 R6 quis remover, sobrevivendo na camada que ela não
tocou.

O teste `TestAtualizacaoDeStatusPersiste` é o primeiro chamador a cair nela. Ele
não é "teste velho a consertar" — é o detector funcionando.

### 1.2 `golangci-lint` no navyr-gateway

```
cmd/server/main.go:2594:6: func effectiveScopeFromTokenClaims is unused (unused)
cmd/server/rotas_de_auth.go:45:32: QF1001: could apply De Morgan's law (staticcheck)
```

Confirmado na `main`: `grep -rn "effectiveScopeFromTokenClaims" cmd/ internal/`
devolve **uma** ocorrência, a declaração. Nenhuma chamada.

A linha do De Morgan é `rotaDeAuth.casa`, que decide **quais rotas passam sem
autenticação**:

```go
if caminho != rota.publica && !(rota.prefixo && strings.HasPrefix(caminho, rota.publica)) {
```

Simplificação aqui muda fronteira de segurança. Não se toca sem teste cobrindo
antes e depois.

### 1.3 `golangci-lint` no navyr-agent — não está na `main`

```
cmd/executor/exec_transporte.go:11:10: SA1019:
  "k8s.io/apimachinery/pkg/util/httpstream/spdy" is deprecated:
  use k8s.io/streaming/pkg/httpstream/spdy directly. (staticcheck)
```

A `main` está em `k8s.io/apimachinery v0.31.3`, onde o pacote **não** é
depreciado. Quem deprecia é o bump do Dependabot. Não é defeito em produção — é
pré-requisito para aceitar a atualização do Kubernetes, e mexe em
`exec_transporte.go`, o caminho do `kubectl exec` por cima do tunnel.

## 2. Regras e critérios de aceitação

| # | Regra | Como se prova |
|---|---|---|
| 1 | Estado que o schema proíbe não é representável no caminho de escrita | Passar `ClusterStatusReady` a `UpdateStatus` não compila, **ou** é rejeitado antes do banco com erro que nomeia a SPEC-005 R6 |
| 2 | A derivação de liveness continua intacta | `go test ./internal/service/... -run StatusDerivado -race` verde; nenhum dos 14 pontos de leitura alterado |
| 3 | Nenhuma migration nova | `ls migrations/ \| tail -1` continua em `000030`; o schema já está certo |
| 4 | `effectiveScopeFromTokenClaims` sai, ou passa a ser usada, com o motivo registrado | `golangci-lint run ./...` sem `unused` em `navyr-gateway` |
| 5 | A fronteira de rota pública não muda de comportamento | Teste de `rotaDeAuth.casa` cobrindo a tabela de rotas **antes** da mudança, verde depois, com a mesma tabela |
| 6 | O bump do Kubernetes no agente passa a ser aceitável | `golangci-lint run ./...` limpo no branch do Dependabot, e o túnel provado por teste de `exec` |
| 7 | Os 3 repos fecham CI verde em `main` | `gh run list --branch main` mostra `success` no workflow `CI` em gateway, agent e orchestrator |

O critério 7 depende da SPEC-030: o job `seguranca` reprova em paralelo. Esta
spec pode ser **trabalhada** antes, mas só **converge** depois.

## 3. Fora de escopo

- **SPEC-030 / `#18`** — o gate do gosec.
- **Reescrever `StatusDerivado`** — está correto e é a decisão da SPEC-005 R6.
- **Remover `ready`/`unreachable` de `models.ClusterStatus`** como constantes:
  elas continuam válidas para o status **derivado**, que é o que os 14 pontos
  de leitura consomem. O problema é só o caminho de escrita.
- **Migração do `spdy` para outro pacote no agente** além do mínimo que o
  `SA1019` exige.

## 4. Plano de execução

| Ciclo | Entrega | Critério de pronto |
|---|---|---|
| 1 | Teste que prova a armadilha antes de fechá-la: chamar `UpdateStatus` com `ready` e exigir rejeição **antes** do banco | `go test ./internal/repository/ -run Ciclo -race` falha pelo motivo certo (TDD: vermelho primeiro), saída colada |
| 2 | Tipo separado para escrita — `ClusterLifecycle` com `pending`/`revoked` — e `UpdateStatus` passa a recebê-lo | Critério 1. Compilação prova: nenhum chamador consegue passar `ready` |
| 3 | `TestAtualizacaoDeStatusPersiste` reescrito para o comportamento real (ciclo de vida persiste; liveness deriva) | `go test ./internal/repository/ -race` verde, saída colada |
| 4 | `navyr-gateway`: teste de tabela de `rotaDeAuth.casa`, depois a simplificação do De Morgan; e decisão sobre `effectiveScopeFromTokenClaims` | Critérios 4 e 5 |
| 5 | `navyr-agent`: troca do import `spdy` e teste do caminho de `exec` pelo tunnel | Critério 6 |
| 6 | Convergência | Critério 7, depois da SPEC-030 |

Cada ciclo termina em commit semântico com a suíte verde.

## 5. Pipeline de entrega — o que se aplica

| Etapa | Status | Observação |
|---|---|---|
| 1. Versionamento | ✅ | commit semântico por ciclo |
| 2. Testes unitários | ✅ | é o coração desta spec; TDD no ciclo 1 |
| 3. Build de imagem | ✅ | orchestrator, gateway e agent produzem imagem |
| 4. Scan de imagem | ✅ | trivy pelo `publish-image.yml`; independe do gate do gosec |
| 5. Push para registry | ✅ | no merge para `main` |
| 6. Teste funcional | ✅ | registrar cluster, revogar, e conferir que liveness deriva do heartbeat |
| 7. Teste E2E | ✅ | a tela de cluster mostra `unreachable` quando o agente cai — é o comportamento que a SPEC-005 R6 consertou e que não pode regredir |

**Correção ao texto da skill:** ela afirma que o CI está bloqueado pelo plano
Free desde 19/08/2026. Deixou de ser verdade em 01/09.

## 6. Decisões em aberto (nenhuma bloqueia o início)

1. **Forma de tornar o estado irrepresentável.** Assumo **tipo novo**
   (`ClusterLifecycle`) em vez de validação em runtime: o compilador é mais
   barato que um erro de banco, e a SPEC-005 R6 já provou que comentário não
   segura. — Custa uma passada de movimento puro nos chamadores de
   `UpdateStatus`; se ficar invasivo demais, cai para validação na função com
   erro nomeado.

2. **`effectiveScopeFromTokenClaims`.** Assumo **remover**: `git log` mostra que
   nasceu com o handoff de escopo e o caminho vivo hoje é outro. — Custa perder
   código que alguém pretendia usar; mitigo citando o SHA na nota de ciclo, de
   onde volta em um `git show`.

3. **O De Morgan em `rotaDeAuth.casa`.** Assumo aplicar a simplificação **só se**
   o teste de tabela do ciclo 4 passar idêntico antes e depois; caso contrário
   anoto `//nolint:staticcheck` com o motivo. — Custa um lint anotado; é mais
   barato que mexer em fronteira de autenticação por estética.

4. **`spdy` no agente.** Assumo trocar o import para
   `k8s.io/streaming/pkg/httpstream/spdy` mantendo a assinatura de
   `exec_transporte.go`. — Custa: se a API do pacote novo divergir, o ciclo 5
   cresce e vira card próprio.
