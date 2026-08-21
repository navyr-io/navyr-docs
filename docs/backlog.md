# Backlog

Lugar único onde todo item vive: o que veio do plano, o que foi descoberto
executando e o que espera decisão sua. Existe porque até 20/08 não existia — as
descobertas iam direto para o trabalho e só apareciam para você depois de
prontas, o que tornou o projeto impossível de gerenciar do seu lado.

**Regra a partir daqui:** nada entra em execução sem estar nesta lista antes. Se
uma descoberta bloqueia o que estou fazendo, eu paro e pergunto — não corrijo e
conto depois.

**Quadro:** https://github.com/orgs/navyr-io/projects/1 — é lá que o trabalho é
gerenciado. Esta página é o detalhamento: a base de cada percentual e o motivo
de cada item estar incompleto.

**Última revisão: 2026-08-20**

---

## Completude por fase

Percentual por item entregue, não por esforço. Um item vale 1; parcial vale a
fração que a coluna "base do número" justifica.

| Fase | Itens | Completude | O que falta |
|---|---|---|---|
| 0 · Migração multi-repo | 6 | **100%** | — |
| 1 · Segurança e CVEs | 5 | **100%** | — |
| 2 · CI que valida | 7 | **86%** | branch protection (0%) |
| 3 · Qualidade e testes | 7 | **94%** | cobertura por risco (60%) |
| 4 · Produção e governança | 5 | **93%** | releases semver (67%) |
| 5 · Fechamento | 4 | **67%** | segurança restante (67%), GitHub Team (0%) |
| **Total** | **34** | **91%** | 30,9 itens de 34 |

---

## Os cinco itens abertos, com a base de cada número

### 2.5 · Branch protection — 0%

**Base:** 0 de 11 repositórios protegidos.

**Por que zero e não "pouco":** não é trabalho não feito, é trabalho impossível.
A API responde `403 — Upgrade to GitHub Pro or make this repository public` em
repositório privado no plano Free. Testei os dois endpoints, `branches/protection`
e `rulesets`. Não há caminho parcial: ou o plano permite, ou nenhum repositório
pode ser protegido.

**Destrava com:** GitHub Team, ou tornar os repositórios públicos.

---

### 3.2 · Cobertura por risco — 60%

Este item tem dois critérios, e eles não andam juntos. O número único esconde
isso, então seguem os dois:

| Critério | Medida | Estado |
|---|---|---|
| Caminhos onde a falha é cara | 5 de 5 | **100%** |
| Serviços com suíte real | 4 de 7 | **57%** |

Os cinco caminhos — isolamento entre organizações, invalidação de sessão,
cifragem de credencial, travessia do túnel e cadeia de migrations — estão
cobertos. São os caminhos em que o defeito é silencioso e a consequência é
vazamento ou acesso indevido.

O piso por serviço é o que falha:

| Serviço | Cobertura | Arquivos de teste |
|---|---|---|
| `navyr-gateway` | 16,2% | 5 |
| `navyr-auth` | 14,1% | 10 |
| `navyr-orchestrator` | 11,8% | 21 |
| `navyr-billing` | 4,3% | 4 |
| `navyr-agent` | **1,3%** | 1 |
| `navyr-community` | **0,0%** | **0** |
| `navyr-collector` | **0,0%** | **0** |

**Por que incompleto:** `community` e `collector` nunca tiveram suíte. O
`collector` foi criado na Fase 0 e nasceu sem teste — falha minha, não herança.
O `agent` ganhou um arquivo de teste e parou aí.

**Nas minhas mãos.** Não depende de decisão nem de conta.

---

### 4.5 · Releases semver — 67%

**Base:** 3 artefatos por repositório × 11 repositórios = 33. Existem 22.

| Artefato | Existe |
|---|---|
| `CHANGELOG.md` | 11 de 11 |
| `.github/workflows/release.yml` | 11 de 11 |
| Tag `v0.1.0` | **0 de 11** |

