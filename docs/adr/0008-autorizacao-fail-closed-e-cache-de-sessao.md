# 0008 — Autorização fail-closed e cache de sessão

**Status:** Aceita · **Data:** 2026-08

> **Ampliada pelo [ADR 0009](0009-sessao-de-navegador-sem-token-no-cliente.md).**
> As decisões aqui seguem válidas, mas o cache de sessão deve nascer como
> sessão autoritativa de BFF, não como cache de validação — implementar as duas
> em sequência significaria refazer.

## Contexto

A trava de permissões granulares do gateway libera quando não sabe responder.
São dois trechos, em serviços diferentes, que sozinhos parecem razoáveis e
juntos abrem a porta.

No `navyr-auth`, ao validar a sessão:

```go
permissions, permErr := h.authService.AdminGetUserEffectivePermissions(...)
if permErr != nil {
    permissions = nil        // erro descartado, sem log e sem métrica
}
```

No `navyr-gateway`, ao decidir se a requisição passa:

```go
if required == "" || len(permissions) == 0 {
    return nil               // lista vazia libera
}
```

Somando os dois, **uma falha na consulta de permissões vira acesso liberado**.
Não é preciso ninguém explorar nada: basta o banco engasgar no momento errado
para um usuário restrito passar em rota que deveria receber `403`. E como o erro
é descartado sem registro, isso é hoje **invisível** — não aparece em log, não
incrementa contador, não gera alerta.

O agravante vem do modelo de edições. Na edição livre,
`AdminGetUserEffectivePermissions` devolve `ErrRecursoEnterprise` sempre; o
`auth` achata esse erro em lista vazia exatamente como achataria um erro de
banco. O gateway recebe o mesmo valor para três situações distintas:

| Situação | O que chega no gateway | Decisão correta |
|---|---|---|
| Edição livre, sem IAM granular | lista vazia | liberar — o RBAC por papel já decidiu |
| Usuário sem permissões atribuídas | lista vazia | **negar** — foi decisão explícita de um administrador |
| A consulta falhou | lista vazia | **negar e registrar** |

O problema real não é qual lado da trava está ligado. É que **a informação
necessária para decidir foi descartada antes de chegar na trava.**

Contexto operacional relevante: o gateway chama `validateSession` no `auth` em
**toda** requisição, e não existe cache de sessão. Redis já está no gateway,
usado por rate limit e pub/sub. Não há tracing distribuído — só o orchestrator
tem OpenTelemetry.

## Decisão

**1. O `auth` para de achatar os três casos.** A resposta de validação passa a
distinguir "não se aplica", "sem permissões atribuídas" e "não consegui
verificar". O erro deixa de ser descartado.

**2. O gateway passa a ser fail-closed**, com a decisão por caso da tabela
acima. Negar na dúvida é o princípio padrão, e a razão é assimetria de
consequência: negar errado produz um `403`, uma reclamação e um diagnóstico em
minutos; liberar errado não produz sintoma nenhum — o sistema funciona
perfeitamente enquanto vaza.

Note que isto é **mais** restritivo que inverter a condição: hoje o usuário que
um administrador deliberadamente deixou sem permissão também passa em tudo.

**3. A telemetria separa segurança de disponibilidade.** Dois rótulos distintos
no `errorCounter`, seguindo o padrão que já existe no gateway:

| Rótulo | Significa | Natureza |
|---|---|---|
| `permission_denied` | usuário sem a permissão exigida | evento de segurança, esperado, *bom* que suba sob ataque |
| `permission_lookup_failed` | não foi possível verificar | **incidente de disponibilidade**, alarme com limiar baixo |

Sem essa separação não há como distinguir "funcionando como projetado" de
"sistema quebrado negando todo mundo".

**4. Cache de sessão no Redis, com invalidação por evento.** A validação passa a
ser guardada com **TTL de 30 segundos**, e quando uma permissão muda o `auth`
publica evento e o gateway invalida a entrada.

O precedente de mercado mais próximo é o authorizer por webhook do Kubernetes,
que resolve o mesmo problema com TTL assimétrico —
`authorization-webhook-cache-authorized-ttl` em 5 min e
`unauthorized-ttl` em 30 s. Introspecção de token OAuth costuma ficar entre 30 s
e 5 min.

Escolhemos 30 s simétrico por aritmética: um usuário a 20 requisições por minuto
gera 2 chamadas ao `auth` em vez de 20 — corte de **90%**. Subir para 5 min leva
a ~99%, e **os 9% adicionais custam uma janela de revogação 10 vezes maior**.
Com a invalidação por evento, o TTL não é o mecanismo principal de revogação: é
a rede de segurança para quando o evento se perde, e rede de segurança curta é
melhor.

Três regras acompanham:

- **Nunca servir valor expirado.** `auth` fora com cache vencido **nega**. É o
  custo do fail-closed, aceito explicitamente.
