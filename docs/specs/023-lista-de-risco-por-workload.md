# SPEC-023 — A lista de risco por workload dá 40 a todos

**Card:** `navyr-io/navyr-frontend#30` · **Severidade:** Médio
**Descoberto:** ao verificar a correção da SPEC-019 (28/08/2026)

## O defeito

O painel **WORKLOAD RISK SCORES** existe para priorizar. Ele não distinguia o
workload mais perigoso do menos perigoso.

Medido ao vivo, no mesmo instante em que o backend reportava:

| Workload | falhas | `risk_score` do backend | a lista mostrava |
|---|---|---|---|
| `legado-privilegiado` (`privileged: true`) | 6 | **60** | 40 · Critical |
| `sem-limites` (`hostNetwork`) | 6 | **60** | 40 · Critical |
| `kube-proxy` | 1 | **8** | 40 · Critical |
| `kindnet` | 1 | **8** | 40 · Critical |
| `coredns` | 1 | **8** | 40 · Critical |

Seis linhas, seis vezes `40`, seis vezes `Critical`, barras do mesmo tamanho.

## Causa

`SecurityIntelligencePage.tsx:167` calculava o próprio score em vez de usar o do
backend:

```ts
const sevWeight = sev === "critical" ? 40 : sev === "high" ? 20 : sev === "medium" ? 10 : 5;
const newScore = Math.min(100, prev.score + sevWeight);
```

Soma de pesos de severidade dos achados daquele workload. Como quase todo
workload aqui tem exatamente **um** achado `critical`, todos saem em 40.

O `risk_score` que o backend devolve por achado existe no tipo
(`SecurityIntelligencePage.tsx:15`) e era descartado.

## Por que só agora

Antes da SPEC-019, `risk_score` media **saúde** — o privilegiado valia 40 e o
`coredns` valia 92. Usá-lo teria sido pior que ignorá-lo. Com a escala
corrigida, ele passa a ser a fonte certa: distingue 6 falhas de 1 falha, coisa
que a soma de pesos de severidade não faz.

Os dois números coincidiam em 40 por acidente aritmético, e um escondia o outro.

## Decisões

**D1 — Usar o número do backend quando ele existir.**

**D2 — Máximo, e não soma.**
O `risk_score` do backend já agrega as checagens daquele workload. Somar
achados leves até parecerem um grave é justamente o erro que esta lista cometia.

**D3 — Manter a soma de pesos como alternativa, e não removê-la.**
A lista serve **duas abas**. `configRiskFinding` traz `risk_score`;
`rbacRiskFinding` **não traz score nenhum** — verificado em
`security_scan.go:101`. Na ausência dele a soma de pesos continua sendo o melhor
disponível, e é o comportamento que a aba RBAC já tinha, que este card não mediu
e não deve mudar de lado.

**D4 — A severidade do selo continua vindo do achado**, e não do score. São
coisas diferentes: uma é a gravidade da pior checagem, a outra é a nota
agregada.

## Regras

- **R1** — Quando o achado traz `risk_score`, a nota do workload é o **máximo**
  dos `risk_score` dos seus achados.
- **R2** — Sem `risk_score`, mantém-se a soma de pesos por severidade.
- **R3** — A barra tem comprimento proporcional à nota exibida.
- **R4** — A ordenação usa a mesma nota, de modo que a lista priorize de fato.

## Critérios de aceitação

1. `legado-privilegiado` pontua **60**; `kube-proxy` pontua **8**.
2. A barra do primeiro é visivelmente mais longa.
3. O primeiro aparece antes do segundo na lista.
4. Prova negativa: voltando a somar peso por severidade, os testes falham com
   `expected '40' to be '60'` — o número exato do card.
