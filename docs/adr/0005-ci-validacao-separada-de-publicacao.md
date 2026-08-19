# 0005 — Validação e publicação em workflows distintos

**Status:** Aceita · **Data:** 2026-08

## Contexto

O CI de cada serviço vivia num arquivo só, chamando o workflow reutilizável
`go-service.yml` e concedendo `packages: write` no topo, porque o job de imagem
publica no GHCR.

Ao atacar a fila do Dependabot descobriu-se que **28 dos 36 pull requests não
tinham CI nenhum**. Todos morriam em `startup_failure`, antes de qualquer job
iniciar. O alarme de deriva de dependência — instalado justamente depois de 123
CVEs acumuladas em silêncio — estava mudo desde que foi instalado.

A causa: permissões são resolvidas **na partida, para todo o grafo de jobs**,
antes de qualquer condição `if:` ser avaliada. Em execução disparada pelo
Dependabot o `GITHUB_TOKEN` é limitado a leitura. Um único pedido de escrita
acima do teto derruba a execução inteira.

Isso não se resolve com `if:` no job, nem movendo a permissão do topo do
arquivo para o job: em ambos os casos ela continua no grafo.

## Decisão

Dois arquivos por repositório, separados **por gatilho**:

- `ci.yml` — `push` e `pull_request`, apenas `contents: read`. Chama
  `go-service.yml`, que valida e escaneia. Nenhum job pede escrita.
- `publish.yml` — apenas `push` para `main`. Chama `publish-image.yml`, o único
  com `packages: write`.

Como uma execução de pull request nunca avalia `publish.yml`, ela nunca encosta
numa permissão de escrita.

## Consequências

**O que ganhamos.** CI roda em todo PR, inclusive os do Dependabot. E a
permissão de escrita passou a existir só onde de fato publica, o que é o
princípio do menor privilégio aplicado por construção e não por intenção.

**O que custou:**

1. **Os dois workflows rodam em runners separados e não compartilham build.**
   A primeira versão da separação construía a imagem duas vezes em todo push
   para `main` — 241s no `ci.yml` para escanear e 466s no `publish.yml` para
   publicar, cerca de 12 minutos por push. Corrigido restringindo o job de
   imagem do `ci.yml` a pull request e movendo o portão de scan para dentro do
   `publish-image.yml`.

2. **Sobra um scan duplicado enquanto não houver branch protection.** Hoje o
   push direto para `main` é possível, então o `publish` precisa escanear por
   conta própria. Quando todo merge passar por PR, o scan do PR vira o portão e
   o do publish fica supérfluo.

3. **Dois arquivos por repositório em vez de um**, e a relação entre eles não é
   explícita para quem lê só um deles. Daí este ADR.

## Alternativas descartadas

**Manter tudo em um arquivo e pular o job de publicação com `if:`.** Testada
contra o GitHub e **não funciona**: a permissão do job pulado continua no grafo
e a execução ainda falha na partida.

**Não rodar CI em PR do Dependabot.** Era o estado de fato, e é o que se queria
corrigir.

**Publicar com PAT em vez do `GITHUB_TOKEN`.** Contorna o teto de permissão,
ao custo de um segredo de longa duração com escopo de escrita no registry — pior
do que o problema.
