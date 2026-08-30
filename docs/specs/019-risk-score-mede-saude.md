# SPEC-019 — O `risk_score` mede saúde e é exibido como risco

**Card:** `navyr-io/navyr-orchestrator#17` · **Severidade:** Alto
**Descoberto:** varredura de interface ponta a ponta, Fase 3 (28/08/2026)

## O defeito

`configRiskScore` começa em 100 e **subtrai** por checagem falha. É um score de
**saúde**: 100 significa nada errado.

`internal/handler/security_scan.go:1157`:

```go
func configRiskScore(checks []configRiskCheck) (int, map[string]int) {
	score := 100
	for _, c := range checks {
		switch strings.ToLower(c.Severity) {
		case "danger", "critical", "high":
			score -= 20
		default:
			score -= 8
		}
	}
```

Mas o campo se chama `risk_score`, o tipo é `configRiskFinding`, e a aba se
chama **Config Risk**.

### Medido, com cargas plantadas de propósito

`GET /api/v1/clusters/{id}/security/config-risk`, 40 achados:

| Workload | Como foi plantado | `risk_score` | Checagens falhas |
|---|---|---|---|
| `homologacao/legado-privilegiado` | `privileged: true` | **40** | 6 |
| `homologacao/sem-limites` | `hostNetwork`, sem probes | **40** | 6 |
| `homologacao/worker-root` | `runAsUser: 0`, `SYS_ADMIN` | **60** | 5 |
| `kube-system/coredns` | — | **92** | 1 |
| `kyverno/kyverno-admission-controller` | — | **92** | 1 |

**Os mais perigosos pontuam menos. Os mais seguros pontuam mais.** Ordenar ou
priorizar por esse número inverte a fila de trabalho.

## Correção de uma afirmação do card

O card dizia que o número aparece na tela como `legado-privilegiado | 40 |
Critical`, atribuindo o `40` da lista **WORKLOAD RISK SCORES** a este campo.
**São dois números diferentes que coincidem.**

Aquela lista não consome `risk_score`: o frontend calcula o próprio score em
`SecurityIntelligencePage.tsx:167`, acumulando peso por severidade
(`critical` = 40, `high` = 20, `medium` = 10), com teto em 100 — e ali **maior é
pior**, que é o certo. Um workload com um achado crítico dá 40 pelos dois
caminhos, por coincidência aritmética.

O `risk_score` por achado **não é renderizado em lugar nenhum** hoje.

### Mas o defeito é visível, por outro caminho

O que a tela mostra é o **score agregado** — `configRiskResponse.Score`, que
vem de `computeConfigRiskOverallScore` e é a média dos `risk_score`. Ele
alimenta o anel do topo da aba, rotulado **"Config Risk score"**, com
`riskLabel` ao lado.

Medido: `score: 73`, exibido como `73 / 100` num anel **vermelho** rotulado
**"High Risk"**. Quem lê entende "73 de risco, alto". Significa o contrário:
73 de saúde, ou seja, problemas moderados.

O rótulo "High Risk" e a cor vêm da **contagem** de achados críticos, não do
score, então os dois se reforçam na direção errada.

## Decisão

**Fazer o número medir risco**, e não renomear o campo para `health_score`.

O produto já tem `risk_score` em dois outros lugares, e nos dois **maior é
pior**:

- `aiops_service.go:578` — `critical*20 + high*10 + (100-compliance)*0.3`, com
  teto em 100;
- a aba **Attack Path**, onde `SecurityIntelligencePage.tsx:661` trata
  `risk_score >= 80` como crítico.

Dois campos chamados `risk_score` com significados opostos, no mesmo painel, é
o risco de verdade. O nome está certo; o cálculo é que é o forasteiro.

Reforça a decisão o fato de o próprio código já converter de volta:
`buildConfigRiskSnapshot` devolve `ScorePenalty: 100 - score`, ou seja, o
agregador de postura já queria o risco e desfazia a inversão na saída.

## O frontend não muda

Verificado, e vale registrar porque é contra-intuitivo:

- O anel preenche `circ * (1 - score/100)` — **maior score preenche mais**. Sob
  semântica de risco isso passa a estar certo, sem tocar no componente.
- `ringColor` e `riskLabel` derivam da contagem de achados, não do score.
- `rbacRiskResponse` **não tem** campo `score`; na aba RBAC o anel já renderiza
  vazio. Só a aba Config é afetada.
- `SecurityInsight.tsx` consome apenas `findings`.

## Regras

- **R1** — `configRiskScore` devolve risco: soma das penalidades, com teto em
  100. Zero significa nenhuma checagem falha.
- **R2** — `computeConfigRiskOverallScore` devolve **0** quando não há achado,
  e não 100.
- **R3** — `buildConfigRiskSnapshot` devolve `ScorePenalty` igual ao score, sem
  o `100 - score`.
- **R4** — Existe teste que fixa a direção da escala: um conjunto de checagens
  graves pontua **mais** que um conjunto leve.

## Critérios de aceitação

1. `legado-privilegiado` (6 falhas, privilegiado) pontua **mais** que
   `coredns` (1 falha).
2. Workload sem nenhuma checagem falha não aparece — e, se a lista estiver
   vazia, o agregado é 0.
3. O anel da aba Config passa a mostrar um número que cresce com o problema.
4. Prova negativa: com `score := 100` e subtração de volta, o teste de R4 falha.
