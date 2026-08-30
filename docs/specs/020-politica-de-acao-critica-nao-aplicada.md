# SPEC-020 — A política de ação crítica é salva e nunca aplicada

**Card:** `navyr-io/navyr-gateway#21` · **Severidade:** Alto
**Descoberto:** varredura de interface ponta a ponta, levantamento (28/08/2026)

## Medido, com a configuração mais restritiva ligada

```sql
UPDATE organizations SET critical_dual_approval=TRUE, critical_break_glass=FALSE
WHERE id='b5514337-…';
-- gravado: t | f
```

Administrador escala um deployment, **sem aprovação nenhuma**:

```
POST /api/v1/deployments/legado-privilegiado/scale  {"replicas":3}
→ HTTP 200 {"replicas":3}

replicas antes: 1     replicas depois: 3
SELECT count(*) FROM action_approvals;  →  0
```

Nenhuma solicitação de aprovação foi criada, nenhum segundo aprovador foi
consultado, e a ação destrutiva passou.

## O que a investigação encontrou — quatro camadas, não uma

O card dizia "nada lê essas colunas". É verdade, mas é a menor das três razões
pelas quais o controle não funciona.

### 1. A configuração da organização não entra na decisão

Quem decide é `loadPolicyConfig()` (`cmd/server/main.go:1315`), que lê
`POLICY_CONTEXT_ENABLED`, `BREAK_GLASS_ENABLED` e `BREAK_GLASS_TOKEN` **do
ambiente, uma vez na partida**. As colunas `critical_dual_approval`,
`critical_break_glass` e `critical_window_minutes` só são lidas para exibir e
gravar, no `navyr-auth`.

### 2. Existe um segundo portão — e ele isenta justamente quem liga o controle

`evaluateCriticalActionGate` (`main.go:1369`) exige `X-Approval-ID`,
`X-Approval-Justification` e `X-Approved-By`, e um segundo aprovador quando o
papel é `support`. É um portão real. Mas:

```go
if normalizeRole(r.Header.Get("X-User-Role")) == "owner" {
    return criticalActionDecision{allowed: true}
}
```

E `normalizeRole` mapeia **`org_admin` → `owner`**. Ou seja, todo administrador
de organização atravessa o portão sem tocar nele — e administrador é exatamente
quem liga "exigir aprovação dupla".

### 3. O portão confere presença de cabeçalho, não aprovação

Ele testa se as três strings são **não vazias**. Não valida `X-Approval-ID`
contra `action_approvals`, não confere se o aprovador é outra pessoa, não olha
prazo.

`navyr-frontend/src/screens/LabsPage.tsx:348` demonstra isso literalmente:

```tsx
headers: { "X-Approval-ID": "lab-session",
           "X-Approval-Justification": "starting operational lab",
           "X-Approved-By": "navyr-ui" }
```

A interface emite a própria aprovação, com valores fixos no código.

### 4. Nada cria solicitação de aprovação para ação destrutiva

A tabela `action_approvals` existe (migração `000008`), tem repositório, tem
handlers de aprovar e rejeitar, e tem tela (`ApprovalsPage`). Mas
`CreateActionApproval` é chamado **de um único lugar**:
`aiops_service.go:138`, para remediação sugerida pelo AIOps.

Nenhuma operação destrutiva — delete, exec, scale, drain, rollout — cria
solicitação. E **nenhum cliente envia `X-Approval-ID`** fora do literal do
LabsPage.

## Por que não dá para simplesmente "passar a ler a coluna"

**Não há caminho no produto para fornecer uma aprovação legítima.** Se o gateway
passasse a exigir aprovação de verdade quando `critical_dual_approval` está
ligado, toda ação crítica se tornaria impossível: a interface não sabe pedir
aprovação, e nada cria a linha que seria aprovada. Ligar o controle deixaria a
organização travada.

Trocar "mente" por "trava" não é correção.

### E o gateway não consegue ler a configuração sozinho

Verificado:

- o gateway **não tem banco** — não há `pgxpool` nem `DATABASE_URL` no
  repositório;
- o `navyr-auth` **não reconhece** a assinatura de contexto interno que o
  gateway usa para falar com o orchestrator (`grep` por `Internal-Context` no
  `navyr-auth`: zero ocorrências);
- `AdminGetOrganizationSettings` exige token de administrador, então ler a
  configuração com o token do usuário falharia para operador.

Fazer a configuração decidir exige mudança no `navyr-auth` — outro repositório,
e sem valor enquanto não existir o fluxo de aprovação.

## Decisão

**Parar de prometer o que não existe, e cardar o que precisa existir.**

É o padrão que o projeto já usa: a SPEC-009 desabilitou o EmailTab e a busca
LDAP com aviso explícito enquanto não havia backend, em vez de deixar a tela
afirmar que funcionavam.

O dano deste card é a **afirmação**: *"A second admin must approve before
execution."* Um administrador lê isso, liga o controle, e passa a acreditar que
operações destrutivas exigem dois aprovadores. Não exigem, e ele não tem como
descobrir sozinho.

**A correção mora no `navyr-frontend`, não no `navyr-gateway`** — apesar de o
card ter sido aberto no gateway. O gateway não tem, hoje, nenhuma mudança segura
a fazer: qualquer aperto no portão quebra o Labs, que depende do buraco.

## Regras

- **R1** — Os quatro controles do painel "Critical action policy" ficam
  desabilitados, com aviso explícito de que a política ainda não é aplicada.
- **R2** — O aviso diz **o que de fato acontece hoje**, não "em breve": a
  configuração é gravada e nenhuma decisão a consulta.
- **R3** — O valor gravado continua sendo lido e exibido. Desabilitar o controle
  não é apagar o dado de quem já configurou.
- **R4** — Existe teste que falha se os controles voltarem a ficar habilitados
  sem que a aplicação exista.

## Cardado à parte

- **Fluxo de aprovação de ação crítica** (épico): criar solicitação para
  operação destrutiva, validar `X-Approval-ID` contra `action_approvals`,
  exigir aprovador diferente do solicitante, respeitar
  `critical_window_minutes`, e fazer a configuração da organização decidir.
  Atravessa gateway, orchestrator, auth e frontend.
- **`LabsPage` emite a própria aprovação** com cabeçalhos literais.
- **`least_privilege_hint`** ("Alert on permission conflicts") não é lido em
  lugar nenhum, nem para exibir alerta — é decoração completa.

## Critérios de aceitação

1. Os quatro controles aparecem desabilitados, com o aviso visível.
2. O valor gravado continua sendo exibido corretamente.
3. Prova negativa: reabilitando os controles, o teste de R4 falha.
