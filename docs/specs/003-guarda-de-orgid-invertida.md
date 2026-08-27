# SPEC-003 — A guarda de `orgId` invertida, e o que ela estava escondendo

**Estado:** proposta
**Data:** 23/08/2026
**Card:** navyr-io/navyr-frontend#19
**Relacionada:** [SPEC-001](001-fronteira-de-autenticacao.md), [SPEC-002](002-cookie-no-authmiddleware.md)

## Problema

Seis lugares no frontend escrevem a guarda de early-return ao contrário:

```ts
if (!!orgId) return;   // desiste quando o orgId EXISTE
```

`!!x` é verdadeiro quando `x` existe. Cada ocorrência desiste exatamente no caso
em que deveria prosseguir. A intenção era `if (!orgId)`.

O mesmo código usa a forma correta com `clusterId` em nove lugares
(`ActionsPage.tsx:60`, `NodesPage.tsx:69`, `LabsPage.tsx:343` entre outros). O
contraste dentro do mesmo repositório é o que mostra erro mecânico, não intenção.

## O que foi medido

Cluster kind conectado, sessão de `owner`, stack local em 23/08.

| # | Arquivo:linha | Efeito observado |
|---|---|---|
| 1 | `hooks/useOrgEventStream.ts:24` | `fetch(/api/v1/events/stream)` nunca é chamado. O feed "Intelligence — cross-cluster operational feed" do cockpit não recebe nada, para nenhum usuário. |
| 2 | `settings/GovernanceTab.tsx:77` | As settings nunca são lidas. Os toggles de aprovação dupla e break-glass renderizam o padrão do `useState`, não a política da org. |
| 3 | `settings/GovernanceTab.tsx:112` | "Salvar URLs" sempre responde "Carregando perfil — tente novamente". Reproduzido no navegador. |
| 4 | `screens/ClusterCreatePage.tsx:57` | `orchestrator_url` da org nunca é lido, então nunca é anexado à URL do manifesto. |
| 5 | `screens/ClustersPage.tsx:45` | Idem no comando "Reinstalar agente". |
| 6 | `settings/OrgTab.tsx:20` | `save()` retorna "Saved locally." antes de qualquer chamada. |

O item 4 tem consequência além da própria tela: ele torna morto o único contorno
documentado do `navyr-orchestrator#10`. A organização configura a URL do túnel e
o produto a ignora.

## A descoberta que muda a correção

**`PUT /api/v1/admin/organizations/{id}/settings` substitui, não faz merge.**
Medido: um PUT sem `platform_url` e `orchestrator_url` devolveu os dois como `""`.

Isso importa porque `OrgTab.save()` (item 6) faz:

```ts
setOrganization(localOrg);                    // só estado React
if (!!orgID) { setMsg("Saved locally."); return; }
await updateOrganizationSettings(orgID, {
  critical_dual_approval: true, critical_break_glass: false,
  critical_window_minutes: 30, least_privilege_hint: true,
});
```

Três fatos que só aparecem juntos:

1. O campo de nome (`localOrg`) **nunca é enviado ao backend**. `OrganizationSettings`
   não tem campo de nome, e não existe rota de rename — `/auth/admin/organizations/{id}/settings`
   é a única que o auth serve para organização.
2. A chamada envia valores de governança **fixos no código**, que não têm relação
   com o que a tela edita.
3. Como o PUT substitui, essa chamada apagaria `platform_url` e `orchestrator_url`
   e reescreveria a política de ações críticas com os literais acima.

**A guarda invertida é hoje a única coisa que impede essa escrita destrutiva.**
Corrigir só a guarda no item 6 troca um botão inerte por um botão que corrompe a
política de aprovação da organização ao salvar um nome de exibição.

## Regras

**R1 — Nas 5 primeiras ocorrências, a guarda passa a ser `if (!orgId) return;`.**
Nenhuma outra mudança de comportamento nesses arquivos.

**R2 — `OrgTab.save()` deixa de chamar `updateOrganizationSettings`.**
Uma tela não escreve campos que não edita. Enquanto não existir rota de rename, o
botão não pode alegar persistência que não acontece.

**R3 — Nenhuma mensagem afirma persistência que não ocorreu.**
`"Saved locally."` e `"Saved locally — backend unavailable."` descrevem um
armazenamento local que não existe em nenhuma das duas telas. Erro exibe o erro
real, no padrão que `saveUrls` e `saveKms` já usam:
`e instanceof Error ? e.message : ...`.

**R4 — Quem escreve settings envia o objeto completo.**
Consequência direta de o PUT substituir. `GovernanceTab` já lê as settings antes
de gravar (item 2, uma vez corrigido), então cumpre R4 por construção. Qualquer
chamador novo de `updateOrganizationSettings` precisa ler antes de escrever.

**R5 — O rename de organização vira card próprio, não entra aqui.**
Falta rota no backend. Fora do escopo desta spec.

## Critérios de aceitação

1. `grep -rn 'if (!!' src/` não retorna nada.
2. Com sessão válida, `GovernanceTab` exibe os valores gravados na organização —
   e não os padrões do `useState`. Verificável gravando `critical_break_glass=true`
   via API e recarregando a aba.
3. "Salvar URLs" grava e responde "URLs salvas com sucesso.".
4. `ClusterCreatePage` anexa `?orchestrator_url=...` ao comando exibido quando a
   org tem a configuração.
5. `useOrgEventStream` emite `GET /api/v1/events/stream` — verificável na aba de
   rede do navegador.
6. Salvar em `OrgTab` não altera `critical_*`, `platform_url` nem `orchestrator_url`.
   Verificável lendo as settings antes e depois.
7. Nenhuma string "Saved locally" permanece no código.

## Fora de escopo

- Rota de rename de organização (R5).
- A semântica de substituição do PUT — é o contrato atual do backend; esta spec
  se adapta a ele, não o muda.
