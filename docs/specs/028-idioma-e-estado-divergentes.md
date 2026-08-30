# SPEC-028 — A interface passa a falar um idioma só

**Card:** `navyr-io/navyr-frontend#29` · **Severidade:** Baixo (dívida)
**Descoberto:** varredura de interface ponta a ponta, Fase 5 (28/08/2026)

## Remedição antes de agir — e uma correção de método

| Item | No card | Medido agora |
|---|---|---|
| `EmptyState` / texto à mão | 19 / 21 | 20 / 19 |
| `ErrorBanner` / mensagem à mão | 4 / 16 | 4 / **26** |
| Telas misturando idiomas | 12 | **17**, e depois **28** |
| Raio: token / px cravado | 318 / 193 | 320 / 191, em 12 valores |

**A medição de idioma do card estava errada.** Ela olhava o arquivo inteiro, e
este repositório comenta em português por convenção — qualquer arquivo com
comentário aparecia como "misto". Refeita olhando só o que chega à tela, com
comentários podados.

E **o meu primeiro detector também estava errado**: estreito demais. Ele achou
17 arquivos; depois de alargar o dicionário, apareceram outros 11. O caso que
melhor ilustra é `"Matriz Atual (Usuarios x Cluster)"` — não casava com nada,
porque nenhuma das suas palavras estava na lista.

Isso virou regra no teste-guarda: **um guarda que só pega o que já se conhece
não guarda contra o que vem depois.**

## O achado de maior alavanca

`src/lib/error-explainer.ts` estava **inteiro em português** — e é ele que
alimenta o `ErrorBanner`. Toda falha que a interface classifica saía em
português, numa interface inglesa em todo o resto. Um arquivo, todos os
consumidores.

As mensagens novas dizem o que aconteceu **e o que fazer**: *"Your session
expired or was revoked. Sign in again."* serve mais que *"Sessao invalida"*.

## O `ErrorBanner` duplicado

Havia dois componentes com o mesmo nome:

- `src/ui/ErrorBanner.tsx` — classifica por `explainError`, tem `role="alert"`;
- uma cópia local em `screens/observability/ClusterWorkspace.tsx:118` — um
  `div` sem papel ARIA, com API diferente (`{message}` contra `{error}`).

A cópia existia porque o chamador tem uma **mensagem de domínio** vinda do
servidor, e não um objeto de erro: passá-la por `explainError` a degradaria
para *"Something failed outside the API layer: …"*.

Resolvido dando ao compartilhado uma segunda entrada — `error` **ou**
`message`. Duas entradas, um componente, uma aparência; melhor que dois
componentes com o mesmo nome.

## Escopo entregue

1. Idioma: **zero** string portuguesa visível, verificado por detector largo.
2. Carregamento: uma forma (`Loading…`) e uma reticência. Havia nove variações
   em dois idiomas, com `...` e `…`.
3. `ErrorBanner` duplicado removido.
4. Os specs que dependiam do texto antigo, atualizados — `getByLabel("Senha")`,
   `getByPlaceholder("Sua senha")`, `getByLabel("Usuario")`, os títulos de erro
   do explainer e a regex de sucesso.
5. Teste-guarda com dicionário largo.

## Escopo NÃO entregue, e por quê

**191 raios cravados, em 12 valores distintos.** Edição em massa de 191 pontos
é churn com risco de regressão e zero valor para quem usa. Vira card com
proposta de catraca, não de mutirão.

**Adoção ampla de `EmptyState` e `ErrorBanner`** nos 19 + 26 pontos escritos à
mão. Mexer em caminho de erro em 45 lugares logo depois de treze correções é o
pior momento possível. Vira card próprio.

## Regras

- **R1** — Nenhuma string visível em português. Comentário em português segue
  sendo a convenção e não é alvo.
- **R2** — O explicador de erro responde em inglês.
- **R3** — Um `ErrorBanner` só.
- **R4** — O teste-guarda usa dicionário largo, e não a lista do que já se
  conhece.

## Critérios de aceitação

1. O detector largo devolve zero ocorrências.
2. Suíte de unidade, suíte com mock e suíte contra a stack, todas verdes.
3. Prova negativa: devolvendo o explainer ou o rótulo do login ao português, o
   guarda falha.
