# SPEC-012 — Nenhum agente em campo recebe atualização

**Estado:** implementada e verificada — 28/08
**Card:** navyr-io/navyr-orchestrator#14

## Problema

O manifesto emite:

```go
image = "ghcr.io/navyr-io/navyr-agent:latest"   // manifest.go:22
```
```yaml
imagePullPolicy: IfNotPresent                   # manifest.go:145
```

A combinação significa que, depois da primeira instalação, o nó já tem `:latest`
em cache e **nunca mais consulta o registry**. O agente fica congelado na versão
que baixou no dia da instalação.

## Como isso apareceu

O agente do cluster de teste acumulou 1429 reinícios
(`navyr-orchestrator#12`). Nenhum deles trouxe imagem nova — `IfNotPresent`
garantiu que a mesma camada em cache subisse 1429 vezes.

Foi sorte a correção do #12 não depender de imagem nova: a imagem em campo já
servia `/healthz` e `/ready`, e só o manifesto estava errado. Se dependesse, o laço
de reinício não a teria entregue nunca.

## Por que é problema de segurança

O agente roda no cluster do cliente com `ClusterRole` amplo. Um CVE nele — na
stdlib Go, numa dependência, no próprio código — **não tem caminho de entrega**.

E a plataforma não sabe o que está em campo: todo cluster reporta `:latest`, e dois
clusters com a mesma tag podem rodar binários diferentes. Não há como responder
"quais clusters estão vulneráveis" depois de um CVE.

## O que existe hoje

| | |
|---|---|
| `NAVYR_AGENT_IMAGE` | já permite sobrescrever a imagem inteira no manifesto |
| `release.yml` do agente | dispara em tag `vX.Y.Z` e publica a imagem versionada |
| tags `v0.1.0` | **não existem** — bloqueadas pelo teto de minutos do CI (`navyr-deploy#3`) |
| leitura do Deployment pelo túnel | já existe, de `CaminhoDeLivenessDoAgente` (SPEC-004 R5) |
| versão embutida no binário | **não existe** — o build usa `-ldflags="-s -w"`, sem `-X` |
| heartbeat | envia só `seen_at`; é ponto de extensão limpo |

## O que está bloqueado e o que não está

**Bloqueado:** publicar a imagem com tag semver depende das tags existirem, e elas
dependem do CI.

**Não bloqueado:** tornar a versão em campo conhecível, e parar de emitir uma tag
móvel por padrão. É o que esta spec trata.

## Decisões tomadas

**D1 — O agente reporta a versão no heartbeat.** *(28/08)*

Versão embutida no binário via `-ldflags -X`, enviada no heartbeat que já existe.
É a verdade do que está rodando, não do que foi declarado — sobrevive a alguém
editar o Deployment à mão e a um nó com cache antigo, que são justamente os casos
em que a tag mente.

**D2 — O manifesto emite digest.** *(28/08)*

`ghcr.io/navyr-io/navyr-agent@sha256:…` em vez de `:latest`. Imutável e auditável
hoje, sem depender de tag git — e faz `IfNotPresent` funcionar como deveria, porque
digest diferente é imagem diferente para o kubelet.

Verificado que o pacote é **público** e o ghcr emite token anônimo de pull:

```
GET https://ghcr.io/token?scope=repository:navyr-io/navyr-agent:pull  → token
GET https://ghcr.io/v2/navyr-io/navyr-agent/manifests/latest
  → docker-content-digest: sha256:326494f8…
```

Nenhuma credencial precisa ser armazenada.

## Regras

**R1 — O agente embute a versão no build e a reporta no heartbeat.**
`-ldflags "-X main.versao=…"`. Sem valor injetado, reporta `desconhecida` — um
binário construído fora do pipeline não deve fingir ser uma versão.

**R2 — O orquestrador guarda a versão reportada por cluster.**
Coluna nova, preenchida no heartbeat. Agente antigo não reporta e fica como
desconhecido até reinstalar — estado honesto, não erro.

**R3 — O manifesto emite digest, resolvido na geração.**
A resolução usa token anônimo. `NAVYR_AGENT_IMAGE` continua sobrescrevendo tudo,
e se já vier com digest é usada como está.

**R4 — Falha ao resolver o digest não impede a instalação.**
Diferente da SPEC-010, onde adivinhar produzia agente que nunca conecta: aqui a tag
móvel **funciona**, só não é auditável. Recusar por indisponibilidade do registry
trocaria um problema de auditoria por um de disponibilidade. Cai para a tag, e
registra em log que caiu.

**R5 — O resultado da resolução é cacheado.**
Sem cache, cada geração de manifesto faz duas chamadas externas. O digest de uma
tag muda raramente, e um cache curto elimina o custo sem esconder atualização.

## Critérios de aceitação

1. O manifesto gerado contém `@sha256:` e não `:latest`.
2. Com o registry inalcançável, o manifesto ainda é gerado, com a tag, e o log
   registra a queda.
3. `NAVYR_AGENT_IMAGE` continua tendo precedência.
4. Um agente construído pelo pipeline reporta versão não vazia no heartbeat.
5. A versão reportada aparece na resposta de health do cluster.
6. Agente que não reporta aparece como desconhecido, sem erro.

## Fora de escopo

- Criar as tags `v0.1.0` (`navyr-deploy#3`).
- Atualização automática do agente. Com `IfNotPresent` e tag imutável, atualizar é
  reaplicar o manifesto — e a SPEC-004 já sinaliza agente obsoleto na interface.
