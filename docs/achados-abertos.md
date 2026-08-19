# Achados em aberto

Defeitos e riscos encontrados durante a profissionalização da plataforma
(agosto/2026) que **ainda não foram corrigidos**. Cada item registra o que é, por
que importa e o que a correção envolve.

Os achados **já corrigidos** estão nas mensagens de commit dos repositórios
correspondentes; o backlog SEC-04..14 do `NAVYR.md` foi fechado na Fase 1.

**Última revisão: 2026-08-19**

---

## Segurança

### Token em query string
**Onde:** `navyr-orchestrator` (manifesto do agente), `navyr-auth` (callback SSO)

O endpoint de manifesto autentica por `?token=` e o redirect de SSO devolve
`sso_token`, `refresh_token` e e-mail na query string. Query string vaza para
access log de proxy, histórico do navegador e cabeçalho `Referer` de qualquer
recurso externo que a página carregue.

**Correção:** mover para cabeçalho no manifesto — o cliente usa `curl`, então é
viável. No SSO exige mudar o handoff para o frontend, provavelmente via cookie
de uso único ou POST.

### JWT em `localStorage`
**Onde:** `navyr-frontend` — era o SEC-13

Vulnerável a XSS: qualquer script injetado lê o token. Migrar para cookie
`HttpOnly` é mudança de arquitetura de sessão, com impacto em CORS, CSRF e no
fluxo de refresh. Ficou fora da Fase 1 por isso.

### 63 achados de análise de taint sem triagem
**Onde:** todos os serviços Go

O `gosec` reporta 64 achados nas regras G702/G704/G705/G710. Um foi confirmado
real e corrigido (injeção de parâmetro no OAuth do community). Os outros 63 não
foram analisados um a um — foram excluídos do gate porque numa frota cujo
trabalho é proxiar requisições essas regras disparam constantemente e afogariam
o sinal.

**Correção:** uma passada dedicada de triagem, marcando cada um como falso
positivo com justificativa ou corrigindo. Só então voltar a ativá-las no gate.

### Monaco carregado de CDN
**Onde:** `navyr-frontend`

Não há `loader.config()` e `monaco-editor` nunca é importado direto — só
`@monaco-editor/react`. O editor é baixado da CDN jsdelivr em runtime. Três
consequências: a versão servida está fora do nosso controle, a CSP precisa
liberar jsdelivr, e **deploy air-gapped não funciona** — que é objetivo
declarado da Fase 6 do roadmap.

**Correção:** apontar o loader para o pacote local e configurar os workers no
Vite. Exige verificação visual do editor.

---

## Correção e robustez

### Nenhum serviço tem retry de conexão ao banco
**Onde:** todos os serviços Go

O boot faz `log.Fatalf` se o Postgres não estiver pronto. Em Docker Compose o
`restart` recupera; em Kubernetes vira `CrashLoopBackOff` até o banco subir —
recuperável, mas ruidoso e confuso para quem está diagnosticando outra coisa.

### `navyr-collector` reintroduz kubeconfig
**Onde:** `navyr-collector/internal/publish/kubeconfig.go`

Obtém kubeconfig por cluster através da API interna do orchestrator, para o SDK
do Helm operar. O agent tunnel foi construído justamente para a plataforma nunca
precisar de kubeconfig do cluster do cliente — a migration `000009` removeu o
modo direto.

Pode ser decisão consciente (o SDK do Helm não fala pelo túnel), mas contradiz o
diferencial que a documentação vende. **Merece um ADR justificando ou uma
revisão do desenho.**

### Migrations sem versionamento
**Onde:** todos os serviços com banco

Não existe tabela `schema_migrations`. A lista é descoberta em disco e
reexecutada inteira a cada boot, dependendo de todo `CREATE` ter `IF NOT
EXISTS`. Funciona, e é idempotente hoje — verificado. Mas uma migration futura
que não seja idempotente quebra o serviço em todo restart, e o defeito só
aparece no segundo boot.

Quatro migrations não têm `.down.sql`.

---

## Infraestrutura

### NetworkPolicy bloqueia DNS
**Onde:** `navyr-helm/navyr-platform/templates/networkpolicy.yaml`

