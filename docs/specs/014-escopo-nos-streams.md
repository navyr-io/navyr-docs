# SPEC-014 — Escopo de namespaces nos streams

**Estado:** implementada e verificada — 28/08
**Card:** navyr-io/navyr-orchestrator#16

## Problema

`WatchEvents` e `WatchPods` eram as duas últimas funções de escopo trocando
`namespace` vazio por `"default"`. Num cluster restrito, o feed ao vivo mostrava um
namespace enquanto todas as outras telas mostravam o escopo permitido — duas partes
da interface discordando sobre o mesmo cluster.

Ficaram de fora da SPEC-011 porque devolvem canal, não coleção: `listarNoEscopo`
não se aplica.

## O que foi feito

`observarNoEscopo` abre um watch por namespace do escopo e une os eventos num canal
só. Sem restrição, abre um watch só com `""` — o `client-go` percorre o cluster, e
multiplexar seria custo sem ganho.

O encerramento é o que exige cuidado, e é o que os testes cobrem:

- todo produtor sai pelo fechamento da origem ou pelo contexto;
- o canal de saída é fechado por uma goroutine que espera todos, senão o consumidor
  ficaria bloqueado sem distinguir "acabou" de "ainda vem";
- o envio é dentro de um `select` com `ctx.Done()` — sem ele, um consumidor que
  parou de ler deixa os produtores pendurados.

Prova negativa do último ponto: removido o `select` de envio, o teste acusa
`goroutines antes=2 depois=8`.

### Um defeito encontrado no caminho

Os handlers definiam os cabeçalhos SSE e entravam no laço sem escrevê-los. Em Go,
`Header().Set` não emite nada — os cabeçalhos só saem no primeiro `Write`. Com um
cluster estável, que é o caso normal, o cliente ficava esperando indefinidamente e
o stream parecia morto. Agora há `WriteHeader` + `Flush` antes do laço.

## O que esta spec NÃO resolve

**O canal alimentado por ela não recebe nada**, por um defeito independente:
`navyr-io/navyr-agent#16`.

O agente faz `io.ReadAll(resp.Body)` antes de montar o envelope de resposta. Um
watch devolve corpo que nunca termina, então a leitura bloqueia para sempre e nada
volta pelo túnel. Não é bug de borda — o `Envelope` tem um campo `Body` com o
conteúdo completo, e não existe representação para resposta contínua.

Medido: com evento provocado, 0 bytes em 22s, tanto com namespace explícito quanto
cluster-wide. Namespace explícito não passa pelo multiplexador, o que isola a causa.

As duas camadas são independentes. Corrigir o túnel faz esta entrar em operação sem
mudança.

## Critérios de aceitação

1. Sem restrição, abre um watch só com `""`. ✅ teste
2. Com allowlist, abre um por namespace permitido e une. ✅ teste
3. Cancelar o contexto não deixa goroutine pendurada. ✅ teste com contagem
4. O canal de saída fecha quando as origens acabam. ✅ teste
5. Namespace explícito não multiplexa. ✅ teste
6. Os cabeçalhos SSE saem antes do primeiro evento. ✅ verificado na stack
7. Nenhuma função de escopo troca vazio por `default`. ✅ restam só as 2 escritas

Os testes passam com `-race`.
