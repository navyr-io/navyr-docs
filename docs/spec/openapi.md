# A spec unificada foi aposentada

`spec/openapi.yaml` descrevia o backend inteiro num arquivo só, com
`version: 0.9.0-phase14-local` — era artefato de **design**, do contrato
pretendido, não da API implementada.

Foi removida em 19/08 depois de verificado que virou subconjunto: das suas 82
rotas, **as 82 estão cobertas** pelas specs por serviço, que ainda trazem 20
que ela não tinha.

## Onde está o contrato agora

| Serviço | Spec |
|---|---|
| gateway | [openapi.yaml](https://github.com/navyr-io/navyr-gateway/blob/main/openapi.yaml) |
| auth | [openapi.yaml](https://github.com/navyr-io/navyr-auth/blob/main/openapi.yaml) |
| billing | [openapi.yaml](https://github.com/navyr-io/navyr-billing/blob/main/openapi.yaml) |
| orchestrator | [openapi.yaml](https://github.com/navyr-io/navyr-orchestrator/blob/main/openapi.yaml) |
| community | [openapi.yaml](https://github.com/navyr-io/navyr-community/blob/main/openapi.yaml) |

Índice e convenções em
[navyr-docs/docs/api.md](https://github.com/navyr-io/navyr-docs/blob/main/docs/api.md).

## Por que não manter as duas

Duas descrições do mesmo contrato divergem — a questão é quando, não se. E
haviam divergido: as 4 rotas que só existiam aqui eram os caminhos que o
**gateway transforma**, reescrevendo para o billing com o `tenant_id` extraído
do JWT. Nenhuma spec de serviço as documentava, porque nenhum serviço as serve
daquela forma. Elas foram para a spec do gateway, que é onde o comportamento
mora.

Mesmo raciocínio do [ADR 0006](https://github.com/navyr-io/navyr-docs/blob/main/docs/adr/0006-helm-como-caminho-unico.md),
que aposentou o Kustomize pelo mesmo motivo.

## Os outros arquivos deste diretório

O restante de `spec/` são documentos de produto e design — visão, personas,
fluxos, regras de sistema. Continuam válidos como registro de intenção, e não
descrevem implementação. Não confunda um com o outro: quando divergirem, o
código e as specs por serviço é que dizem o que existe.
