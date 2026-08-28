# SPEC-010 — A URL do túnel embutida no manifesto do agente

**Estado:** implementada e verificada — 27/08
**Card:** navyr-io/navyr-orchestrator#10

## Problema

O manifesto que o cliente aplica com `kubectl apply` carrega a URL que o agente
usa para discar de volta. Ela é derivada da requisição, e a derivação erra em
exatamente a topologia de produção.

`internal/handler/agent_manifest_handler.go:121-134`:

```go
func deriveOrchestratorURL(r *http.Request) string {
	host := r.Host
	if host == "" { host = "localhost" }
	if idx := strings.LastIndex(host, ":"); idx > 0 { host = host[:idx] }
	scheme := "ws"
	if r.TLS != nil { scheme = "wss" }
	return fmt.Sprintf("%s://%s:8083", scheme, host)
}
```

**Dois defeitos independentes.**

**1. A porta 8083 é fixa.** Atrás de Ingress o cliente alcança 443. A 8083 é a
porta interna do orquestrador, que num deploy Kubernetes não é exposta — a
NetworkPolicy do chart inclusive a fecha.

**2. `r.TLS` é sempre `nil`.** A requisição chega ao orquestrador **através do
gateway**, em HTTP interno. `r.TLS` nunca é preenchido, mesmo com o navegador do
cliente em HTTPS.

Combinados: um cliente em `https://navyr.empresa.com` recebe um manifesto que manda
o agente discar `ws://navyr.empresa.com:8083`.

## Por que é silencioso

A UI diz "Cluster registrado" e mostra o comando. O `kubectl apply` funciona — os 6
recursos são criados. O agente entra em CrashLoopBackOff dentro do cluster **do
cliente**, onde ninguém da Navyr olha. Do lado da plataforma o cluster fica
`pending` para sempre, sem erro em lugar nenhum.

É o mesmo formato do `navyr-orchestrator#12`, que ficou 1429 reinícios sem ninguém
ver.

## O que existe hoje na cadeia

Medido nos três componentes:

| Componente | O que faz |
|---|---|
| nginx do frontend | envia `Host`, `X-Real-IP`, `X-Forwarded-For`. **Não** envia `X-Forwarded-Proto` nem `X-Forwarded-Host`. |
| gateway | tem `TRUST_PROXY_HEADERS` (SEC-12), default **não confiar**. Repassa identidade ao orquestrador; não repassa `X-Forwarded-*`. |
| orquestrador | não lê `X-Forwarded-*` — verificado por grep no arquivo. |

Ou seja: a informação de esquema e host público **não existe** na cadeia. Nenhum
componente a produz, então não é questão de o orquestrador ler o cabeçalho errado.

## O contorno que já existe

A configuração `orchestrator_url` por organização (Settings → Governance) tem
precedência sobre a derivação, e o frontend a passa como query param. Ela voltou a
funcionar na SPEC-003 — antes, a guarda `if (!!orgId)` impedia o frontend de ler as
settings, e a configuração era ignorada.

Quem configurou nunca passa pelo caminho quebrado. A derivação é o **default**, e
vale para toda organização que não configurou — que é o estado de toda conta nova.

## Decisões tomadas

**D1 — Variável de ambiente no orquestrador.** *(27/08)*

`NAVYR_PUBLIC_WS_URL`. Quem instala a plataforma sabe o endereço público dela — é
a mesma pessoa que configura o Ingress, e o chart pode preenchê-la a partir do host
já declarado ali.

A alternativa dos cabeçalhos `X-Forwarded-*` foi recusada por dois motivos: toca
três componentes, e faria a URL que vai no manifesto do cliente depender de um
cabeçalho forjável — com a confiança mal configurada, um cliente influenciaria o
que vai no manifesto de outro. Para um valor que decide para onde o agente disca,
o custo do erro é alto demais.

**D2 — Sem fonte confiável, o endpoint recusa e explica.** *(27/08)*

Entregar um manifesto que quebra dentro do cluster do cliente é o pior resultado
possível: o `kubectl apply` funciona, os 6 recursos sobem, e a falha aparece onde
ninguém da Navyr olha. Falhar aqui é alto e cedo.

`localhost` e `127.0.0.1` continuam derivando, como caso explícito de
desenvolvimento — é onde a derivação está certa, e é o único lugar onde ela está.

## Precedência

1. `orchestrator_url` da organização, passado como query param. Já funciona.
2. `NAVYR_PUBLIC_WS_URL` do orquestrador.
3. Derivação da requisição — **somente** se o host for `localhost` ou `127.0.0.1`.
4. Recusa, nomeando a configuração que falta.

## Regras

**R1 — `NAVYR_PUBLIC_WS_URL` é lida e validada.**
Precisa ser URL absoluta com esquema `ws` ou `wss`. Valor malformado é erro de
configuração e deve falhar na leitura, não produzir manifesto silenciosamente
errado.

**R2 — A porta deixa de ser fixa no código.**
Ela vem da URL configurada. Atrás de Ingress o cliente alcança 443, e `:8083` é a
porta interna que o chart nem expõe.

**R3 — Fora de localhost e sem configuração, o endpoint responde erro.**
A mensagem nomeia a variável e a configuração por organização, porque as duas
resolvem e quem lê o erro precisa saber que tem escolha.

**R4 — `r.TLS` deixa de ser consultado.**
Ele é sempre `nil` atrás do gateway. Um teste que dependesse dele passaria em
unidade e falharia em produção — que é como o defeito chegou aqui.

**R5 — O compose de demonstração define a variável.**
Senão a stack local passa a recusar, e a jornada de onboarding quebra.

## Critérios de aceitação

1. Com `NAVYR_PUBLIC_WS_URL=wss://navyr.exemplo.com`, o manifesto sai com essa URL
   exata — sem `:8083` acrescentado.
2. Sem a variável e com host `localhost`, o manifesto sai como hoje.
3. Sem a variável e com host que não seja localhost, o endpoint responde erro
   nomeando a configuração.
4. `orchestrator_url` da organização continua tendo precedência sobre as duas.
5. A jornada de onboarding na stack local segue funcionando ponta a ponta.
6. Nenhum caminho consulta `r.TLS`.

## Fora de escopo

- O `platform_url`, que resolve o mesmo problema para a URL do `curl` e já
  funciona. Esta spec trata da URL do WebSocket.
