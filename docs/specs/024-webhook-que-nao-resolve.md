# SPEC-024 — URL de webhook que não resolve vira "auth unavailable"

**Card:** `navyr-io/navyr-gateway#22` · **Severidade:** Médio
**Descoberto:** varredura de interface ponta a ponta, Fase 4 (28/08/2026)

## Medido

| URL informada | Resultado | Tempo |
|---|---|---|
| `https://example.com/hook` (resolve) | **201 Created** | 95 ms |
| `https://exemplo.local/hook` (não resolve) | **503 "auth unavailable"** | 1514 ms |

A mesma chamada direto no `navyr-auth`, sem o gateway no meio:

```
HTTP 400 em 3383ms
{"code":"invalid_request","message":"webhook url host does not resolve"}
```

O auth produz a mensagem certa e acionável. O gateway desiste antes e a
substitui por um erro de plataforma.

## Causa

A validação anti-SSRF da Fase 1 (SEC-05) resolve o hostname do webhook antes de
aceitá-lo — **corretamente**, para impedir que a URL aponte para
`169.254.169.254` ou faixas privadas. Resolver um host inexistente leva ~3,4 s.

O cliente padrão do gateway tem `Timeout: 1500 * time.Millisecond`
(`main.go:301`). Ele desiste no meio, e `proxyJSONRequest` traduz a falha em
`503 "auth unavailable"` (`proxy.go:177`).

O usuário digita uma URL errada e a plataforma se acusa de estar fora do ar.

## O mecanismo já existia

`rotaDeAuth` tem o campo `lenta`, e `clienteParaRotaLenta` tem 30 s. O
comentário que documenta o campo descreve **exatamente este defeito**, para o
teste de SMTP:

> *"o gateway desistia antes e devolvia `auth unavailable` — o operador via um
> erro da plataforma quando o que falhou era a configuração dele."*

A SPEC-013 tratou a doença no SMTP e no LDAP. A rota de webhook nunca foi
marcada.

## Decisões

**D1 — Não relaxar a validação anti-SSRF.** Ela está certa. O problema é o
timeout do cliente engolir o diagnóstico.

**D2 — Marcar só os verbos que saem para a rede.**
`POST`, `PUT` e `PATCH` resolvem DNS. `GET` e `DELETE` são banco. Marcar a
família inteira daria 30 s à listagem, e uma consulta travada seguraria a
conexão vinte vezes mais.

Isso exigiu um campo novo, `metodos`, em `rotaDeAuth` — a estrutura só sabia
casar por caminho.

**D3 — A entrada restrita vem antes da genérica.**
O despacho pega a **primeira** entrada que casa. Se a genérica viesse antes,
venceria sempre e o `lenta` nunca se aplicaria. A ordem é parte da correção, e
por isso tem teste próprio.

**D4 — Reusar `clienteParaRotaLenta`**, e não criar uma terceira constante de
timeout. 30 s é folga sobre os 3,4 s medidos; é teto, não espera.

## Regras

- **R1** — `POST`, `PUT` e `PATCH` em `/api/v1/auth/webhooks` usam o cliente de
  rota lenta.
- **R2** — `GET` e `DELETE` mantêm o timeout curto.
- **R3** — Todo verbo de `/api/v1/auth/webhooks` continua casando com **alguma**
  entrada administrativa, ou seja, continua atrás do gate de papel. Restringir
  por verbo não pode abrir buraco de autorização.
- **R4** — A entrada restrita precede a genérica, com teste que falha se a ordem
  inverter.

## Critérios de aceitação

1. URL que não resolve devolve o `400` do auth com a mensagem original, e não
   `503`.
2. URL que resolve continua devolvendo `201`.
3. Listar webhooks continua rápido.
4. Prova negativa: com `lenta: false`, com a ordem invertida, ou marcando a
   leitura como lenta, os testes falham.
