# Reverter uma release

## Kubernetes (Helm)

```bash
NS=navyr
helm history navyr -n "$NS"          # veja as revisões e escolha a última boa
helm rollback navyr <revisão> -n "$NS"
```

Ou pelo script:

```bash
./scripts/ops/rollback_release.sh helm navyr navyr <revisão>
```

**O que o rollback do Helm não desfaz: migrations.** Elas rodam na subida do
serviço e não têm `.down.sql` em quatro casos. Reverter a imagem para uma
versão que não conhece o schema novo pode falhar de formas diferentes conforme
a mudança:

- **Coluna adicionada:** normalmente inofensivo. A imagem antiga a ignora.
- **Coluna removida ou renomeada:** a imagem antiga quebra em `SELECT`.
- **Constraint adicionada:** a imagem antiga insere dado que a constraint
  rejeita.

Se a migration for do segundo ou terceiro tipo, o caminho é
[restaurar o backup](backup-restauracao.md), não reverter a imagem.

## Compose

```bash
export REGISTRY=ghcr.io/navyr-io
./scripts/ops/rollback_release.sh docker <tag-anterior>
```

O script retaga as imagens da tag informada para `:latest` e recria os
serviços.

**Onde achar a tag anterior:** hoje as imagens são publicadas com o **SHA curto
puro** — `7f21918`, não `sha-7f21918`.

As tags `sha-<commit>` e `main` vêm do workflow de Publicação e estão congeladas
em **19/08/2026**, última execução bem-sucedida antes do bloqueio de cobrança
(navyr-deploy#4). Tudo publicado depois disso foi enviado manualmente, com SHA
puro. Reverter para uma tag `sha-` te leva a agosto, não à versão anterior.

Confira a lista de verdade antes de escolher — o comando abaixo devolve as tags
que existem, e é ele que manda.

```bash
curl -s "https://ghcr.io/token?scope=repository:navyr-io/navyr-gateway:pull" \
  | python3 -c 'import sys,json;print(json.load(sys.stdin)["token"])' \
  | xargs -I{} curl -s -H "Authorization: Bearer {}" \
      "https://ghcr.io/v2/navyr-io/navyr-gateway/tags/list" | python3 -m json.tool
```

> Não existe tag semver publicada. `values-prod.yaml` já apontou para
> `v0.1.0-20260501`, que **não existe** — instalar com aquele arquivo dava
> `ImagePullBackOff` nos cinco serviços. Hoje ele está fixado em tags `sha-`
> reais. Confira a tag antes de reverter para ela.

## Verificação

```bash
kubectl -n navyr get pods                    # todos 1/1
kubectl -n navyr get deploy -o wide          # a imagem é a esperada?
kubectl -n navyr logs -l app=navyr-gateway --tail=30 | grep -i erro
```

O gateway em `1/1` é o sinal mais informativo: o `/ready` dele consulta auth,
billing, community e orchestrator. Gateway pronto significa que os quatro
respondem.
