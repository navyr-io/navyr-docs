# SPEC-018 — `location /auth/` no nginx contorna o gate de papel

**Card:** `navyr-io/navyr-frontend#22` · **Severidade:** Alto
**Descoberto:** varredura de interface ponta a ponta, levantamento (28/08/2026)

## O defeito

O nginx que serve a SPA também expõe um caminho direto ao serviço de identidade,
contornando o gateway por completo — e com ele o gate de papel que a SPEC-009
construiu.

```
TOK=$(curl -s -X POST :8081/auth/login -d '…' | jq -r .token)

curl -H "Authorization: Bearer $TOK" :5173/auth/org/kms-config   → 200
curl -H "Authorization: Bearer $TOK" :5173/api/v1/auth/org/kms-config → 403 (org_viewer)
```

`navyr-frontend/nginx/templates/default.conf.template:14`.

### A superfície é maior do que a medição inicial

O card media duas rotas. São **77**, entre elas `/auth/admin/users`,
`/auth/admin/api-keys`, `/auth/admin/grants`, `/auth/users/{id}/role` e
`/auth/org/kms-config` — todas alcançáveis sem o `isAdminRole` do gateway.

O gateway usa allowlist explícita rota a rota, e o comentário em
`navyr-gateway/cmd/server/despacho.go:279` diz por quê: um passthrough por
prefixo "tornaria as 73 rotas do auth alcançáveis daqui". O nginx faz
exatamente esse passthrough, por fora.

## Correção da premissa do card

O card afirmava que `/auth/` era **caminho morto** desde a SPEC-009. **Não é.**

A SPA não faz nenhum `fetch` para `/auth/` — isso se confirma. Mas
`src/screens/settings/SSOTab.tsx:309` renderiza, num campo *read-only*, o
Redirect URI que o administrador deve registrar no provedor de identidade:

```tsx
<input type="text" readOnly value={`${window.location.origin}/auth/sso/oidc/callback`} />
```

Não é uma chamada da SPA: é uma **navegação de topo** que o IdP dispara no
navegador ao devolver o usuário. Cadeia verificada:

- o `navyr-auth` serve `/auth/sso/{provider}/callback` (`rotas.go:167`);
- o gateway **não tem** rota de callback;
- logo, hoje o único caminho até esse Redirect URI é o `location /auth/`.

Remover o bloco inteiro mataria o login federado em silêncio — a navegação
cairia no `try_files $uri /index.html` e o usuário veria a SPA em vez de voltar
autenticado.

## Decisão

**Estreitar, não remover.**

Passam direto ao serviço de identidade **apenas os dois pontos de navegação** do
login federado — `/auth/sso/{provider}/login` e `/auth/sso/{provider}/callback`.
Essas duas rotas são desenhadas para chegar sem sessão: é o IdP que redireciona
o navegador para elas. Todo o resto de `/auth/` é API e tem de atravessar o
gateway.

Considerada e descartada: mover o Redirect URI para trás do gateway. É a solução
mais limpa a prazo, mas exige rota nova no `navyr-gateway` — outro repositório,
outro card. Estreitar entrega o ganho de segurança inteiro agora, sem depender
disso, e não fecha a porta para fazê-lo depois.

Considerada e descartada: remover assumindo que o SSO por OIDC está morto. A SPA
de fato não **inicia** login federado em lugar nenhum, então o caminho pode nunca
ter sido exercitado. Mas a tela de Settings o anuncia ao administrador, e apagar
um caminho que o produto documenta, com base em suposição, é como se criam
defeitos que só aparecem no cliente.

## Regras

- **R1** — Só `/auth/sso/{provider}/login` e `/auth/sso/{provider}/callback`
  chegam ao serviço de identidade pelo nginx.
- **R2** — Qualquer outro caminho sob `/auth/` responde `404` no nginx. Não cai
  no `try_files` da SPA: devolver a página inicial com `200` para
  `/auth/admin/users` esconderia o que está acontecendo.
- **R3** — `AUTH_UPSTREAM` permanece, porque o SSO continua precisando dela.
- **R4** — A regra tem de valer em produção, não só no demo. **Verificado:** o
  chart não traz configuração de nginx própria — `navyr-helm` só define a
  variável `AUTH_UPSTREAM` (`deployments.yaml:149`), e o `default.conf.template`
  da imagem é a única fonte. Corrigir o template basta, e é o que se quer: uma
  cópia da regra no chart seria uma segunda cópia para divergir.
- **R5** — Teste que falha se o bloco largo voltar. O teste poda linhas de
  comentário antes de casar.

## Critérios de aceitação

1. `GET :5173/auth/org/kms-config` com Bearer válido responde **404**, não 200.
2. `GET :5173/auth/admin/users` com Bearer válido responde **404**.
3. `GET :5173/auth/sso/oidc/callback` **continua** alcançando o `navyr-auth`.
4. `GET :5173/api/v1/auth/org/kms-config` segue atravessando o gateway e o gate
   de papel, inalterado.
5. Prova negativa: com o `location /auth/` largo de volta, o teste de R5 falha.
