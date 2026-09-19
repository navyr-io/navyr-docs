# Ciclo 3 — cancelado: nenhuma exclusão nova é necessária

## O que entrou

Nada. Este ciclo existe como registro de que a entrega planejada **não foi
feita, de propósito**.

O plano original da §4 previa: *"aplicar o caminho que o ciclo 2 determinar, no
`go-service.yml`, com o motivo no comentário da exclusão"*. As duas rotas
previstas eram subir a versão do gosec, ou excluir `G704`/`G705` com a mesma
razão técnica aceita para o `G702` em 20/08.

O ciclo 2 eliminou as duas:

- **Subir versão não muda nada.** 2.28.0 e 2.29.0 produzem saída idêntica na
  mesma árvore — 5 achados, `Nosec: 54`, `Issues: 5`.
- **Excluir não é necessário.** `#nosec` funciona em regras de taint; a anotação
  só precisa estar na linha que o gosec reporta.

## Decisões

**Não tocar em `.github/workflows/go-service.yml`.** A alternativa era aproveitar
a passagem para "arrumar" a lista de exclusão — por exemplo reabrindo o `G702`,
que foi desligado em 20/08 pela mesma razão que o ciclo 2 refutou.

Rejeitada por duas razões. A primeira é escopo: o `G702` está na §3 da spec
(fora de escopo), e aumentar o alcance de um card em execução é precisamente o
que a regra do quadro existe para impedir. A segunda é que mexer no workflow
reutilizável afeta os **11 repositórios de uma vez** — é a mudança com maior
alcance disponível nesta spec, e ela não é necessária para o objetivo.

O gate permanece exatamente como está: `-exclude=G702,G118`, com `G704`, `G705`
e `G710` ligadas.

## Critério de pronto

```
$ grep -n 'exclude=' .github-org/.github/workflows/go-service.yml
150:            -exclude-generated -exclude=G702,G118 ./...
```

Inalterado em relação ao ciclo 1. O `git status` do `.github-org` está limpo —
nenhuma alteração local no repositório do gate.

## Estado ao fim do ciclo

O trabalho migra inteiro para o ciclo 4, que agora tem natureza diferente da
planejada: em vez de configurar o gate para reprovar menos, anotar o código para
que o gate **não tenha o que reprovar**. O gate continua inteiro, o que preserva
a regra que achou o takeover de SSO.
