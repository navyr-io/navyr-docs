# SPEC-001 · Fronteira de autenticação do frontend

**Status:** aprovada por Erick em 23/08 · **Card:** navyr-io/navyr-frontend#17 · **Data:** 2026-08-23

---

## Objetivo

Definir por onde cada chamada de autenticação do frontend passa, e sob qual
verificação. Hoje isso não está escrito em lugar nenhum, e é exatamente onde os
três defeitos de 23/08 apareceram.

## Requisito

Desde o ADR 0009 a credencial vive num cookie `HttpOnly` e o token fica no
Redis, com o gateway. **Toda chamada autenticada precisa passar pelo gateway**,
porque é ele que troca o cookie pelo token. Chamada que vai direto ao auth chega
sem credencial.

## Regras

**R1.** Chamada que exige sessão vai para o gateway (`API_BASE`), nunca para o
auth (`AUTH_BASE`).

**R2.** Chamada anterior à sessão — cadastro, recuperação de senha — **vai pelo
gateway**, para herdar o limite de taxa. Não é preferência: medido em 23/08, o
cadastro **não tem limite em lugar nenhum**, e cria conta e organização. O login,
que é o vizinho óbvio, já passa pelo `rateLimitMiddleware`. A recuperação de
senha tem limite no auth por identidade; o do gateway conta por origem, que é
outro vetor.

**R3.** Operação privilegiada passa por `isAdminRole` no gateway. "Privilegiada"
é a que altera identidade, papel, permissão, ciclo de vida ou **pertencimento à
organização** de outro usuário — convite entra por aqui.

O gate não muda quem pode fazer o quê: `CanManageMembers` no auth aceita
`SuperAdmin`, `PlatformAdmin` e `OrgAdmin`, o mesmo conjunto que `isAdminRole`
reconhece. É defesa em profundidade com zero mudança de comportamento.

**R4.** O gateway expõe rotas do auth **uma a uma**, nunca por prefixo.
`/api/v1/auth/*` → `/auth/*` tornaria as 73 rotas do auth alcançáveis, e as 20+
administrativas ganhariam caminho sem gate.

**R5.** Nada no frontend chama `/auth/token/validate`. É o endpoint que o
gateway usa internamente para resolver a sessão a partir do token; chamá-lo do
navegador manda pedido sem credencial, porque quem tem o token é o gateway.

**Correção de 23/08:** a primeira versão desta regra dizia que ninguém no
frontend o chamava. Errado — `validateSession` chamava, e o `OrgTab` usa, para
ler o `org_id` da sessão. A regra vale; a premissa estava errada. O destino não é
remoção: é `/api/v1/auth/session`, o endpoint do BFF, que devolve a mesma forma
`{ user }` a partir do cookie.

## Tabela de decisão

| Chamada | Rota hoje | Destino | Gate | Regra |
|---|---|---|---|---|
| `register` | `/auth/register` | gateway | limite de taxa | R2 |
| `requestPasswordReset` | `/auth/password/reset/request` | gateway | limite de taxa | R2 |
| `getSessionProfile` | `/auth/session/profile` | gateway | sessão | R1 |
| `switchSessionOrganization` | `/auth/session/organization/switch` | gateway | sessão | R1 |
| `createInvite` | `/auth/invites` | gateway | **admin** | R3 |
| `changeUserRole` | `/auth/users/{id}/role` | gateway | **admin** | R3 |
| `setUserLifecycle` | `/auth/admin/users/{id}/lifecycle` | gateway | **admin** | R3 |
| `validateSession` | `/auth/token/validate` | gateway, em `/api/v1/auth/session` | sessão | R5 |

## Critérios de aceitação

1. `grep "AUTH_BASE}" src/lib/api/auth.ts` devolve **zero**.
2. Para cada rota da tabela, existe rota correspondente no despacho do gateway.
3. Papel não-admin recebe **403** nas três marcadas como admin, pelo gateway.
4. Não existe passagem por prefixo: `/api/v1/auth/admin/*` **não** responde 200
   nem para `owner`.
5. A jornada login → organização → criar cluster completa no navegador.
6. `register` responde **429** depois de estourar o limite. Hoje não responde
   nunca — não há limite em lugar nenhum para essa rota.
7. `createInvite` responde **403** para papel não-admin, pelo gateway.

## Casos de erro

| Situação | Esperado |
|---|---|
| Cookie ausente ou expirado | 401, e a interface volta ao login |
| Papel insuficiente em rota admin | 403, com aviso visível na tela |
| Auth indisponível | 503, distinguível de credencial recusada |

## Fora de escopo

Rotas de SSO e TOTP — não foram inventariadas e entram em spec própria.

## Decisões de 23/08

**`createInvite` leva gate.** O conjunto de papéis é idêntico ao que o auth já
exige, então nenhum usuário legítimo perde acesso.

**Cadastro e recuperação de senha passam pelo gateway.** O dado que decidiu: o
cadastro não tem limite de taxa em lugar nenhum, e cria conta e organização.

## O que já foi feito

`c62455b` implementou as rotas de sessão e o gate das duas privilegiadas no
gateway, e é o que as regras R1, R3 e R4 descrevem. **Foi feito antes desta
spec**; ela documenta o que existe e define o que falta — o lado do frontend, e
`createInvite`, que segue sem gate.
