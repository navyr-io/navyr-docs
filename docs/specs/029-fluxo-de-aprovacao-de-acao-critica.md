# SPEC-029 — Fluxo de aprovação de ação crítica

**Épico:** `navyr-io/navyr-gateway#23`
**Motivado por:** SPEC-020, que desabilitou o painel por não haver o que aplicar

## O estado que a SPEC-020 mediu

Com `critical_dual_approval = TRUE`, um administrador escalou um deployment de
1 para 3 réplicas sem aprovação nenhuma: `HTTP 200`, e `action_approvals` com
**zero** linhas.

Quatro camadas explicam:

1. A configuração da organização não entra na decisão — o gateway decide por
   variável de ambiente lida uma vez na partida.
2. `evaluateCriticalActionGate` isenta `owner`, e `normalizeRole` mapeia
   `org_admin → owner`. Quem liga o controle é quem é isento dele.
3. O portão confere **presença** dos cabeçalhos, não aprovação: qualquer string
   não vazia passa. `LabsPage.tsx:348` manda `"lab-session"` e `"navyr-ui"`.
4. Nada cria solicitação para operação destrutiva. `CreateActionApproval` é
   chamado de um único lugar: remediação do AIOps.

## O que já existe, e não precisa ser construído

- Tabela `action_approvals` (migração `000008`), com `status`, `approved_by`,
  `expires_at`, `requested_by`, `action`, `resource`, `namespace`.
- Repositório e handlers de listar, aprovar e rejeitar.
- Tela `ApprovalsPage`.
- Rotas `/internal/v1/…` no orchestrator, com validação de assinatura de
  contexto interno que o gateway já sabe produzir.

As peças estão quase todas no lugar. Falta ligá-las.

## Decisão de arquitetura — de onde o gateway lê a política

A SPEC-020 mediu três becos: o gateway **não tem banco**; o `navyr-auth` **não
reconhece** a assinatura de contexto interno; e `AdminGetOrganizationSettings`
**exige token de administrador**, então falharia para operador.

**A política viaja na validação de sessão.**

O gateway já chama `POST /auth/token/validate` em **toda** requisição, sem
cache. `ValidateToken` devolve hoje `{user, permissions}`; passa a devolver
também `{policy}`, com os quatro campos críticos da organização.
`GetOrganizationByID` já os lê e não exige papel nenhum.

Isso resolve tudo de uma vez, e é o motivo de ser a escolha certa:

- **sem endpoint novo** e sem mecanismo de autenticação novo;
- **sem cache para envelhecer** — desligar a política tem efeito na próxima
  requisição, e não em algum TTL;
- **funciona para qualquer papel**, porque a validação de sessão não é gateada;
- **sem latência extra**, porque a chamada já acontece.

Descartado: novo endpoint interno no auth com assinatura de contexto. Seria
construir um segundo caminho de confiança entre dois serviços que já têm um.

## Ordem de execução

**A → B → C → D.** O item A destrava a decisão: sem ele o gateway não consegue
nem saber se a organização quer o controle ligado.

### A · `navyr-auth` — a política viaja na sessão

`ValidateToken` passa a incluir `policy` com `critical_dual_approval`,
`critical_break_glass`, `critical_window_minutes` e `least_privilege_hint`.

### B · `navyr-orchestrator` — solicitar e consultar aprovação

- `POST /api/v1/approvals` — cria solicitação `pending` para uma ação
  destrutiva, com `expires_at` derivado de `critical_window_minutes`.
- `GET /internal/v1/approvals/{id}` — o gateway consulta estado, organização,
  `approved_by`, `expires_at`, `action` e `resource`.

### C · `navyr-gateway` — o portão passa a decidir de verdade

`evaluateCriticalActionGate` valida `X-Approval-ID` contra o orchestrator:

- existe, está `approved`, pertence à organização da sessão, não expirou;
- **`approved_by` é diferente de quem executa** — hoje nada impede aprovar a
  própria ação;
- a ação e o recurso da aprovação **casam com a requisição**, para uma
  aprovação de "escalar web" não autorizar "excluir banco".

E a isenção de `owner` deixa de valer **quando a organização ligou aprovação
dupla**. A isenção existe para o caminho de bootstrap; ela não pode esvaziar um
controle que o administrador ligou de propósito.

### D · `navyr-frontend` — pedir aprovação, e reabilitar o painel

Fluxo de solicitar, acompanhar o pendente e enviar o `X-Approval-ID` real. Os
quatro controles de Governance voltam a ser editáveis.

## Restrições inegociáveis

- **Falhar fechado.** Política ilegível ou orchestrator fora ⇒ decisão mais
  restritiva, nunca a mais permissiva.
- **A organização só soma restrição.** Nunca remove o que as variáveis de
  ambiente já impõem.
- **Não afrouxar nada que hoje já esteja fechado.**
- **Enforcar sem caminho de solicitação trava a organização** — foi a razão de a
  SPEC-020 desabilitar em vez de enforçar. B vem antes de C, sempre.

## Verificação

O critério final é o inverso da medição que abriu o épico: com
`critical_dual_approval = TRUE`, um administrador **não** consegue escalar um
deployment sem uma aprovação real, dada por outra pessoa, dentro da janela.
