# Origem e estado destes documentos

Estes arquivos vieram de `navyr-deploy/spec/` em 31/08/2026.

## Por que saíram de lá

O `navyr-deploy` é o repositório de instalação — compose, `.env.example`, scripts
de operação. Quem vai subir o Navyr precisa disso e não precisa da especificação
de produto. Separar as duas coisas permite tratar a publicação do **instalador**
como uma decisão, e a da **estratégia de produto** como outra.

## Estado: legado

**Estes documentos estão desatualizados.** Foram escritos quando o projeto se
chamava **KubeOps** e vivia num monorepo, arquivado desde 18/08/2026. O nome
antigo aparece ao longo do texto.

Eles descrevem intenção de produto de uma fase anterior, não o comportamento
atual da plataforma. Para o que o Navyr faz hoje, a fonte é:

- `docs/architecture.md`, `docs/components.md`, `docs/api.md` — arquitetura corrente
- `docs/adr/` — decisões com data e contexto
- `docs/specs/` — specs por correção, escritas antes de cada mudança
- os `openapi.yaml` de cada serviço — o contrato que vale, verificado no CI

Estão preservados por valor histórico: mostram o raciocínio de produto que
originou a plataforma. Não devem ser lidos como requisito vigente.
