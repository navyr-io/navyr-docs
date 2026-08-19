# 0003 — Um repositório por serviço

**Status:** Aceita · **Data:** 2026-05, com consequências registradas em 2026-08

## Contexto

A plataforma nasceu como monorepo (`kubeops`), com os serviços em
`services/<nome>`. A migração para um repositório por serviço aconteceu em
26/05.

## Decisão

Cada serviço tem repositório próprio em `github.com/navyr-io/*`, com seu
`go.mod`, seu Dockerfile e seu CI.

## Consequências

**O que ganhamos.** Publicação independente por serviço. Escopo de permissão
por repositório. Histórico e CI que falam de uma coisa só.

**O que custou — e este ADR existe principalmente para registrar isto:**

1. **A migração não foi finalizada, e ninguém percebeu por dois meses.** O
   monorepo continuou recebendo commits até 01/06, depois do split. Uma linha
   de feature inteira — stack de observabilidade e onboarding de cluster —
   ficou presa lá: 6 handlers, uma migration, o repositório de tokens de
   registro, ~24 arquivos de frontend e o serviço `collector` **inteiro**, que
   nunca teve repositório. Só foi drenada em 18/08.

   A lição não é sobre multi-repo: é que **migração sem data de corte e sem
   arquivamento da origem não termina.** O que impediu a divergência de
   recomeçar foi arquivar o `kubeops`, não a decisão de dividir.

2. **Todo ferramental compartilhado precisa de um lar.** O CI bom do monorepo
   não sobreviveu ao split — os repos ficaram com `docker build && push`, sem
   teste, lint ou scan, e 4 deles sem CI nenhum. A solução foi workflow
   reutilizável no repo `.github` da organização (ver [0005](0005-ci-validacao-separada-de-publicacao.md)).

3. **Correção de segurança vira N pull requests.** Um helper de validação de
   SSRF usado por dois serviços ou vira módulo publicado, ou é duplicado
   conscientemente. Não há terceira opção honesta.

4. **Não há commit atômico entre serviços.** Mudança de contrato entre gateway
   e auth exige coordenação e compatibilidade nas duas pontas — o que é bom
   como disciplina e ruim como velocidade.

## Alternativas descartadas

**Voltar ao monorepo.** Foi considerada durante a auditoria de agosto, quando a
divergência apareceu. Descartada porque os repositórios remotos já eram os do
`navyr-io` e o produto já estava publicado a partir deles: reverter custaria
mais do que terminar a migração.

**Monorepo com publicação seletiva por caminho.** Resolve o item 3 e boa parte
do 2, ao custo de um CI mais complexo. Continua sendo a alternativa razoável se
o item 3 virar recorrente.