- **Nunca cachear falha.** Cache negativo de "não consegui verificar"
  transformaria falha momentânea em minutos de indisponibilidade.
- **Redis é cache, não autoridade.** Redis fora faz o gateway voltar a chamar o
  `auth` direto — que é exatamente o comportamento de hoje. Redis fora é a
  latência atual de volta, **não** indisponibilidade, e nunca nega ninguém.

O risco de o Redis cair não é negar, é **efeito manada** sobre o `auth`. Três
mitigações, todas automáticas: cache local em memória como primeiro nível,
`singleflight` para colapsar consultas idênticas concorrentes, e circuit breaker
no Redis para não pagar timeout em toda requisição enquanto ele está fora.

O evento **não decide nada** — apenas apressa a expiração. Essa distinção é a
razão de o desenho ser assim: autorização é síncrona, no caminho da requisição,
e não pode esperar fila. Event-driven aplicado à *decisão* traria consistência
eventual, e em permissões isso significa **permissão revogada continuar valendo
por uma janela** — regressão de segurança, o oposto do que este ADR conserta.
Aplicado à *invalidação*, dá cache com revogação quase imediata.

## Alternativas descartadas

**Inverter a condição no gateway e mais nada.** Lista vazia passa a negar. É a
correção de uma linha, e quebra a edição livre inteira: no OSS a lista é sempre
vazia, então todo usuário receberia `403` em qualquer rota com permissão
exigida. Segura e inutilizável.

**Permissões nas claims do JWT, sem consulta.** Elimina a falha do caminho da
requisição e é a opção mais rápida. Descartada porque revogação passa a depender
do ciclo do token: tirar permissão de alguém não teria efeito até o refresh. Em
IAM, atraso de revogação é o defeito mais caro.

**Autorização por eventos.** Descrita acima. Descartada para a decisão,
adotada para a invalidação.

## Critério de aceitação — experimentos de caos

A mudança não é considerada entregue por passar em teste unitário. Fail-closed e
degradação só se provam derrubando as dependências de verdade, porque o modo de
falha *é* o comportamento sob teste.

| Experimento | Asserção |
|---|---|
| Matar o **Redis** | Nenhum `403` novo. Latência sobe. Cliente não vê erro. |
| Matar o **auth** | Nega. `permission_lookup_failed` sobe. Alerta dispara. |
| Injetar latência no **auth** | Timeout previsível, sem cascata para os outros serviços |
| Matar o **Redis sob carga** | Sem manada: chamadas ao `auth` colapsadas pelo `singleflight` |
| Expirar o cache com **auth fora** | Nega — confirma que valor expirado nunca é servido |

O segundo é o mais valioso: é o único jeito de provar que o fail-closed funciona
como projetado, e não como esperança. O último existe porque "nunca servir
expirado" é fácil de escrever e fácil de violar sem perceber.

## Consequências

**O que ganhamos.** A trava passa a negar quando não sabe. O caso do usuário
deliberadamente sem permissões passa a ser respeitado. A falha deixa de ser
invisível. E o cache reduz a latência que hoje inclui uma chamada HTTP síncrona
ao `auth` em toda requisição — o mesmo caminho que a meta de 300ms atravessa.

**O que aceitamos, e é o ponto que exige projeto.** Fail-closed transforma uma
falha do `auth` em **indisponibilidade da plataforma inteira**. É o preço
correto — melhor parar do que liberar errado — mas muda a natureza do cache: ele
deixa de ser otimização e vira **parte da disponibilidade**. Três decisões
passam a ser de arquitetura, não de ajuste:

- qual TTL equilibra revogação e resiliência;
- o que acontece quando o **Redis** cai, já que ele passa a estar no caminho;
- se um valor expirado pode ser servido enquanto o `auth` se recupera — e por
  quanto tempo. Servir valor velho é, literalmente, escolher disponibilidade em
  cima de segurança, e precisa ser escolha consciente com limite explícito.

**Nada muda para a edição livre.** O primeiro caso continua liberando.

**Pré-requisito de execução.** `enforceGranularPermissions` está hoje em **0% de
cobertura**, `rateLimitMiddleware` em 0%, e o despacho de 132 rotas não tem
referência em teste nenhum. Mexer na autorização antes de cobri-la é refatorar
sem rede o único ponto público do produto. **A ordem é: teste primeiro, depois
esta mudança.**

**Achado adjacente, decisão própria.** O rate limit já é fail-open quando o
Redis falha, com comentário no código dizendo isso — e some **silenciosamente**,
sem log nem métrica. Fail-open ali é defensável: é proteção contra abuso, não
autorização. Ser silencioso não é. O `memoryLimiter` já existe no código e hoje
só é usado quando a `REDIS_URL` é inválida na partida; deveria ser o fallback de
runtime, em vez de "libera tudo".
