# 0009 — Sessão de navegador sem token no cliente

**Status:** Proposta · **Data:** 2026-08

Reconcilia e amplia o [ADR 0008](0008-autorizacao-fail-closed-e-cache-de-sessao.md).

## Contexto

A SPA guarda as credenciais em `localStorage`:

```js
localStorage.getItem("navyr.token")          // acesso
localStorage.getItem("navyr.refresh_token")  // e renovação
```

`localStorage` é legível por **qualquer JavaScript que rode na página**. Não há
permissão, escopo ou proteção — é gaveta aberta para todo script, inclusive de
dependência transitiva comprometida.

O efeito não é "atacante age enquanto o usuário está na página". É que o token é
*bearer* — quem o segura **é** o usuário, sem vínculo com dispositivo, IP ou
sessão — então ele é **levado embora** e usado de outra máquina, depois. E como o
refresh token está junto e vale 30 dias, o atacante **se renova sozinho**: a
janela deixa de ser a validade do JWT e passa a ser a paciência dele. A
exfiltração sai como uma requisição qualquer do navegador; o servidor não tem
sinal nenhum de que aconteceu.

**O que isso destranca aqui não é um CRUD.** O Navyr faz `exec` em pod, lê
secret, apaga workload e aplica RoleBinding **em cluster de produção do
cliente**. O mesmo roubo que num SaaS comum resulta em "leram meus dados" aqui
resulta em acesso operacional à infraestrutura de terceiro. É o pior custo
possível de roubo de credencial, e é o fato que ordena as opções abaixo.

Dois fatos do estado atual pesam na decisão:

- O **gateway já tem formato de BFF**: é o único ponto público, a SPA fala só com
  ele, e ele já valida a sessão chamando o `auth` **em toda requisição**, já
  injetando contexto interno assinado nos serviços de trás.
- O **ADR 0008 já compromete o gateway com estado em Redis**. A decisão de
  torná-lo stateful, e o Redis parte da disponibilidade, já foi tomada.

## Decisão

**O navegador deixa de segurar credencial. Destino: BFF no gateway.**

A SPA recebe um **cookie de sessão opaco** (`HttpOnly`, `Secure`, `SameSite`); o
gateway guarda o registro de sessão no Redis e é ele quem porta os tokens ao
falar com os serviços internos. Nenhum JWT chega ao JavaScript.

### Por que BFF e não cookie com JWT dentro

Cookie `HttpOnly` com JWT resolve a exfiltração, e já seria ganho grande: converte
"credencial roubada e portátil" em "sequestro temporário dentro da página" — com
XSS o atacante ainda faz requisições, porque o navegador anexa o cookie sozinho,
mas não leva nada para casa. **Quem promete mais que isso está vendendo.**

O que só o BFF acrescenta, e que decide o caso:

- **Revogar é apagar estado no servidor** — imediato, sem janela. JWT é válido por
  si mesmo e não há como desfazê-lo antes do vencimento.
- **O navegador nunca vê credencial que funcione fora dele.** Numa venda
  Enterprise, é a única das opções que responde "o navegador não guarda token" no
  questionário de segurança.

### O que muda no ADR 0008

O 0008 continua válido no essencial, e **um dos seus problemas some**:

| Decisão do 0008 | Sob BFF |
|---|---|
| `auth` distingue os três casos de permissão | **Mantida**, sem alteração |
| Gateway nega na dúvida | **Mantida**, sem alteração |
| Métrica separa segurança de disponibilidade | **Mantida**, sem alteração |
| TTL de 30 s + invalidação por evento | **Simplifica** — ver abaixo |

Boa parte da complexidade do 0008 — calibrar TTL, invalidação por evento para
encurtar o atraso de revogação — existe porque a decisão de autorização é
derivada de um JWT autovalidável. Com a sessão sendo a fonte da verdade no
servidor, revogação deixa de ser corrida contra relógio: apagar o registro
encerra a sessão. O TTL muda de natureza, de "quão velha pode ser uma decisão de
autorização" para "quanto tempo uma sessão ociosa sobrevive".

**Consequência de sequenciamento:** o 0008 não deve ser implementado como cache
de validação e depois refeito. Os dois são o mesmo desenho, e o 0008 é a primeira
metade dele.

### Onde a sessão nasce

O endpoint `POST /auth/sso/handoff`, criado em 21/08 para tirar o token da query
string, é o lugar natural: hoje ele devolve `{user, token, refresh_token}` em
JSON, e passa a estabelecer a sessão. É mudança no que ele responde, não fluxo
novo.

Ajuda que a SPA **nunca implementou o pouso do login federado** — não há
consumidor a migrar, então o fluxo pode nascer já no formato certo.

## Alternativas descartadas

**Manter `localStorage` e só mitigar** (TTL curto, CSP estrita). Reduz janela, não
fecha. Descartada pelo custo de roubo descrito no contexto.

**Cookie `HttpOnly` com o JWT dentro.** Boa, e seria aceitável em outro produto.
Descartada por não resolver revogação e por ainda entregar ao navegador uma
credencial válida por si mesma.

