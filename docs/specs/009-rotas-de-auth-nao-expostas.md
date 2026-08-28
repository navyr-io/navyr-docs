# SPEC-009 — As telas de Settings chamam rotas que o gateway não expõe

**Estado:** implementada e verificada — 27/08
**Card:** navyr-io/navyr-frontend#18
**Depende de:** navyr-io/navyr-auth#19
**Continua:** [SPEC-001](001-fronteira-de-autenticacao.md)

## Problema

Oito telas de configuração estão inoperantes atrás do BFF, e o erro que exibem
culpa o papel do usuário.

Medido com cookie de sessão válido de um `owner`:

| Rota chamada pelo frontend | `/auth/…` | `/api/v1/auth/…` |
|---|---|---|
| `/auth/org/kms-config` | 401 | 403 |
| `/auth/org/smtp-config` | 404 | 403 |
| `/auth/webhooks` | 401 | 403 |
| `/auth/ai-providers` | 401 | 403 |
| `/auth/admin/groups` | 401 | 403 |
| `/auth/ldap/config` | 401 | 403 |
| `/auth/2fa/totp/status` | 401 | 403 |
| `/auth/password/change` | 405 | 403 |
| `/auth/sso/providers/oidc/config` | 401 | 403 |
| `/auth/sso/providers/oidc/profiles` | 200 | 200 |

Só a última funciona — é a única que a SPEC-001 chegou a registrar.

O corpo do 403:
```json
{"code":"forbidden","message":"role 'owner' is not allowed to execute feature 'unknown'"}
```

A mensagem é enganosa. Rota não registrada cai no `feature 'unknown'` e é recusada
por RBAC — fail-closed correto, diagnóstico errado. Custou tempo nesta própria
investigação.

## Inventário

**52 chamadas em 9 arquivos**, em duas classes:

- **Classe A — 40 chamadas** usam `${API_BASE}/auth/…`: base certa, prefixo
  errado. Deveria ser `/api/v1/auth/…`.
- **Classe B — 12 chamadas** usam `${AUTH_BASE}/auth/…`: ignoram o gateway.
  `AUTH_BASE` é `VITE_AUTH_URL ?? origin`, e sem a variável vira a mesma URL
  quebrada.

Telas afetadas: `GovernanceTab`, `EmailTab`, `NotificationsTab`, `AIProvidersTab`,
`SSOTab`, `AdminPage`, `ProfilePage`, `GroupsTab`, `AIAssistant`.

## Três causas distintas, com correções diferentes

**1. Rotas que o gateway já expõe, chamadas pelo caminho errado.**
A família `/api/v1/admin/groups*` está registrada e responde 200 — medido.
`AdminPage` e `GroupsTab` chamam `/auth/admin/groups`. Correção só no chamador.

**2. Rotas que o auth serve e o gateway não expõe.**
`kms-config`, `webhooks`, `ai-providers`, `ldap/*`, `2fa/totp/*`,
`password/change`, `sso/providers/oidc/config`, `sso/providers/oidc/test`.
Precisam ser registradas uma a uma — a regra R4 da SPEC-001 proíbe repasse por
prefixo, porque a medição que a originou mostrou que o prefixo exporia 73 rotas do
auth, 20+ delas administrativas.

**3. Rotas que não existem em lugar nenhum.**
`/auth/org/smtp-config` e `/auth/ldap/test-query` não estão registradas no
`navyr-auth`. Verificado por varredura em `cmd/server/rotas.go`. As telas que as
chamam nunca funcionaram, e expor não resolve — não há o que expor.

## A descoberta que ordena o trabalho

O `navyr-auth` **não** enforce papel de forma uniforme. No mesmo arquivo
(`internal/service/auth_segredos.go`):

| Método | Checa papel? |
|---|---|
| `UpdateOrgKMSConfig` | sim — `CanManageMembers(actor.Role)` |
| `GetOrgKMSConfig` | não |
| `GetOrgSMTPConfig` | não |
| `UpdateOrgSMTPConfig` | **não** |

