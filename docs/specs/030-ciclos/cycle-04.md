# Ciclo 4 — os sete serviços passam no gate, com o gate inteiro

## O que entrou

Nove anotações `#nosec`, em quatro repositórios. Nenhuma linha de lógica
alterada — só comentários.

| Repositório | Arquivo | Anotações |
|---|---|---|
| `navyr-gateway` | `cmd/server/main.go` | 1 × `G704` na linha do `client.Do(req)` |
| `navyr-gateway` | `cmd/server/sessao.go` | 2 × `G124` (cookie), 2 × `G117` (`RefreshToken`) |
| `navyr-auth` | `cmd/server/migration_lock.go` | 1 × `G115` (chave do `pg_advisory_lock`) |
| `navyr-community` | `cmd/server/oauth_state.go` | 2 × `G124` (cookie de estado do OAuth) |
| `navyr-orchestrator` | `internal/handler/agent_manifest_handler.go` | 1 × `G705` (entrega do manifesto) |

Commits: `cab3b32`, `e68db38`, `2aa359e`, `8c96a2e`.

## Decisões

**Anotar o sink que faltava, em vez de mover as 54 anotações existentes.** A
spec, corrigida após o ciclo 2, previa *"mover as 54 anotações do gateway para a
linha do sink"*. Ao olhar o código antes de executar, isso se mostrou errado.

O padrão do repositório é **duas anotações por chamada externa** — uma no
`http.NewRequestWithContext` e outra no `client.Do` — porque o gosec reporta a
segunda. Exemplo em `main.go:490-496`:

```go
	// #nosec G704 -- destino fixado por configuracao (env), nao pelo request...
	req, err := http.NewRequestWithContext(ctx, http.MethodGet, endpoint.String(), nil)
	if err != nil {
		return false, err
	}
	// #nosec G704 -- destino fixado por configuracao (env), nao pelo request...
	resp, err := client.Do(req)
```

Há **53 sites seguindo esse padrão e nenhum deles dispara**. Era a evidência de
que as anotações não estavam erradas — estavam **incompletas em um site só**.

O `novoConsultorDeAprovacao` (SPEC-029 C), que entrou durante o apagão do CI,
copiou apenas a primeira metade do padrão. Mover 54 anotações teria quebrado 53
que funcionam para consertar 1 que faltava.

**Anotar em vez de alterar código correto**, nos casos não-taint. `G115`, `G117`
e `G124` apontam para desenho deliberado e já documentado em comentário: a chave
de advisory lock derivada de sha256, o `Marshal` do registro de sessão para o
Redis, e `Secure` vindo de `cookieSeguro()`/`seguro` para não quebrar dev em
http. A alternativa — mudar `Secure` para literal `true` — quebraria o
desenvolvimento em http sem dar pista, que é exatamente o que o comentário
existente já avisava. O comentário o gosec não lê; o `#nosec` ele lê.

## Critério de pronto

Comando, com os flags idênticos aos do `go-service.yml`, nos **sete** serviços
Go — não só nos quatro alterados:

```
$ gosec -severity medium -confidence medium -exclude-generated -exclude=G702,G118 ./...
```

| Repositório | Antes | Depois |
|---|---|---|
| `navyr-gateway` | `Nosec 54 · Issues 5 · exit 1` | `Nosec 59 · Issues 0 · exit 0` |
| `navyr-auth` | `Issues 1 · exit 1` | `Nosec 5 · Issues 0 · exit 0` |
| `navyr-community` | `Issues 2 · exit 1` | `Nosec 3 · Issues 0 · exit 0` |
| `navyr-orchestrator` | `Issues 1 · exit 1` | `Nosec 20 · Issues 0 · exit 0` |
| `navyr-agent` | — | `Nosec 6 · Issues 0 · exit 0` |
| `navyr-billing` | — | `Nosec 1 · Issues 0 · exit 0` |
| `navyr-collector` | — | `Nosec 0 · Issues 0 · exit 0` |

**7 de 7 em `exit 0`.** Fecha o critério 1 da §2.

O gate continua inteiro — critério 3 da §2:

```
$ grep -n 'exclude=' .github-org/.github/workflows/go-service.yml
150:            -exclude-generated -exclude=G702,G118 ./...
```

`G704`, `G705` e `G710` ligadas, incluindo a que sinalizou o takeover de SSO.

Validação local nos quatro repositórios alterados:

```
navyr-gateway       gofmt ok · vet ok · go test ./... -race -> 0
navyr-auth          gofmt ok · vet ok · go test ./... -race -> 0
navyr-community     gofmt ok · vet ok · go test ./... -race -> 0
navyr-orchestrator  gofmt ok · vet ok · go test ./... -race -> 0
```

Push com o hook `pre-push` validando cada SHA: `build OK` nos quatro.

## Estado ao fim do ciclo

O gate passa localmente nos sete serviços com todas as regras de taint ligadas.
Falta a confirmação no CI real, que é o critério 7 e o objeto do ciclo 6.

Os testes de integração do `navyr-orchestrator` continuam reprovando por motivo
alheio a esta spec — `UpdateStatus` grava um status que a constraint do banco
rejeita. Está especificado na SPEC-032, card `navyr-deploy#19`, e não é
condição para o critério 1 daqui.