**Token de acesso só em memória, refresh em cookie.** O passo mais barato, e foi
a primeira recomendação — revista. O custo que a justificava ("o gateway é
stateless") **já está sendo pago pelo 0008**, o que muda a comparação. Continua
sendo o caminho se o BFF for adiado, mas como etapa, não como destino.

## Consequências

**O que ganhamos.** Nenhuma credencial utilizável fora do navegador. Revogação
imediata. Uma resposta melhor em avaliação de segurança de cliente. E parte da
complexidade do 0008 deixa de ser necessária.

**O que aceitamos, e é o ponto mais duro.** O Redis passa a guardar a **sessão**,
não um cache dela. Sob o 0008, perder o Redis significava voltar a chamar o
`auth` — degradação de latência. Sob BFF, **perder o Redis desloga todo mundo.**

Isso não é falha de segurança, é de disponibilidade. **Decidido em 21/08: AOF**,
com as condições abaixo — sem elas, ligar `appendonly yes` é teatro.

### Persistência: AOF com `everysec`, em Redis próprio

**Pré-requisito que invalida a configuração atual.** O Redis roda hoje como
`Deployment`, **sem volume nenhum**, com `--save "" --appendonly no` e
`readOnlyRootFilesystem: true`. Ligar AOF assim escreveria em sistema de arquivos
somente leitura — e se escrevesse, o arquivo morreria no primeiro reagendamento
do pod. **AOF exige antes `StatefulSet` + `PersistentVolumeClaim`.**

**O custo do AOF é só em escrita.** Ler sessão — o caminho de toda requisição sob
BFF — não paga nada. Escrever sessão acontece no login. A conta seria trivial se
não fosse por um detalhe: existe **um Redis só**, compartilhado por gateway
(rate limit e pub/sub), collector e orchestrator — e o rate limit faz `INCR` **em
toda requisição**. Com AOF no Redis compartilhado, cada requisição vira um
append.

| `appendfsync` | Custo | Perda máxima |
|---|---|---|
| `always` | fsync por escrita: ~1–2 ms em disco de rede, ~0,1–0,3 ms em NVMe local | 1 escrita |
| `everysec` | fsync em thread de fundo, 1×/s. Sem custo constante; risco é pico quando o fsync anterior ainda não terminou | ~1 segundo |
| `no` | nenhum | ~30 s |

Contra a meta de 300 ms, nem o `always` aparece na média: 2 ms é 0,7% do
orçamento. **O problema é pico, não média** — o rewrite do AOF faz `fork`, com
pico de memória por copy-on-write, mitigável com `no-appendfsync-on-rewrite yes`
ao custo de durabilidade naquela janela.

Escolhemos **`everysec`** e **Redis de sessão separado do Redis de cache**.
Sessão precisa de durabilidade; contador de rate limit e pub/sub não precisam de
nenhuma — perder um contador é irrelevante, e mantê-los no mesmo processo faria o
`INCR` de toda requisição entrar no arquivo sem motivo. Com a separação, o AOF
cobre apenas escrita de sessão, que acontece no login, e sai do caminho quente.

`always` foi descartado: paga fsync por escrita para proteger contra perder
**uma** sessão recém-criada, cuja consequência é o usuário refazer o login. Não
compensa o custo nem a variância.

### Renovação preguiçosa de TTL

Decisão de desenho que pesa mais que o modo de fsync: **a sessão não pode ser
escrita a cada requisição.** Expiração deslizante ingênua — renovar o TTL toda
vez que a sessão é lida — transforma **toda leitura em escrita**, e aí o AOF
passa a custar no caminho de toda requisição, anulando a separação acima.

O TTL só é estendido se passou mais de um intervalo mínimo desde o último toque.
As escritas ficam proporcionais a sessões ativas por minuto, não a requisições
por segundo.

**CSRF passa a existir.** Cookie é enviado automaticamente pelo navegador, então
requisição forjada de outro site carrega a credencial. `SameSite` cobre a maior
parte; escrita ainda pede token de CSRF.

**Streaming precisa de cuidado.** `/api/v1/intelligence/stream` e
`/api/v1/events/stream` são SSE, e o exec é WebSocket. Todos passam a resolver
sessão por conexão, não por requisição. É trabalho, não bloqueio.

**O túnel do agente não é afetado.** Ele é agente→orchestrator, autenticado por
token próprio, e é registrado antes da cadeia de middleware.

## Sequência

1. Implementar o **0008 já no formato do 0009** — sessão no Redis como fonte da
   verdade, não cache de validação
2. Cookie opaco emitido no handoff e no login por senha
3. SPA passa a não guardar nada; pouso do SSO nasce já assim
4. SSE e WebSocket migrados
5. `localStorage` removido

**Pré-requisito, herdado do 0008:** os testes do gateway. Autorização e rate
limit saíram de 0% em 21/08; o despacho de 132 rotas continua sem cobertura, e é
por onde tudo isso passa.
