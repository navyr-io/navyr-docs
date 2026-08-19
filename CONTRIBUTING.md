# Como contribuir

Este é um repositório **source-available sob licença comercial** — veja
[LICENSE](LICENSE). Não é open source: a licença não concede direito de
redistribuição nem de fork. Contribuições externas não são aceitas por
enquanto, porque não há CLA definido. Se você chegou aqui por interesse no
produto, fale em **contato@navyr.io**.

O que segue vale para quem trabalha no código.

## Antes de abrir PR

O critério de pronto é o da
[GOVERNANCE.md](GOVERNANCE.md),
não uma lista paralela aqui. Em resumo: compila, testes passam, build da
imagem passa, o serviço sobe, e há evidência de teste funcional.

O CI verifica boa parte disso automaticamente — mas **rode localmente antes de
empurrar**, porque a organização tem teto de minutos e um PR que quebra no
primeiro passo gasta o mesmo que um que passa.

```bash
# Serviços Go
go build ./... && go vet ./... && go test ./...
golangci-lint run ./...

# Frontend
npm ci && npx tsc -b --noEmit && npm test && npx playwright test
```

> **Go 1.26.** `golangci-lint` e `gosec` ainda não têm release compilada com
> 1.26 e se recusam a analisar o módulo. Compile-os com o toolchain do
> projeto: `go install github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.12.2`.
> O CI já faz isso.

## Commits

Mensagem semântica descrevendo **o quê e por quê** — o "como" está no diff.
Prefixos em uso: `feat`, `fix`, `refactor`, `test`, `docs`, `ci`, `build`,
`chore`.

Quando o commit corrige um defeito, o corpo diz **como o defeito se
manifestava**. É o que permite reconhecer o mesmo problema quando ele voltar
em outro lugar.

## Achados que você não vai corrigir agora

Registre em
[docs/achados-abertos.md](docs/achados-abertos.md).
Achado não registrado é achado perdido — e o registro tem valor maior quando
descreve o modo da falha, não só o sintoma.

## Dependências

O Dependabot abre PR mensalmente, agrupando `minor` e `patch`. **Major vem em
PR próprio de propósito**: bump de major costuma exigir migração, e agrupá-lo
torna impossível distinguir qual dos pacotes quebrou.

Atualização de segurança ignora o intervalo mensal e chega imediatamente.

## Segurança

Vulnerabilidade não vai em issue nem em PR. Veja [SECURITY.md](SECURITY.md).