Escrever a configuração de SMTP da organização decide para onde vão os e-mails de
redefinição de senha. Sem gate, qualquer conta autenticada aponta o servidor para
um que controla e passa a receber os tokens de recuperação de toda a organização.

Não é alcançável hoje porque a rota não existe. **Passa a ser se esta spec a
expuser sem gate.** Rastreado em `navyr-auth#19`, que precisa fechar antes.

### Registro de método

Uma varredura minha marcou os `Admin*` de LDAP como sem checagem, e quase virou o
eixo desta spec. Eram **stubs OSS** devolvendo `ErrRecursoEnterprise`; a
implementação Enterprise checa `CanManageMembers` normalmente. Verificar método a
método separou o falso positivo do achado real.

## Decisões tomadas

**D1 — O gateway gateia por papel também.** *(27/08)* Vira R3.

Duas linhas de defesa: uma lacuna no auth não vira exposição. Foi exatamente uma
lacuna dessas que este levantamento encontrou, e ela existiu por tempo
indeterminado sem ninguém notar.

**D2 — As telas sem rota exibem que a funcionalidade não está disponível.**
*(27/08)* Vira R5.

Mesmo tratamento que a SPEC-007 deu ao mock: não fingir que funciona. Tentar uma
chamada que sempre falha produz uma mensagem de erro genérica que sugere problema
transitório, quando a causa é permanente.

## Regras

**R1 — Cada rota é exposta uma a uma, nunca por prefixo.**
Herdada da R4 da SPEC-001, e a razão continua valendo: o prefixo exporia 73 rotas
do auth, 20+ administrativas.

**R2 — Os chamadores passam a usar `${API_BASE}/api/v1/auth/…`.**
`AUTH_BASE` sai de uso nas 12 chamadas da classe B. Uma SPA atrás de BFF não tem
motivo para conhecer o endereço do serviço de identidade.

**R3 — Rota administrativa exposta leva gate de papel no gateway.**
`isAdminRole`, como as de `/api/v1/admin/*` já levam. São administrativas:
`org/kms-config`, `webhooks`, `ai-providers`, `ldap/*`,
`sso/providers/oidc/config`, `sso/providers/oidc/test`.

**Não** são administrativas, porque descrevem a própria conta de quem chama:
`2fa/totp/*` e `password/change`. Gatear essas impediria um membro de trocar a
própria senha ou ativar o próprio 2FA — que é o oposto do objetivo.

**R4 — Existe teste que compara o gate do gateway com o do auth.**
Duplicar a regra de papel em dois lugares cria a chance de divergirem. O teste é o
que transforma a duplicação em defesa em vez de armadilha.

**R5 — Tela sem rota no backend diz isso, e não tenta a chamada.**
`EmailTab` (`/auth/org/smtp-config`) e a busca LDAP do `GroupsTab`
(`/auth/ldap/test-query`). Card próprio para implementá-las.

**R6 — A recusa por rota não registrada para de culpar o papel.**
`role 'owner' is not allowed to execute feature 'unknown'` descreve uma decisão de
RBAC que não é o que aconteceu. Custou tempo nesta investigação e custaria a
qualquer um.

## Critérios de aceitação

1. As 8 telas de Settings carregam sem 403 com sessão de `owner`.
2. `grep -rn 'AUTH_BASE' src/` não retorna chamada de rota.
3. Com sessão de `member`, as rotas administrativas respondem 403 **pelo gateway**,
   antes de chegar ao auth — verificável no log.
4. Com sessão de `member`, `password/change` e `2fa/totp/*` continuam funcionando.
5. `EmailTab` e a busca LDAP exibem indisponibilidade, sem emitir requisição.
6. Rota não registrada sob `/api/v1/auth/` responde com mensagem que diz que a rota
   não existe, não que o papel é insuficiente.

## Ordem de execução

`navyr-auth#19` fecha primeiro. Expor `org/kms-config` e `org/smtp-config` antes de
a checagem de papel existir abriria o caminho descrito acima.

## Fora de escopo

- Implementar `smtp-config` e `ldap/test-query` no `navyr-auth`. É funcionalidade,
  rastreada em card próprio.
