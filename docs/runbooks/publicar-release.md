# Publicar uma release

## Antes de tudo: o que uma tag dispara

Empurrar uma tag `vX.Y.Z` aciona `release.yml`, que faz duas coisas, nesta
ordem:

1. **Publica a imagem** com a tag da versão em `ghcr.io/navyr-io/<serviço>`.
2. **Cria a GitHub Release**, com as notas extraídas do `CHANGELOG.md`.

O segundo passo **falha se o `CHANGELOG.md` não tiver seção para a versão**.
Isso é proposital: release sem nota não ajuda quem precisa decidir se
atualiza. Corrija o changelog e reempurre a tag.

## Procedimento

```bash
VERSAO=0.2.0

# 1. O CHANGELOG precisa ter a seção da versão ANTES da tag.
#    Confira que o extrator a enxerga:
awk -v v="## [$VERSAO]" '
  index($0, v) == 1 {f=1; next}
  f && index($0, "## [") == 1 {exit}
  f {print}
' CHANGELOG.md

# 2. Commit do changelog.
git add CHANGELOG.md
git commit -m "docs: changelog da $VERSAO"
git push origin main

# 3. Espere o CI do main fechar verde. Tag em commit vermelho publica imagem
#    que não passou no pipeline.
gh run list --limit 1

# 4. Tag anotada, não leve — a anotada guarda autor e data.
git tag -a "v$VERSAO" -m "v$VERSAO"
git push origin "v$VERSAO"
```

## Verificação

```bash
# A release existe e tem corpo?
gh release view "v$VERSAO"

# A imagem com a tag semver foi publicada?
T=$(curl -s "https://ghcr.io/token?scope=repository:navyr-io/<serviço>:pull" \
     | python3 -c 'import sys,json;print(json.load(sys.stdin)["token"])')
curl -s -o /dev/null -w '%{http_code}\n' -H "Authorization: Bearer $T" \
  -H 'Accept: application/vnd.oci.image.index.v1+json' \
  "https://ghcr.io/v2/navyr-io/<serviço>/manifests/$VERSAO"
```

`200` significa publicada. Qualquer outra coisa significa que o job de imagem
não terminou — **não anuncie a versão antes de checar isto.** Já houve tag
apontada em `values-prod.yaml` (`v0.1.0-20260501`) que nunca existiu no
registry, e instalar com ela dava `ImagePullBackOff` nos cinco serviços.

## Errei a tag, e agora

```bash
git tag -d "v$VERSAO"
git push origin ":refs/tags/v$VERSAO"
gh release delete "v$VERSAO" --yes
```

> **Só faça isso se ninguém consumiu a versão.** Tag que alguém já usou não se
> move: publique `vX.Y.Z+1` corrigindo. Mover tag consumida quebra quem fixou
> nela, e o pior é que quebra em silêncio — o `docker pull` traz conteúdo
> diferente sob o mesmo nome.

## Qual número escolher

`MAJOR.MINOR.PATCH`, com o significado usual do SemVer. Para esta plataforma,
as regras que exigem atenção:

- **Migration sem `.down.sql` é quebra de compatibilidade para trás**, mesmo
  que a API não mude. O runbook de reverter release explica por quê: o
  `helm rollback` não desfaz migration.
- **Mudança no contrato entre gateway e um serviço** exige compatibilidade nas
  duas pontas, porque não há commit atômico entre repositórios
  ([ADR 0003](../adr/0003-multi-repo.md)). Publique a ponta compatível
  primeiro.
- Enquanto a versão for `0.x`, a promessa de estabilidade é fraca por
  definição — mas isso não é licença para quebrar sem registrar no changelog.
