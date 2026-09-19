# Ciclo 1 — o gate reproduz localmente, e as 54 anotações são comprovadamente inertes

## O que entrou

Nenhuma alteração de código. Este ciclo é medição, e entrega três fatos:

1. O clone local de `navyr-io/.github` (`/home/erick/navyr-repos/.github-org`)
   foi sincronizado — estava **10 commits atrás**, em `aa6641b` (19/08), com
   `-exclude=G702,G704,G705,G710,G118`. Agora em `c45f0c1` (20/08), com
   `-exclude=G702,G118`, que é o que o CI executa.
2. `gosec` v2.28.0 e v2.29.0 instalados **com o toolchain do projeto** via
   `go install`, em GOBIN separados, para comparação no ciclo 2. A armadilha do
   `CLAUDE.md` vale aqui: binário pronto não acompanha o Go do projeto.
3. Reprodução local do gate com os flags exatos do CI.

## Decisões

**Sincronizar o `.github-org` antes de qualquer medição, em vez de rodar com o
que estava no disco.** A alternativa — medir primeiro e sincronizar depois —
produziria verde local contra vermelho no CI, porque o clone carregava a lista
de exclusão antiga, que ainda continha `G704`, `G705` e `G710`. Foi exatamente
essa divergência que escondeu o problema por duas semanas.

**Comparar duas versões em GOBIN separados, em vez de reinstalar por cima.**
Reinstalar impediria rodar as duas contra o mesmo código na mesma sessão, que é
o que o ciclo 2 exige para ter valor probatório.

## Critério de pronto

Comando, com os flags idênticos aos do `go-service.yml`:

```
$ cd navyr-gateway
$ gosec -severity medium -confidence medium -exclude-generated -exclude=G702,G118 ./...
```

Saída real:

```
[cmd/server/main.go:2941] - G704 (CWE-918): SSRF via taint analysis (Confidence: HIGH, Severity: HIGH)
[cmd/server/sessao.go:206] - G124 (CWE-614): http.Cookie missing or has insecure Secure, HttpOnly, or SameSite attribute (Confidence: HIGH, Severity: MEDIUM)
[cmd/server/sessao.go:194] - G124 (CWE-614): http.Cookie missing or has insecure Secure, HttpOnly, or SameSite attribute (Confidence: HIGH, Severity: MEDIUM)
[cmd/server/sessao.go:168] - G117 (CWE-499): Marshaled struct field "RefreshToken" (JSON key "refresh_token") matches secret pattern (Confidence: MEDIUM, Severity: MEDIUM)
[cmd/server/sessao.go:115] - G117 (CWE-499): Marshaled struct field "RefreshToken" (JSON key "refresh_token") matches secret pattern (Confidence: MEDIUM, Severity: MEDIUM)

Summary:
  Gosec  : dev
  Files  : 15
  Lines  : 6598
  Nosec  : 54
  Issues : 5
```

`exit=1`.

**Bate exatamente com o que o CI reportou** no run 33473335841
(`navyr-gateway`, 01/09): os mesmos 5 achados, nas mesmas linhas. A reprodução
local é fiel.

### O dado que este ciclo entrega e que não estava na spec

```
  Nosec  : 54
  Issues : 5
```

O gosec **contou as 54 anotações `#nosec G704`** e mesmo assim reportou o G704
de `main.go:2941`. Isso eleva a afirmação do commit `c45f0c1` de mensagem de
commit a fato medido nesta árvore: as anotações não são ignoradas por erro de
sintaxe nem por posição — são **lidas e desconsideradas** pelas regras de taint.

Fecha o critério 5 da §2 da spec pelo lado do diagnóstico: hoje há 54 anotações
inertes no `navyr-gateway`, e nenhuma delas suprime coisa alguma.

## Estado ao fim do ciclo

O gate reproduz. A causa está medida, não inferida. O ciclo 2 pode rodar a
comparação 2.28.0 × 2.29.0 contra a mesma árvore, que é o que decide entre
manter as regras ligadas ou aplicar ao `G704`/`G705` a razão técnica já aceita
para o `G702`.
