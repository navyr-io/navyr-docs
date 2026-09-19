# Ciclo 2 — `#nosec` **funciona** em regras de taint. As 54 anotações estão na linha errada.

## O que entrou

Nenhuma alteração de código permanente. Três medições, e a terceira derruba a
premissa da spec.

## Decisões

**Testar no código real em vez de insistir no reprodutor mínimo.** Construí um
programa de 20 linhas com fonte de taint (`r.URL.Query().Get("id")`), sink
(`client.Do`) e handler registrado no `main`. O `G704` nunca disparou nele,
embora o analisador estivesse rodando — provado porque o `G114` (ListenAndServe
sem timeout) foi reportado no mesmo arquivo. Continuar ajustando o reprodutor
custaria mais que medir no alvo verdadeiro.

**Editar `main.go` temporariamente e reverter, em vez de deduzir da
documentação.** A alternativa era aceitar a afirmação do commit `c45f0c1` como
verdadeira — que foi exatamente o que a spec fez, e o que este ciclo mostrou
estar errado.

## Critério de pronto

### Medição 1 — `gosec` 2.29.0 contra a mesma árvore

Mesmos flags do CI, mesmo código do ciclo 1:

```
[cmd/server/main.go:2941] - G704 (CWE-918): SSRF via taint analysis (Confidence: HIGH, Severity: HIGH)
[cmd/server/sessao.go:206] - G124 (CWE-614): http.Cookie missing or has insecure ...
[cmd/server/sessao.go:194] - G124 (CWE-614): http.Cookie missing or has insecure ...
[cmd/server/sessao.go:168] - G117 (CWE-499): Marshaled struct field "RefreshToken" ...
[cmd/server/sessao.go:115] - G117 (CWE-499): Marshaled struct field "RefreshToken" ...

Summary:
  Nosec  : 54
  Issues : 5
```

**Idêntico à 2.28.0.** Subir de versão não muda nada — a decisão 2 da §6 da spec
fica respondida com "não".

### Medição 2 — anotação na linha do sink

O `navyr-gateway` anota a linha do `http.NewRequestWithContext` (2930). O gosec
reporta a linha do `client.Do(req)` (2941). Inseri `// #nosec G704` **na linha
que o gosec reporta**, rodei, e reverti com `git checkout --`:

```
[cmd/server/sessao.go:206] - G124 (CWE-614): http.Cookie missing or has insecure ...
[cmd/server/sessao.go:194] - G124 (CWE-614): http.Cookie missing or has insecure ...
[cmd/server/sessao.go:168] - G117 (CWE-499): Marshaled struct field "RefreshToken" ...
[cmd/server/sessao.go:115] - G117 (CWE-499): Marshaled struct field "RefreshToken" ...
  Nosec  : 55
  Issues : 4
```

**O `G704` sumiu.** `Nosec` subiu de 54 para 55, `Issues` caiu de 5 para 4.

Árvore revertida e limpa depois do teste: `git status --porcelain` vazio.

## O que isto significa

A afirmação do commit `c45f0c1`, repetida na §1 desta spec, no card `#18` e no
`CLAUDE.md`, é **falsa**:

> as regras de taint do gosec 2.28 **não honram `#nosec`**, nem no destino nem
> na origem

Honram. O mecanismo funciona. As 54 anotações do `navyr-gateway` não são inertes
por defeito da ferramenta — estão **na linha errada**: na construção do request,
não no sink que o gosec reporta.

Isso inverte a decisão da spec. Não é preciso:

- excluir `G704`/`G705` (decisão 3 da §6) — o gate pode ficar inteiro;
- subir a versão do gosec (decisão 2) — 2.29.0 é idêntica à 2.28.0;
- remover as 54 anotações (decisão 4) — elas passam a valer quando movidas.

O trabalho vira **mover anotação para a linha reportada**, que preserva o gate
que achou o takeover de SSO em 20/08. É o melhor desfecho possível dos quatro
que a §2 da spec listou, e nenhum de nós o tinha como provável.

## Consequência fora do escopo desta spec

O `G702` foi excluído em 20/08 pela **mesma razão agora refutada**. Se `#nosec`
funciona em regras de taint, o `G702` provavelmente também é recuperável com
anotação na linha certa — o que devolveria ao gate a detecção de execução de
comando no orchestrator (`trivy`, `syft`, `cosign`).

Não é deste card. Fica registrado para virar card próprio.

## Estado ao fim do ciclo

A spec precisa ser corrigida antes do ciclo 3 — ela está em `proposta`, então
pode ser editada sem quebrar a regra de imutabilidade. O `CLAUDE.md` e o card
`#18` também afirmam o que este ciclo refutou.
