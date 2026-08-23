# SPEC-002 · O cookie de sessão precisa passar pelo authMiddleware

**Status:** aprovada por Erick em 23/08 · **Card:** navyr-io/navyr-frontend#17 · **Data:** 2026-08-23

---

## Requisito

`AuthMiddleware` recusa com **401** toda requisição sem cabeçalho
`Authorization`, e roda **antes** do despacho. O despacho é quem sabe resolver o
cookie de sessão — mas nunca é alcançado.

Medido com curl, mesmo cookie válido nas três:

| Rota | Onde está registrada | Resultado |
|---|---|---|
| `/api/v1/auth/session` | fora do `authMiddleware` | **200** |
| `/api/v1/auth/session/profile` | atrás dele | **401** |
| `/api/v1/clusters` | atrás dele | **401** |

É a raiz do #17: as correções anteriores mapearam rotas que o middleware barra
antes.

## Regras

**R1.** Requisição com cookie de sessão **válido** atravessa o `AuthMiddleware`
e chega ao despacho, que já resolve a sessão e injeta o token.

**R2.** Cookie **ausente** mantém o comportamento de hoje: exige `Authorization`,
e sem ele responde 401. O Bearer continua valendo — agente, integrações e o
teste `test_e2e_full_stack.sh` dependem dele.

**R3.** Cookie **presente e inválido** — forjado, expirado, revogado — responde
401 e **não** cai para Bearer. Aceitar o cabeçalho depois de um cookie ruim
deixaria os dois caminhos abertos numa requisição só. Já há teste no despacho
para isso (`TestCookieInvalidoRecusaSemCairParaBearer`) e a propriedade precisa
valer também aqui.

**R4.** O middleware **não valida** a sessão: só reconhece que há cookie e
delega. Validar em dois lugares cria duas verdades sobre a mesma sessão — quem
decide é o despacho, com o Redis.

## Restrição

O `AuthMiddleware` vive em `internal/middleware` e não conhece a loja de sessão,
que é de `cmd/server`. A correção **não pode** inverter essa dependência: o
middleware não deve importar a loja.

## Critérios de aceitação

1. Com cookie válido, `/api/v1/auth/session/profile` e `/api/v1/clusters`
   respondem **200** — hoje respondem 401.
2. Sem cookie e sem `Authorization`, seguem respondendo **401**.
3. Com `Authorization: Bearer` válido e sem cookie, seguem funcionando.
4. Com cookie inválido e `Authorization` válido na mesma requisição, responde
   **401** — o cookie ruim não cai para Bearer.
5. `/exec/ws` continua aceitando só o ticket, sem `Authorization`.
6. A jornada login → organização → criar cluster completa no navegador.

## Casos de erro

| Situação | Esperado |
|---|---|
| Cookie expirado no Redis | 401, e a interface volta ao login |
| Redis de sessão indisponível | 503, distinguível de credencial recusada |
| Cookie com valor malformado | 401, sem consultar o Redis |