**Por que incompleto, e por que de propósito:** a tag dispara o workflow que
publica a imagem versionada e cria a GitHub Release. Com o CI recusando iniciar
jobs, uma tag produz release sem imagem — um `v0.1.0` que aponta para nada é
pior que a ausência de tag, porque parece entrega.

**Destrava com:** GitHub Team. É um comando por repositório depois disso.

---

### 5.2 · Segurança restante — 67%

**Base:** 3 sub-itens do plano, 2 fechados.

| Sub-item | Estado |
|---|---|
| Token do manifesto na query string | **Fechado** — o servidor recusa query string; o frontend usa cabeçalho |
| Triagem dos 72 achados de taint | **Fechado** — G704, G705 e G710 religadas no gate; só G702 e G118 seguem fora, com motivo técnico escrito |
| Handoff de SSO | **0%** |

**Por que o terceiro está em zero:** `auth_handler.go` redireciona com
`sso_token` e `refresh_token` na query string, em 2 lugares. Corrigir exige
trocar o mecanismo de entrega — código de uso único trocado por `fetch`, ou
cookie. Não é ajuste de uma linha como foi o do manifesto; é mudança no fluxo de
login federado, que precisa de teste antes.

**Nas minhas mãos**, mas com risco maior que o tamanho sugere.

---

### 5.4 · GitHub Team — 0%

**Base:** 5 passos sequenciais, nenhum possível hoje.

1. Confirmar que o pipeline volta a rodar
2. Deixar os **97 commits** acumulados fecharem verde
3. Mergear os **20 PRs** do Dependabot
4. Criar as tags, conferindo que a imagem existe no registry antes de anunciar
5. Ligar branch protection

**Bloqueio de mão dupla:** este item destrava o 2.5 e o 4.5. Enquanto ele não
sai, três dos cinco itens abertos ficam parados por dependência, não por
prioridade.

---

## Dívidas técnicas

Separadas do plano de propósito: nenhuma delas estava previsto, e é por isso que
esta seção existe.

### Esperando decisão sua

| Dívida | Por que não decido eu |
|---|---|
| **Motor de labs não roda na imagem publicada** | A saída óbvia — acesso ao daemon Docker — é root no host num serviço multi-tenant. Worker separado ou feature self-hosted desligada por padrão |
| **JWT em `localStorage`** | Cookie `HttpOnly` mexe em CORS, CSRF e refresh. É projeto próprio |
| **Cinco arquivos grandes fora da lista do plano** | 3.300 a 1.717 linhas. Mesma técnica já usada, mas entram em escopo por decisão |

### Nas minhas mãos, aguardando prioridade

| Dívida | Tamanho |
|---|---|
| Handoff de SSO com token na URL | Contida, mas no fluxo de login federado |
| `community` e `collector` sem teste | Duas suítes do zero |
| E2E só cobre caminho feliz | Sem erro, permissão negada ou sessão expirada |
| `main()` do gateway com 2.952 linhas | **Exige cobrir o despacho com teste antes** — hoje 15,7% |

### Fora de alcance

As **8 CVEs** restantes são `Fixed in: N/A`. Não há ação além de esperar o
upstream. Ficam listadas no relatório do CI, visíveis, sem bloquear.

---

## Contabilidade das descobertas

Vinte e dois achados foram registrados durante a execução. Cerca de **quinze
nasceram fora do plano** — NetworkPolicy, retry de banco, Monaco e fontes de
CDN, CI que nunca rodou em PR do Dependabot, ferramentas de SAST que não
acompanhavam o toolchain, kubeconfig em três caminhos, motor de labs, colunas
sem migration, e outros.

Eles eram reais e vários eram mais graves que itens planejados. **O problema não
foi encontrá-los, foi o caminho que fizeram:** iam direto para a correção e
apareciam para você prontos, sem nunca terem passado por uma fila visível. Isso
tirou de você a decisão sobre a própria ordem do trabalho.

Esta página é o conserto. Descoberta nova entra aqui como item antes de virar
trabalho.
