# SPEC-026 — Telas de ConfigMaps, Secrets, PVCs e PVs

**Card:** `navyr-io/navyr-frontend#34` · **Severidade:** Médio
**Motivada por:** SPEC-021, ao remover links que não levavam a lugar nenhum

## O que faltava

Quatro recursos que a plataforma **já expõe pela API** não tinham tela.

Medido ao vivo, com a sessão do navegador:

```
GET /api/v1/configmaps?namespace=homologacao              -> 1 item
GET /api/v1/secrets?namespace=homologacao                 -> 200
GET /api/v1/persistentvolumeclaims?namespace=homologacao  -> 200
GET /api/v1/persistentvolumes                             -> 200
```

As quatro rotas estão na allowlist do gateway
(`navyr-gateway/cmd/server/rotas.go:60,62,122,128`) e o orchestrator as
implementa. O frontend nunca leu: `grep` por `listConfigMaps`, `listSecrets`,
`listPVC` e `listPersistentVolume` em `src/lib/api/` dava **zero**.

Havia até **escrita sem leitura**: `resizePersistentVolumeClaim` e
`updatePersistentVolumeReclaimPolicy` já estavam escritos em `storage.ts`, sem
nenhuma tela que os chamasse.

## Por que os links tinham saído

Antes da SPEC-021 a barra lateral oferecia os quatro apontando para
`cluster-config/configmaps` e `resources/pvcs` — seções que não existem em
`SECTIONS` nem em `ViewMode`. O clique caía no Overview e em Storage Classes,
**em silêncio** (`navyr-frontend#25`).

Removi os links naquele momento porque oferecer link antes de a tela existir é
pior que não oferecer: o operador clica, vê outra coisa, e conclui que a
plataforma está quebrada. Um link ausente ele nem procura.

## Decisões

**D1 — Uma tela, quatro tipos, com `:kind` na rota.**
No padrão do `NetworkPage`, que já resolve o mesmo problema. O tipo vem da URL
e não de estado interno — foi a lição da SPEC-021: estado que não corresponde à
rota faz toda rota profunda cair no padrão, em silêncio.

**D2 — Colunas próprias de cada recurso.**
Uma tabela genérica de `{nome, namespace, kind}` não serviria: PVC precisa de
status, tamanho, storage class e volume ligado; PV precisa de reclaim policy e
do claim.

**D3 — PV não oferece filtro de namespace.**
É recurso de escopo de cluster. Oferecer o filtro sugeriria um recorte que não
existe.

**D4 — Secret nunca mostra valor, e isso não é escolha da tela.**
`models.Secret` devolve `type`, `data_keys` e `data_count` — nunca o valor.
Revelar exige `GetSecretData`, rota separada que o gateway classifica como
feature sensível com gate próprio. A listagem é segura por construção, e o
teste guarda as duas metades: o que aparece, e o que a tela **não foi buscar**.

Os nomes das chaves vão no `title`, porque o subtítulo da tela promete *"key
names"* — promessa que a coluna de contagem sozinha não cumpria.

**D5 — Os estados de volume entram na paleta do `StatusBadge`.**
`Bound`, `Available`, `Released` e `Lost`, e não uma prop `tone` na tela: a cor
de um estado é propriedade do estado, e duas telas mostrando "Bound" em cores
diferentes seria a divergência que o `navyr-frontend#29` cataloga.

**D6 — Erro de carregamento é dito, e não vira estado vazio.**
"Não há" e "não deu para saber" são coisas diferentes. Foi essa confusão que a
SPEC-007 encontrou na tela de Security, que preenchia o vazio com dado
inventado.

## Regras

- **R1** — O tipo listado vem de `:kind` na rota.
- **R2** — PV não oferece filtro de namespace; os outros três oferecem.
- **R3** — A tela de Secrets não chama nenhuma rota que devolva valor.
- **R4** — Falha ao listar aparece como falha, e não como lista vazia.
- **R5** — Os quatro links voltam à navegação, e cada um leva à sua tela.

## Critérios de aceitação

1. As quatro rotas renderizam contra o cluster real, sem erro de carregamento.
2. A barra lateral oferece os quatro.
3. A listagem de Secrets não emite requisição que devolva valor.
4. Prova negativa: com o tipo fixo, com o filtro de namespace em PV, e com o
   erro escondido atrás do estado vazio, os testes falham.
