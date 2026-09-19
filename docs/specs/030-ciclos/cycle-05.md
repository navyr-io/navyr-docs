# Ciclo 5 — a checagem de drift entra onde alguém de fato olha

## O que entrou

Uma alteração no `CLAUDE.md`, em duas seções.

**No bloco de comandos de validação**, como primeiro passo, antes de qualquer
`go build`:

```bash
# SEMPRE PRIMEIRO — o gate do CI mora no .github-org e o clone nao atualiza
# sozinho. Saida vazia = alinhado; qualquer linha = seu SAST local esta mentindo.
git -C .github-org fetch -q origin main && git -C .github-org log --oneline HEAD..origin/main
```

**Na seção de armadilhas**, a correção do que esta spec refutou: `#nosec`
funciona em regras de taint, e precisa estar na linha que o gosec reporta. A
entrada anterior afirmava o contrário, citando o `c45f0c1`.

## Decisões

**Pôr a checagem no bloco de comandos, não só na seção de armadilhas.** A
armadilha do `.github-org` já estava documentada em prosa desde o começo desta
sessão, e isso não teria evitado nada: quem vai validar antes de um push abre o
bloco de comandos e copia, não lê a seção de armadilhas. A prosa explica; o
bloco é o que se executa.

Rejeitada a alternativa de criar um script (`scripts/gate-drift.sh`). Um comando
de duas linhas dentro do fluxo que já se usa tem mais chance de ser rodado do
que um script que precisa ser lembrado. Se ele crescer, vira script.

**Corrigir a armadilha antiga em vez de acrescentar uma nova abaixo.** Deixar as
duas afirmações no arquivo — a errada e a certa — obrigaria o próximo leitor a
decidir qual vale. A entrada errada saiu.

## Critério de pronto

```
$ git -C .github-org fetch -q origin main && git -C .github-org log --oneline HEAD..origin/main
$
```

Saída vazia: o clone local está alinhado com o gate que o CI executa. Fecha o
critério 6 da §2.

Para contraste, a mesma checagem no início desta spec teria devolvido 10 linhas,
de `b9835f0` até `c45f0c1` — e foi essa divergência que fez o `gosec` local
passar verde enquanto o CI reprovava, por duas semanas.

## Estado ao fim do ciclo

Restam os critérios 5 e 7 da §2, ambos no ciclo 6:

- **5** — nenhuma anotação inerte no código. O ciclo 4 mostrou que as 53
  anotações existentes do gateway não eram inertes; o que faltava era uma. O
  critério precisa ser verificado com o texto que ele de fato pede.
- **7** — os sete serviços fechando CI verde em `main`. Em execução no momento
  em que esta nota foi escrita.