A regra `backend-allow` libera egress apenas para Postgres:5432. Sem UDP/53,
auth, billing e orchestrator não resolvem nome nenhum — nem entre si, nem para
provedores de IA, KMS ou SMTP. **Com a policy aplicada, a plataforma não
funciona.**

Não foi detectado antes porque o chart nunca tinha sido instalado.

### Três caminhos de deploy paralelos
**Onde:** `navyr-helm`

Helm, Kustomize em `k8s/` e o Compose em `navyr-deploy` descrevem a mesma
plataforma de formas independentes. Já divergem: o Kustomize é mais endurecido
que o Helm em probes e securityContext.

**Correção:** consolidar no Helm — mas portando antes as correções que só
existem no Kustomize.

### Chart ainda se chama `kubeops`
Nome do chart, e todos os recursos usam prefixo `kubeops-`. Resíduo do rebrand.

---

## Qualidade

### Arquivos-deus
| Arquivo | Linhas | Cobertura |
|---|---|---|
| `navyr-orchestrator/internal/handler/kubernetes_handler.go` | 4.316 | 12,7% |
| `navyr-gateway/cmd/server/main.go` | 4.330 | 15,4% |
| `navyr-auth/internal/service/auth_service.go` | 3.808 | 30,5% |
| `navyr-orchestrator/cmd/server/main.go` | 3.033 | 2,9% |
| `navyr-frontend/src/screens/SettingsPage.tsx` | 1.864 | — |

A lógica dentro de `main` não é testável, o que trava a cobertura. Dividir é o
trabalho de **maior risco** do backlog: o `main.go` do gateway é o único ponto
público, com JWT, RBAC e enforcement. Refatorar isso com 15% de cobertura é
refatorar sem rede.

**Ordem recomendada:** cobrir primeiro `orchestrator` e `billing` com o harness
de integração, depois dividir, começando por `kubernetes_handler.go`, que é
fatiável por grupo de recurso e tem risco menor que o gateway.

### E2E incompleto
**Onde:** `navyr-frontend/tests/e2e`

Os 4 specs percorrem 5 dos ~7 passos. As asserções finais falham porque os
payloads mockados não batem com o que a tela de detalhe renderiza. O job roda e
publica relatório, mas não bloqueia (`continue-on-error`).

Os seletores por texto continuam frágeis a mudança de copy. A correção
estrutural é migrar para `data-testid`, como já foi feito nos cards de
organização e de cluster.

### Logging não estruturado
Todos os serviços usam o `log` da stdlib. Sem `org_id`, `cluster_id` ou
`request_id` nos registros, correlacionar um incidente em produção depende de
`grep` e sorte.

### Cobertura abaixo da meta
Meta da Fase 3 é 60% em `internal/service` e `internal/repository`.

Atual: `auth/internal/repository` 6,6% · `orchestrator/internal/service` 11,7% ·
`auth/internal/service` 30,5%. O harness de integração existe em `navyr-auth` e
ainda não foi replicado para `orchestrator` e `billing`.

---

## Vulnerabilidades sem correção upstream

Quatro CVEs alcançáveis em `navyr-orchestrator` e `navyr-collector`, todas
transitivas do SDK do Helm, sem correção publicada. Declaradas em
`.govulncheck-ignore` de cada repositório com justificativa. Mais nove no
binário do `helm` embarcado na imagem do orchestrator, e duas no `dompurify`
que o Monaco traz.

Todas já estão na versão mais recente disponível. Revisar quando o Dependabot
abrir PR para `containerd`, `golang.org/x/crypto` ou `monaco-editor`.

---

## Governança

`LICENSE` e `SECURITY.md` existem apenas em `navyr-docs`. Faltam nos outros 11
repositórios, junto com `CONTRIBUTING.md`, `CODEOWNERS`, `CHANGELOG.md` e
templates de issue e PR.

Sem ADRs e sem runbooks de incidente — os scripts em `navyr-deploy/scripts/ops`
existem e não estão documentados.

`docs/deployment.md` descreve um schema de values do Helm que não corresponde ao
chart real. `openapi.yaml` não existe em `navyr-gateway` nem em `navyr-billing`,
e a spec unificada em `navyr-deploy/spec` convive com as por serviço sem
reconciliação.

**Branch protection** não pode ser configurada: o plano Free da organização
retorna 403 para repositório privado. O CI roda e reprova de forma visível, mas
nada impede o merge.
