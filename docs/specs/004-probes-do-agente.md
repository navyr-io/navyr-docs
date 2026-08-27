# SPEC-004 — Os probes do agente no manifesto de instalação

**Estado:** implementada e verificada — 27/08
**Data:** 27/08/2026
**Card:** navyr-io/navyr-orchestrator#12 (Crítico)

## Problema

O manifesto que o produto entrega ao cliente define um liveness probe que aponta
para um caminho que o agente não serve. O kubelet recebe 404, falha três vezes e
mata o contêiner. O ciclo se repete para sempre.

Medido no cluster kind, agente instalado pelo caminho oficial:

```
restartCount: 1150
última saída: reason=Error exitCode=137 (SIGKILL)
```

`exitCode 137` é SIGKILL do kubelet — o processo não está saindo sozinho.

## O que o agente serve, e o que o manifesto pede

Medido **na imagem que já está no cluster**, não na compilada agora:

| Caminho | Resposta | Semântica implementada |
|---|---|---|
| `/health` | **404** | não existe |
| `/healthz` | 200 | processo vivo — sempre 200 |
| `/ready` | 200 | túnel conectado **e** heartbeat com menos de 60s |

`navyr-agent/cmd/executor/main.go:191-206` implementa os dois últimos. O
`/ready` já tem exatamente a semântica que um readiness probe precisa:

```go
if !wsConnected.Load() || last == 0 || time.Since(time.Unix(last,0)) > 60*time.Second {
    http.Error(w, "not ready", http.StatusServiceUnavailable)
```

`navyr-orchestrator/internal/agentmanifest/manifest.go:161-164` emite:

```yaml
livenessProbe:
  httpGet: { path: /health, port: 8090 }
```

E não emite readiness nenhum.

**A decisão de design estava certa e o manifesto nunca a alcançou.** Não é falta
de funcionalidade no agente — é o gerador do manifesto ignorando o que o agente
oferece.

## Consequência de a imagem em campo já estar correta

Como `/healthz` e `/ready` existem na imagem instalada, **corrigir apenas o
manifesto resolve, sem publicar imagem nova**. Quem já instalou reaplica o
manifesto e o agente para de morrer, usando a mesma imagem que já está no nó.

Isso importa porque o manifesto emite `imagePullPolicy: IfNotPresent` com a tag
`:latest` (`manifest.go:145` e `:22`). A combinação significa que **um agente
instalado nunca baixa imagem nova** — o nó já tem `:latest` em cache e não
consulta o registry outra vez. Os 1150 reinícios não atualizaram nada.

## Regras

**R1 — O liveness passa a apontar `/healthz`.**
Corrige o defeito. `healthz` é a convenção do ecossistema e o agente já a segue;
mudar o agente para `/health` alinharia o par afastando-se da convenção.

**R2 — O manifesto passa a emitir um readinessProbe apontando `/ready`.**
O endpoint existe e reflete o túnel. Sem readiness, o pod é dado como pronto assim
que o processo sobe, mesmo com o túnel caído.

Consequência verificada: o manifesto não cria `Service` — os recursos são
Namespace, ServiceAccount, ClusterRole, ClusterRoleBinding, Secret e Deployment.
Então `NotReady` não corta tráfego de ninguém; o efeito é só tornar o estado do
túnel visível em `kubectl get pods`. Risco operacional nulo, ganho de diagnóstico
real.

**R3 — O liveness não observa o túnel.**
`/healthz` responde 200 com o túnel caído, e deve continuar assim. Liveness que
depende de rede externa transforma partição de rede em laço de reinício — que é
o defeito que esta spec corrige, por outro caminho.

**R4 — A correção não exige imagem nova do agente.**
Verificado endpoint a endpoint na imagem em campo. Nenhuma mudança em
`navyr-agent` faz parte desta spec.

**R5 — O orquestrador detecta agente com probe obsoleto.**
Pelo túnel, lê o Deployment `navyr-agent` em `navyr-system` e compara o caminho do
liveness com o que o manifesto atual emite. Divergência é sinal de instalação
anterior à correção.

A leitura é oportunista: se o túnel estiver caído ou o RBAC não permitir, o
resultado é "desconhecido", nunca erro — um diagnóstico não pode derrubar a tela
que ele existe para informar.

**R6 — A interface mostra o aviso junto da ação que o resolve.**
Cluster com agente obsoleto exibe aviso na lista e no cockpit, com o comando de
reinstalação — que já existe. Aviso sem ação ao lado vira ruído que o operador
aprende a ignorar.

## Critérios de aceitação

1. O manifesto gerado contém `path: /healthz` no liveness e `path: /ready` no readiness.
2. Aplicado num cluster limpo, `RESTARTS` permanece em `0` por mais de 10 minutos.
3. Com o orquestrador derrubado, o pod fica `0/1 Running` — **não** reinicia. Prova
   que readiness reflete o túnel e liveness não.
4. Restabelecido o orquestrador, o pod volta a `1/1` sem reinício.
5. Reaplicar o manifesto corrigido num cluster que já tinha o agente antigo faz
   `RESTARTS` parar de subir, sem baixar imagem nova.
6. Existe teste em `agentmanifest` que reprova se o caminho do probe divergir dos
   caminhos que o agente registra.

## Decisões tomadas

**D1 — A base instalada é sinalizada na interface.** *(decidido em 27/08)*

Corrigir o manifesto em silêncio deixaria os clusters já instalados morrendo sem
que ninguém percebesse — o cockpit mostra `ready` porque o agente reconecta a cada
reinício, e a coluna `RESTARTS` só aparece para quem rodar `kubectl` no cluster do
cliente. O produto tem o túnel e permissão para olhar, então deve olhar.

Vira as regras R5 e R6 abaixo.

**D2 — A tag da imagem vira versionada, em card próprio.** *(decidido em 27/08)*

`:latest` + `IfNotPresent` mantém todo agente em campo congelado na versão
instalada, inclusive diante de CVE. A correção é emitir tag semver mantendo
`IfNotPresent`, o que torna auditável qual versão cada cluster roda e faz da
atualização parte do fluxo de release.

**Não entra nesta spec** — é mudança no contrato de release, não no defeito dos
probes. Rastreado em card separado.

## Fora de escopo

- Mudanças em `navyr-agent` (R4).
- Emissão de `Service` para o agente — o modelo é outbound-only por design (ADR 0001).
