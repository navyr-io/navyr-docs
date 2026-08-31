# Spec Index

Este diretório mantém especificações de suporte. Para os tópicos de produto canônicos, use `spec/product/`.

## Canônico (produto)
- `spec/product/vision.md`
- `spec/product/features.md`
- `spec/product/flows.md`
- `spec/product/personas.md`
- `spec/product/resource_graph.md`
- `spec/product/security.md`
- `spec/product/system_rules.md`
- `spec/product/testing.md`

## Histórico
Duplicatas antigas foram movidas para:
- `docs/archive/spec-legacy/`

## Contrato Backend Ativo

A spec unificada foi aposentada em 19/08 — ver [openapi.md](openapi.md). O
contrato vive nas specs por serviço, no repositório de cada um.

Valide com:

```bash
# Cada spec, no seu repositório
npx --yes @redocly/cli lint openapi.yaml

# Regras que o redocly não cobre, sobre as cinco de uma vez.
# Exige os repositórios dos serviços ao lado deste.
./tests/contract/run_contract_tests.sh
```
