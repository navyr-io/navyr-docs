# SPEC-013 — Rotas que faltam no auth para telas que já existem

**Estado:** implementada e verificada — 28/08
**Cards:** navyr-io/navyr-auth#20, navyr-io/navyr-frontend#20

## Problema

Três telas oferecem funcionalidade que o backend nunca teve. A SPEC-009 as
desabilitou com aviso honesto, em vez de deixá-las falhando com erro genérico.

| Tela | Chama | Estado |
|---|---|---|
| `EmailTab` | `/auth/org/smtp-config` | rota não registrada |
| `GroupsTab` (busca) | `/auth/ldap/test-query` | rota e serviço não existem |
| `OrgTab` (nome) | — | nem rota nem serviço |

## O que a verificação mostrou

**SMTP está pronto e só falta a rota.** `internal/service/auth_segredos.go` tem os
três métodos, e o envio de teste é real — `smtp.SendMail` através de
`newSMTPEmailSender`, com a senha decifrada da organização.

```
GetOrgSMTPConfig     (linha 239)
UpdateOrgSMTPConfig  (linha 269)
TestOrgSMTPConfig    (linha 291)
```

Sob `/auth/org/` o `rotas.go` só registra `kms-config` e `kms-config/test`.

**LDAP tem a capacidade de busca.** `auth_ldap.go` usa `ldap.NewSearchRequest` em
quatro lugares. Um serviço de consulta de teste reusaria o mesmo caminho.

**O rename de organização não tem nada.** `UpdateOrganizationSettings` existe, mas
só escreve `settings` — não há método que altere o nome.

## Um achado do levantamento

`TestOrgSMTPConfig` **não checava papel**, e ficou de fora do `navyr-auth#19`.

O teste daquele card enumerava verbos — `^(Get|Update)Org(KMS|SMTP)Config$` — e o
quinto método não casava. Testar dispara envio real com a credencial de SMTP da
organização, então sem gate qualquer membro consumiria a cota do servidor alheio e
confirmaria que o host existe.

Corrigido junto, e o regex do teste passou a ser `^[A-Z]\w*Org(KMS|SMTP)Config$`.
Enumerar verbos é como se perde o próximo.

## Decisões tomadas

**D1 — A consulta LDAP de teste é implementada.** *(28/08)*
Serve para validar um filtro antes de salvar a configuração. Sem ela, o operador
descobre que o filtro está errado quando um usuário não consegue entrar.

**D2 — Renomear altera só o nome de exibição.** *(28/08)*
Nada referencia a organização por nome — as chaves são UUID. Renomear é troca de
rótulo.

## Regras

**R1 — As três rotas de SMTP são registradas e expostas com gate.**
`GET`, `PUT` e `POST /test`. Entram em `rotasAdministrativasDeAuth` no gateway
(SPEC-009 R3): configuração de SMTP da organização é ação administrativa, e o
teste dispara envio real com a credencial dela.

**R2 — A consulta LDAP de teste valida o filtro antes de ir à rede.**

*Esta regra foi corrigida durante a implementação.* A versão original mandava
escapar o parâmetro com `ldap.EscapeFilter`. **Estava errada**: o campo da tela
pede um filtro — o placeholder é `(&(objectClass=groupOfNames)(cn=devops-team))` —
e escapar destruiria a sintaxe que ele existe para aceitar.

O risco também não é escalada: quem chega nesta rota já é admin e já pode
reescrever o filtro da organização inteira, que roda contra o mesmo diretório com a
mesma credencial de bind.

O que protege é compilar o filtro antes de enviá-lo, para que sintaxe inválida vire
422 com o erro de compilação em vez de erro obscuro do servidor, e o limite de R3.

**R3 — A consulta de teste tem limite de resultados.**
Uma consulta ampla num diretório corporativo devolve dezenas de milhares de
entradas. O botão existe para validar um filtro, não para exportar o diretório.

**R4 — Renomear exige papel administrativo e entra na auditoria.**
Mesmo sendo troca de rótulo, é mudança na organização inteira e aparece em
convites e relatórios.

**R5 — O nome tem validação de tamanho e conteúdo.**
Vazio, só espaço, ou 500 caracteres não são nomes. A tela já tem o campo; o
backend não pode confiar nele.

**R6 — As telas voltam a funcionar, e os avisos da SPEC-009 saem.**

## Critérios de aceitação

1. `GET/PUT /api/v1/auth/org/smtp-config` respondem 200 para admin e 403 para membro.
2. `POST /api/v1/auth/org/smtp-config/test` responde com sucesso ou o erro real do
   servidor SMTP.
3. A consulta LDAP com filtro contendo `*)(uid=*` não devolve o diretório inteiro.
4. `PATCH` do nome da organização altera e persiste; recarregar mostra o novo nome.
5. Nome vazio ou com 500 caracteres é recusado com 400.
6. Membro comum recebe 403 ao tentar renomear.
7. Nenhuma das três telas exibe mais o aviso de indisponibilidade.

## Achados durante a implementação

**O serviço de SMTP estava pronto — inclusive o handler.** `OrgSMTPConfig` e
`OrgSMTPConfigTest` existiam em `org_settings_handler.go`. Faltavam **duas linhas**
de registro no `rotas.go`, e por isso o `EmailTab` recebeu 404 por tempo
indeterminado.

**`TestOrgSMTPConfig` não checava papel**, e ficou de fora do `navyr-auth#19`
porque o teste daquele card enumerava verbos: `^(Get|Update)Org(KMS|SMTP)Config$`
não casa `Test`. Testar dispara envio real com a credencial de SMTP da organização.
Corrigido, e o regex passou a ser `^[A-Z]\w*Org(KMS|SMTP)Config$` — enumerar verbos
é como se perde o próximo.

**O cliente do gateway tem timeout de 1500ms**, e o teste de SMTP espera até 10s.
Toda tentativa devolvia `auth unavailable`: o operador via erro da plataforma
quando o que falhou era a configuração dele. As rotas que abrem conexão com
servidor de terceiro — SMTP e LDAP do cliente — passaram a ser marcadas como lentas
na tabela e usam cliente com 30s.

**`context deadline exceeded` não diz nada** a quem configura SMTP. A mensagem
passou a nomear host, porta e a causa provável.

## Fora de escopo

- Slug único por organização. Nada hoje o consome.
