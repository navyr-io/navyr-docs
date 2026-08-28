# SPEC-015 — Resposta contínua pelo túnel do agente

**Estado:** implementada e verificada — 28/08
**Card:** navyr-io/navyr-agent#16

## Problema

O feed ao vivo de pods e eventos nunca entregou dado nenhum em modo agente — que é
o único modo que existe desde que o direto foi removido.

Medido com evento provocado, cluster conectado e saudável:

```
GET /api/v1/pods/watch?namespace=            → 200, 0 bytes em 22s
GET /api/v1/pods/watch?namespace=kube-public → 200, 0 bytes em 18s
```

## Causa

O protocolo do túnel é requisição/resposta com corpo inteiramente bufferizado.

`navyr-agent/cmd/executor/main.go:352`:
```go
respBody, _ := io.ReadAll(resp.Body)
out := Envelope{Type: "response", ID: env.ID, Status: resp.StatusCode}
```

Um watch do `client-go` devolve corpo que **nunca termina**. `io.ReadAll` bloqueia
para sempre, e o envelope de resposta nunca é montado.

Não é bug de borda: o `Envelope` tem um campo `Body` com o conteúdo completo em
base64, e não existe representação para resposta parcial. **É impossível por
construção.**

Do lado do orquestrador, `RoundTrip` espera **um** envelope, com timeout de 30s, e
`buildHTTPResponse` monta uma resposta completa.

## Decisões técnicas

**Quem decide que a resposta é contínua é o orquestrador, não o agente.**
Ele marca a requisição com `stream: true` quando a query traz `watch=true`, que é a
convenção da API do Kubernetes e o que o `client-go` emite em `.Watch()`. O agente
obedece à marca em vez de adivinhar pelo `Content-Length`, que é heurística e erra
nos dois sentidos.

Consequência boa: um agente antigo, que ignora a marca, se comporta exatamente como
hoje — não há regressão, só ausência de melhoria até reinstalar.

**Três envelopes novos, e o antigo continua valendo.**
`response_start` com status e cabeçalhos, `response_chunk` com sequência e dados, e
`response_end`. O `response` clássico segue para todo o resto, que é o caminho já
exercitado — não vale trocar o caminho comum por um novo para resolver um caso.

**O `readLoop` nunca bloqueia.**
Ele serve todas as requisições da conexão. Um consumidor lento de stream não pode
segurá-lo, senão um watch parado derruba o cluster inteiro. O canal por stream é
limitado, e estourar o limite **encerra aquele stream com erro** em vez de descartar
evento em silêncio — perder evento sem avisar num feed de segurança é pior que
falhar.

**O timeout de 30s vale só para o `response_start`.**
Depois dele o corpo é contínuo por natureza. Aplicar timeout ao corpo mataria todo
watch aos 30 segundos.

## Regras

**R1 — `stream: true` é decidido pelo orquestrador, a partir de `watch=true`.**

**R2 — Agente marcado copia em blocos e emite `response_chunk`.**
Nunca `io.ReadAll` num corpo marcado como contínuo.

**R3 — O orquestrador devolve um `io.Pipe` como corpo.**
O leitor vai para o `http.Response`; o escritor é alimentado pelo `readLoop`.

**R4 — Encerramento propaga nos dois sentidos.**
Cliente desconecta → contexto cancela → o agente para de copiar. Agente cai →
`c.done` fecha → o pipe fecha com erro. Nenhum dos dois deixa goroutine pendurada.

**R5 — Buffer limitado com falha explícita.**
Estouro fecha o stream com erro nomeando a causa.

**R6 — O envelope `response` clássico continua funcionando.**
Compatibilidade com agente em campo, e é o caminho de todo o resto.

## Critérios de aceitação

1. `GET /api/v1/pods/watch?namespace=kube-public` entrega evento em segundos ao
   criar um pod lá.
2. O stream continua aberto além de 30 segundos.
3. Cliente desconectando encerra a cópia no agente, sem goroutine pendurada.
4. Requisição comum continua usando o envelope `response`.
5. Agente que não entende `stream` se comporta como hoje.
6. Testes passam com `-race`.

## Verificado na stack

| | antes | depois |
|---|---|---|
| watch em namespace explícito | 0 bytes em 18s | **1162 bytes**, pod no stream |
| watch cluster-wide | 0 bytes em 22s | **3519 bytes**, 4 namespaces |
| evento após 40s | — | entregue |
| requisição comum | 10 pods | 10 pods, sem mudança |

O stream cluster-wide entregando 4 namespaces é a SPEC-014 entrando em operação: o
multiplexador estava certo desde então e não tinha o que multiplexar.

Cinco streams abertos e fechados em sequência: nenhum panic, nenhum vazamento no
log do agente, túnel seguiu respondendo 200.
