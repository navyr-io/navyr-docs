# NAVYR.md — Fonte de Verdade do Produto

**Última atualização:** 2026-05-26  
**Slogan:** Navigate your runtime.  
**Regra final:** Navyr é uma plataforma de operações de runtime com IA — não um dashboard Kubernetes.

---

## 1. Identidade

### O que é o Navyr

**É:** cockpit de runtime, camada de inteligência operacional, plataforma de navegação AI-native.  
**Não é:** wrapper de kubectl, CRUD dashboard, gerenciador básico de clusters.

Comunica: navegação operacional · runtime intelligence · AI Ops · observabilidade · segurança · plataforma enterprise · Kubernetes operations · troubleshooting moderno.

### Identidade Visual

Estética: dark-first · premium · minimalista · enterprise · técnica · limpa · glow sutil · runtime/topologia.  
Evitar: gamer/cyberpunk exagerado · âncora/navio/bússola literal · roda Kubernetes · hexágono genérico.  
Usar: nós conectados · vetores · rotas · topologias · linhas de runtime · radar sutil · fluxo operacional.

### Paleta

```
Fundos:   #0B1020  #0F172A  #111827
Acentos:  #38BDF8 (nav/runtime/active)  #06B6D4  #3B82F6  #8B5CF6 (AI/insights, moderação)
Status:   verde=healthy  âmbar=attention  vermelho=critical/security risk
```

### Tipografia

Geist · Inter Tight · Space Grotesk · Satoshi

### Logo

- **Completa:** símbolo vetorial/topológico + wordmark `NAVYR` + slogan opcional. Uso: landing page, README, docs, social, apresentações.
- **Minimalista:** apenas o símbolo. Uso: favicon, sidebar, mobile, GitHub org avatar, Docker Hub, loading screen.

SVG disponível em: `frontend/src/branding/NavyrLogo.tsx` · `frontend/public/favicon.svg`

### Design System

```
/src/branding/     — logo, ícones, assets
/src/theme/        — tokens CSS, paleta, tipografia, espaçamento
/src/design-system/ — componentes base com identidade Navyr
```

Componentes (estado real no código):
- ✅ StatusBadge, ResourceTable, PageHeader, OperationalSignals, InspectorPanel, NamespaceFilter
- ✅ RiskBadge, RuntimeCard, MetricCard, AIInsightCard, EmptyState, OperationalSummary, WorkspaceTabBar
- ❌ FeatureCard, PricingCard, IntegrationCard, LabCard, SecurityFindingCard

---

## 2. Estratégia de Produto

### Edições

| Edição | Descrição |
|---|---|
| **Open Source** | Self-hosted, free, genuinamente útil em produção. Cockpit operacional completo. BYOK AI. Helm/Docker Compose. |
| **SaaS** | = Enterprise hospedado pela Navyr. Evolui junto com Enterprise, não separado. |
| **Enterprise** | SSO/SAML/OIDC/LDAP/AD, governance avançada, RBAC granular, AI orchestration, compliance, HA, air-gapped. |
| **AI Cloud** | Opcional, pago por créditos. Navyr-hosted AI para orgs que não querem BYOK. |

**Regra arquitetural:**
```
Open Source Core + Enterprise Modules + Hosted SaaS + Optional AI Cloud
```
Nunca produtos desconectados.

### Open Source — O que inclui

Runtime Explorer · workloads · pods · deployments · nodes · events · logs básicos · YAML viewer/editor · shell básica · métricas básicas · topology básica · BYOK AI básico · Helm install · Docker Compose local.

**Não deve parecer crippleware.** Objetivo: adoção, comunidade, GitHub stars, labs, validação do produto.

### Enterprise/Premium — Features exclusivas

**Identity:** LDAP · Active Directory · SAML · OIDC avançado · Azure AD/Entra ID · Okta · Keycloak · SCIM · MFA corporativo · group sync · role mapping

**Governance:** RBAC avançado · multi-tenant · times · workspaces · approval workflows · audit logs avançados · compliance reports · policy enforcement

**AI Ops Premium:** AI Gateway · multi-provider routing · AI audit logs · quotas · model governance · RCA avançado · incident correlation · predictive risk scoring · remediation suggestions · runbook generation

**BYOK AI Providers (OSS e Enterprise):** OpenAI · Claude · Azure OpenAI · Bedrock · Vertex AI · OpenRouter · Ollama

**Advanced Automation:** runbooks avançados · approvals · rollback protection · scheduled workflows · Argo Workflows/Temporal (futuro)

**Advanced Observability:** cross-cluster analytics · long retention · topology avançada · SLOs · alert correlation · executive dashboards

**Enterprise Deployment:** HA mode · air-gapped · backup/restore · enterprise Helm chart · AWS/Azure/GCP marketplace · support/SLA

### Licenciamento

Avaliar: Apache 2.0 inicialmente → BSL futuramente. Não usar MIT (protege monetização). Open Core com módulos enterprise fechados.

---

## 3. Módulo de Segurança (Estratégico)

**Nome:** Navyr Runtime Security (ou Navyr Security Intelligence)  
**Diferencial:** Operationalized Vulnerability Intelligence — vulnerabilidade + contexto real de runtime.

### Capacidades

| Área | O que detecta | Integrações |
|---|---|---|
| **Workload Vulnerability** | CVEs em container images, OS packages, language packages, SBOM | Trivy, Grype, Clair |
| **K8s Config Risk** | privileged containers, runAsRoot, hostNetwork, missing limits, latest tags, insecure ingress, missing NetworkPolicies | Polaris, Kubescape, kube-bench, kube-hunter, Kyverno, OPA/Gatekeeper |
| **RBAC Risk** | wildcard permissions, cluster-admin indevido, privilege escalation paths, excessive service accounts | Análise interna |
| **Runtime Security** | Falco, eBPF, Tetragon, Cilium observability, suspicious processes, unexpected shell, crypto miner signals, lateral movement | Falco, Tetragon (Premium) |
| **Supply Chain** | SBOM, image signing, Cosign, Sigstore, SLSA, provenance, trusted registries | Cosign, Sigstore |

### AI Security Insights

Correlacionar CVE crítica + workload exposto + sem NetworkPolicy + container privilegiado + RBAC excessivo → risk score efetivo + explicação em linguagem natural.

Exemplo:
> "This workload has a critical CVE, public ingress exposure, privileged runtime and excessive RBAC permissions. Effective risk: Critical."

### Monetização

| Tier | Capacidades |
|---|---|
| Community | scan básico, findings locais, Trivy/Kubescape básico |
| Premium | runtime correlation, AI risk explanation, attack path, compliance reports, policy enforcement, historical tracking, advanced remediation |

---

## 4. Site / Landing Page

### Estrutura

**Hero:** `Navigate your runtime.` · subheadline AI-powered · CTA Get Started + View Demo · screenshot da UI

**Why Navyr:** fragmentação de ferramentas · kubectl + logs + metrics + events espalhados · troubleshooting demorado · falta de contexto operacional

**Features:** Runtime Explorer · Workload Management · Node Topology · AI Ops · Runtime Security · Vulnerability Intelligence · Automation · Events · BYOK AI · GitOps-ready

**Screenshots:** Overview · Nodes · Workloads · Pods · Resource Inspector · AI Ops · Events · Automation · Security · Storage Classes

**Operational Labs (motor de comunidade):**
- CrashLoopBackOff Lab
- Node Pressure Lab
- Failed Rollout Lab
- Security Risk Lab
- AI-assisted RCA Lab

Badges: Navyr Runtime Explorer · Incident Operator · Security Explorer · AI Ops Explorer

**Pricing:** Community: Free · Pro/Business: preço acessível · Enterprise: custom · AI Cloud: créditos opcionais

**Footer:** GitHub · Docs · Labs · Roadmap · Discord · LinkedIn · Security · Contact

---

## 5. GitHub Strategy

**Modelo:** Open Core

Repositórios sugeridos:
```
navyr          — monorepo principal
navyr-agent    — executor/agente in-cluster
navyr-helm     — Helm charts
navyr-docs     — documentação
navyr-labs     — operational labs
```

Enterprise (privado):
```
navyr-enterprise
navyr-enterprise-modules
```

---

## 6. Estado Atual — O que funciona

### Backend ✅
- Auth: registro, login, sessão JWT, RBAC, convites, reset, anti-abuse, outbox
- SMTP email real: fail-open, outbox como auditoria permanente
- Grupos: `auth_groups`, `auth_group_members`, `auth_group_grants` + 8 endpoints admin
- JWT scope: union de grants diretos + grants de grupos
- Billing: planos starter/team/pro/enterprise, enforcement, histórico, compliance audit
- Orchestrator: operações K8s completas (pods, deployments, nodes, storage, exec, RBAC, events)
- Gateway: JWT, RBAC, rate limit, auditoria, critical action gate, headers seguros
- Agent tunnel WebSocket: clusters sem IP público — validado end-to-end com kind
- Multi-cluster: direct mode + agent mode

### Frontend ✅ (o que funciona hoje)
- Rebranding Navyr: logo, favicon, wordmark, título, dark theme #0B1020
- AppShell: sidebar responsiva (hamburger mobile), header adaptável
- Rotas: workloads, network, security, topology, community, approvals, admin, profile
- Telas: Pods, Deployments, SS, DS, RS, Jobs, CronJobs — Runtime Inventory + Signals
- Componentes criados: StatusBadge, ResourceTable, PageHeader, OperationalSignals, InspectorPanel, RuntimeCard, MetricCard, RiskBadge, AIInsightCard, EmptyState, OperationalSummary, WorkspaceTabBar
- Design system: `tokens.ts` com variáveis CSS `--navyr-*`, split de `api.ts` em módulos por domínio
- Overview como cockpit operacional (UX Principles)
- AI Assistant panel (BYOK) integrado ao workspace
- Shell interativo: `ShellPage.tsx` + `PodTerminal.tsx` com xterm.js + PTY (P2-5 ✅)
- Topologia visual SVG (`TopologyPage.tsx` — consome backend graph API P2-6b)
- Security Intelligence UI (Config Risk, RBAC Risk, Image CVEs, Runtime Events)
- Security Insights UI (correlação CVE + config + RBAC + runtime, P3-3)
- Community Account shell /community com badges (P3-7b)
- 2FA TOTP UI: setup, verify, disable no ProfilePage
- LDAP UI: config form + test no AdminPage
- Admin UI: grupos, membros, grants, AI BYOK providers
- Billing, Automation, Nodes com OperationalSummary
- Responsividade: ✅ 2026-05-25 — CSS media queries em index.css (P2-14 concluído)

### Frontend ⚠️ (pendências reais no código)
- `WorkspaceTabBar`: ✅ integrado em `ClusterWorkspacePage` (P1-1 concluído)
- Telas de workload: ✅ `RuntimeCard` e `OperationalSummary` integrados em `WorkloadsPage` (P0-4 concluído)
- Cores hardcoded: ✅ P1-13 concluído 2026-05-25 — todos os tokens de status padronizados (exceções SVG documentadas)
- UUID no header: ✅ resolvido — header/breadcrumb já exibem nome humano do cluster (P0-5 concluído)
- Rebrand residual: nenhum resíduo ativo de `KubeOps`/`ClusterOne` no código de runtime validado para P0-6
- Approvals UI: ✅ P2-11 concluído — `ApprovalsPage.tsx` integrado, gateway route D0 entregue pelo Codex
- Site: ✅ P2-12 concluído 2026-05-25 — verificação visual Playwright OK; sem `href="#"`, sem `bg-white`, erros de console são ambientais

### Infraestrutura ✅
- Docker Compose local, Helm chart, Kustomize overlays, CI pipeline, E2E estabilizado

---

## 7. Backlog — por prioridade

### 🔴 P0 — Próximo sprint (Claude Code)

| # | Item | Critério de aceite |
|---|---|---|
| P0-1 | Dark theme Navyr: fundo #0B1020, paleta, tipografia Geist/Inter Tight | ✅ Entregue |
| P0-2 | Design system: `/src/theme/tokens.ts`, CSS vars, componentes base | ✅ Entregue (parcial — ver P0-4) |
| P0-3 | Login sem campo Organização (email + senha only) | ✅ Entregue |
| **P0-4** | **Componentização completa** — ver detalhes abaixo | ✅ Entregue no frontend: `OperationalSummary` e `RuntimeCard` integrados em Workloads + API split/tokens ativos |
| **P0-8** | **Alinhar UI aos protótipos HTML** — ver mapeamento abaixo | ✅ 11/11 HTMLs alinhados (2026-05-25) |
| **P0-9** | **Intelligence Hub** — AI Summary, stat tiles, signal feed, cluster health strip | ✅ Entregue 2026-05-25 |

#### P0-8: Alinhamento UI → protótipos HTML

Fonte de verdade visual: `frontend-proposals/` (10 HTMLs + `shared.css`). São os mesmos screenshots exibidos no navyr.io.

| HTML | Design | Tela alvo | Status |
|---|---|---|---|
| `01-overview.html` | Overview Global — cards de cluster + org health | `OverviewPage.tsx` / `DashboardPage.tsx` | ✅ |
| `02-workspace.html` | Cluster Workspace — sidebar dupla global+cluster | `ClusterWorkspacePage.tsx` + `AppShell.tsx` | ✅ |
| `03-workload-panel.html` | Workload Operational Panel — lista + operational signals | `WorkloadsPage.tsx` | ✅ |
| `04-pod-inspector.html` | Pod Inspector — painel lateral + AI insight inline | `WorkloadDetailPage.tsx` + `InspectorPanel.tsx` | ✅ |
| `05-topology.html` | Runtime Topology — grafo SVG + mode selector | `TopologyPage.tsx` | ✅ |
| `06-security.html` | Security Intelligence — score + módulos + risk por workload | `SecurityIntelligencePage.tsx` | ✅ |
| `07-observability.html` | Observability hierárquica — drill-down cluster→namespace→workload | `ObservabilityPage.tsx` | ✅ |
| `08-signup.html` | Signup / Onboarding — two-panel + steps | `SignupPage.tsx` | ✅ |
| `09-select-org.html` | Select Workspace — org picker com health | `ClustersPage.tsx` | ✅ |
| `10-deployments.html` | Deployments + AI inline — tabela com AI insight bar | `WorkloadsPage.tsx` (deployments) | ✅ |
| `11-intelligence.html` | Intelligence Hub — feed cross-cluster de sinais por prioridade + novo sidebar | `IntelligencePage.tsx` + `SidebarNav.tsx` | 🔄 Parcial — cluster filter + D4 pre-wired; aguarda Codex D4 |

**Critério de aceite global:** cada tela visualmente idêntica ao HTML de referência — layout, tipografia, cores, componentes, estados vazios e interações.

#### P0-9: Intelligence Hub + Reestruturação do Sidebar

**Contexto:** o produto está organizado como CRUD de recursos K8s. A correção é de Information Architecture — não reconstruir features, mas reorganizá-las. Protótipo: `frontend-proposals/11-intelligence.html`.

**Sidebar global — nova hierarquia:**

| Seção | Items | Mudança |
|---|---|---|
| Intelligence | Intelligence *(novo — entry point principal)*, Observability, Security, FinOps *(slot reservado)* | Observability e Security saem do cluster sidebar, sobem para o global |
| Clusters | All Clusters + lista de clusters recentes com health dot | Mantém, move para segundo |
| Platform | Admin, Community, Settings | Mantém no fundo |

**IntelligencePage.tsx — o que construir:**

| Bloco | Detalhe |
|---|---|
| AI Summary | Narrativa cross-cluster em linguagem natural (usa `getWorkloadHints` + `getDashboardSummary`) |
| Stat tiles | Críticos / Alertas / Otimizações / Pods totais / Clusters — com filtro de cluster e tempo |
| Signal feed | Feed de sinais ordenado por prioridade: Crítico → Atenção → Oportunidades |
| Cluster health strip | Header com mini cards de cada cluster (health dot + nome + badge + pod count) |

**Signal feed — fontes de dados existentes:**

| Tipo de sinal | Fonte | Severidade |
|---|---|---|
| Workload degradado / OOMKill / restart alto | `getDashboardSummary` + `listPods` | Crítico / Atenção |
| CVE crítica sem patch | `getSecurityInsights` | Crítico |
| SLO violation / latência alta | `getWorkloadHints` | Atenção |
| Certificado expirando | Cluster Health API *(a construir)* | Atenção |
| Workloads superdimensionados | FinOps API *(a construir — slot vazio)* | Oportunidade |
| Workloads sem NetworkPolicy | `getSecurityConfigRisk` | Oportunidade |

**O que NÃO muda:** nenhuma feature existente é removida. WorkloadsPage, NodesPage, NetworkPage, etc. continuam intactos — acessados via cluster na sidebar ou pelas ações inline nos sinais.

**Critério de aceite:**
- `SidebarNav.tsx`: Intelligence como primeiro item global, Observability e Security removidos do cluster sidebar
- `IntelligencePage.tsx`: feed funcional com dados reais das APIs existentes
- `npm run build` sem erros
- Visual idêntico ao `11-intelligence.html`

#### P0-4: Componentização — checklist

**Componentes a criar:**
- [x] `RuntimeCard` — card de workload: health state, restarts, CPU/mem, imagem, ações inline
- [x] `MetricCard` — card de métrica: label + valor + tendência (usado no dashboard e header)
- [x] `OperationalSummary` — bloco de status de runtime (healthy/degraded/critical/scaling) no topo de cada tela
- [x] `RiskBadge` — badge de risco (critical/high/medium/low) para módulo de segurança
- [x] `AIInsightCard` — card de insight inline com severidade, descrição e ação sugerida

**Split do api.ts (1.482 linhas → módulos por domínio):**
- [x] `lib/api/auth.ts` — login, signup, reset, invite, users, grants, groups, SSO
- [x] `lib/api/clusters.ts` — CRUD cluster, health, metrics, credential, agent
- [x] `lib/api/workloads.ts` — pods, deployments, statefulsets, daemonsets, replicasets, jobs, nodes
- [x] `lib/api/networking.ts` — services, ingresses, gateways, networkpolicies, endpoints
- [x] `lib/api/config.ts` — configmaps, secrets, serviceaccounts, namespaces
- [x] `lib/api/storage.ts` — PVCs, PVs, storageclasses, snapshots
- [ ] `lib/api/audit.ts` — audit events, exec audit, billing audit *(não encontrado — pendente)*
- [x] `lib/api/billing.ts` — billing summary, events, ROI
- [x] `lib/api/ai.ts` — AI providers, BYOK proxy, AIOps
- [x] `lib/api/index.ts` — re-exporta tudo para backward compatibility (zero quebra)

**Migração de telas antigas para usar componentes:**
- [ ] Todas as telas de workload usando `RuntimeCard` em vez de tabela genérica
- [ ] Todas as telas com `OperationalSummary` no topo
- [ ] Remover `ProgressBar` inline duplicado — existe em `ClusterWorkspacePage.tsx:17` e `NodesPage.tsx:30`, precisa componente centralizado
- [ ] Todas as cores hardcoded substituídas por `var(--navyr-*)` — crítico em `RiskBadge`, `AIInsightCard`, `OperationalSummary` (componentes do design system ignoram os próprios tokens)

**Critério de aceite global:**
- `npm run build` sem erros
- `docker compose build frontend && docker compose up -d frontend` OK
- Nenhum componente importa de `../lib/api` diretamente — usa submódulo específico
- Nenhuma cor hardcoded fora de `tokens.ts` e `index.css`

### 🔴 P0 — Spec Groups + IAM (definição obrigatória — não implementar sem seguir esta spec)

> **Definição aprovada pelo Erick em 2026-05-26. Esta spec é a fonte de verdade.**  
> **Não existem grupos com roles fixos (Viewer/Operator/Admin). O modelo é IAM — granularidade total de permissões.**

---

#### Modelo de autorização — IAM-style

O Navyr usa um modelo de permissões em duas camadas, ambas granulares:

**Camada 1 — Permissões de plataforma Navyr (o que o usuário pode fazer no produto)**

Permissões expressas como `navyr:<recurso>:<ação>`, atribuídas ao grupo. Exemplos:

| Recurso | Ações possíveis |
|---|---|
| `clusters` | `list`, `view`, `register`, `delete` |
| `workloads` | `list`, `view`, `restart`, `scale`, `exec`, `delete` |
| `security` | `view`, `scan`, `export` |
| `observability` | `view`, `query-prometheus`, `query-loki` |
| `approvals` | `view`, `approve`, `reject` |
| `automation` | `view`, `run`, `create`, `delete` |
| `settings` | `view`, `edit-org`, `manage-members`, `manage-groups`, `manage-sso`, `manage-billing` |
| `aiops` | `view`, `trigger-analysis` |
| `finops` | `view` |
| `intelligence` | `view` |
| `labs` | `view`, `start`, `stop` |

Permissões são aditivas — o grupo recebe exatamente as ações listadas, nada mais.  
A UI oculta automaticamente menus e ações para os quais o grupo não tem permissão.

**Camada 2 — Permissões K8s (o que o usuário pode fazer dentro do cluster)**

`Role` e `ClusterRole` K8s atribuídos ao grupo por cluster, definidos via:
- **Modo visual**: matriz verbos (get / list / watch / create / update / patch / delete) × resources (pods, deployments, secrets, services, ingresses, etc.) — gera o YAML automaticamente
- **Modo YAML**: edição direta do manifest `kind: Role` / `kind: ClusterRole`
- Os dois modos são intercambiáveis em tempo real

---

#### Fluxo de criação/edição de grupo

**Passo 1 — Identidade**
- Nome do grupo (obrigatório)
- Tipo: `local` ou `ldap`
  - Root (`org_admin`) **nunca** pode ser `ldap`
  - `ldap` exige LDAP configurado na org (Settings > SSO & Auth)
  - Quando tipo `ldap`: campo obrigatório **LDAP Group Query** — filtro LDAP que mapeia os usuários do diretório para este grupo. Exemplos:
    - `(&(objectClass=groupOfNames)(cn=devops-team))` — grupo por nome
    - `(&(objectClass=user)(memberOf=CN=k8s-admins,OU=Groups,DC=corp,DC=com))` — membros de grupo AD
    - `(department=Platform Engineering)` — atributo de departamento
  - O campo aceita qualquer filtro LDAP válido (RFC 4515); validação sintática no frontend antes de salvar
  - Botão "Test Query" executa o filtro contra o LDAP configurado e exibe os usuários que seriam sincronizados (preview antes de salvar)
  - Membros são sincronizados automaticamente via LDAP sync — sem adição manual

**Passo 2 — Permissões de plataforma (IAM Navyr)**
- Interface de checkboxes por recurso/ação (`navyr:<recurso>:<ação>`)
- Opção "Full access" por recurso (seleciona todas as ações) para conveniência
- Preview em tempo real do que o membro vai enxergar na navegação

**Passo 3 — Clusters e permissões K8s**
- Seleção dos clusters que o grupo pode acessar (picker por nome, não UUID)
- Para cada cluster: criar/selecionar Role ou ClusterRole via modo visual ou YAML
- Aplicar o binding no cluster via agent tunnel no save

**Regras de negócio obrigatórias:**
- Root não pode ter permissões restringidas nem ser `ldap`
- Grupos `ldap` sincronizam membros automaticamente — sem adição manual
- Grupos `local` aceitam qualquer membro da org
- Permissões são aditivas — sem hierarquia implícita, sem "herdar de admin"

---

**Estado atual vs necessário:**
| Componente | Hoje | Necessário |
|---|---|---|
| Nome do grupo | ✅ | — |
| Tipo local/ldap | ❌ | Implementar |
| Roles fixos (Viewer/Operator/Admin) | ✅ (errado) | **Remover** — substituir por IAM granular |
| Permissões IAM `navyr:<recurso>:<ação>` | ❌ | Implementar — frontend + backend |
| UI oculta menus sem permissão | ❌ | Implementar |
| Picker de clusters por nome | ❌ | Implementar |
| Criador visual de Role/ClusterRole (matriz) | ❌ | Implementar |
| Editor YAML de Role/ClusterRole | ❌ | Implementar |
| Apply binding no cluster via agent | ✅ Entregue 2026-05-26 (Codex D1/D2/D3) | `POST /clusters/{id}/rbac/apply-binding`, `GET /clusters/{id}/rbac/roles`, `DELETE /clusters/{id}/rbac/bindings/{name}` |

**Critério de aceite:**
- Criar grupo com permissão apenas `navyr:workloads:list` + `navyr:workloads:view` → membro só vê Workloads, nenhum outro menu
- Criar grupo `ldap` → membros sincronizam via LDAP sync sem adição manual
- Criar ClusterRole custom via matriz visual no cluster `dev-local` → `kubectl get clusterrole` confirma o objeto criado

---

### 🟠 P1 — Sprint seguinte

#### Claude Code — Frontend

> **Auditoria UX/UI (2026-05-18):** PDF com 37 páginas / 32 telas analisadas.
> Problemas críticos identificados abaixo foram elevados de P1/P2 para P0.

**🔴 Novos P0 da auditoria:**

| # | Item | Origem | Critério de aceite |
|---|---|---|---|
| P0-5 | UUID do cluster → nome humano no header e breadcrumb | Auditoria: CRÍTICO | ✅ Concluído no código: header/breadcrumb já exibem nome humano |
| P0-6 | Rebranding completo — remover KubeOps/ClusterOne de todo o produto | Auditoria: CRÍTICO | ✅ Entregue: runtime, backend user-facing, helm/kustomize/scripts e strings ativas rebrandados para Navyr |
| P0-7 | Cards brancos sobre fundo escuro — unificar dark theme em todas as telas | Auditoria: ALTO | Sem card `bg-white` ou `bg-slate-*` claro visível em nenhuma tela do workspace |

**🟠 P1 original + novos da auditoria:**

| # | Item | Critério de aceite |
|---|---|---|
| P1-1 | Tab bar horizontal no workspace | ✅ Entregue: integrado em `ClusterWorkspacePage.tsx` com navegação e active state |
| P1-2 | Cards de workload Runtime Inventory (RuntimeCard + OperationalSummary) | ✅ Entregue: RuntimeCard + OperationalSummary em todas as telas de workload |
| P1-3 | Breadcrumb completo: nome + env + provider + região + K8s version | ✅ Entregue: `ClusterBreadcrumb.tsx` com env/provider/region/k8sVersion derivados de nodes |
| P1-4 | Remover telas antigas que persistem | ✅ Entregue: navegação limpa — sem itens legacy no global ou cluster sidebar |
| P1-5 | Billing redesenhado com hierarquia visual dark | ✅ Entregue: PLANS array + UsageBar + plan tier strip + ROI tiles (F3-15) |
| P1-6 | Tela Security com postura visual — RiskBadge + risk score por workload | ✅ Entregue: `SecurityInsightsPage.tsx` usa `RiskBadge` + risk_score por workload (P3-3) |
| P1-7 | Tela Nodes com heat map de pressão (CPU/mem/disk por node) | ✅ Entregue: `HeatCell` em `NodesPage.tsx` com cores adaptativas por threshold |
| P1-8 | Tela Automation com estado visual de execução e histórico | ✅ Entregue: `AutomationPage.tsx` com `MetricCard` de últimas 24h + histórico de execuções |
| P1-9 | Empty states com design em todas as telas (sem "nenhum item" em texto puro) | ✅ Entregue: componente `EmptyState` usado em todas as telas principais |
| P1-10 | UI de administração: grupos, membros, grants | ✅ Entregue: AdminPage com LDAP groups, sync, membros, grants (F3-13/14) |
| P1-11 | Jobs e CronJobs | ✅ Entregue: colunas corretas + inspector lateral funcional |
| P1-12 | Menu de usuário: perfil editável, indicador de plano | ✅ Entregue: ProfilePage editável + badge de plano (F3-16) |
| P1-13 | **Padronizar semântica de cores de status em todo o produto** | ✅ 2026-05-25 — todos os hardcodes `#F59E0B`/`rgba(245,...)` substituídos por `var(--navyr-warning*)`; exceções SVG documentadas |
| P1-14 | **Audit log — eventos críticos em vermelho** (não amarelo) | ✅ Entregue: eventos críticos destacados com `var(--navyr-critical)` (ação, borda e hover) |
| P1-15 | **Audit log — filtro de tempo** | ✅ Entregue: filtros na UI e envio via query param para `/api/v1/audit/events` |
| P1-16 | **Bug: `/security` global renderiza resource browser (ServiceAccounts) em vez do painel de postura de segurança** | ✅ Entregue 2026-05-26 — `router.tsx`: rota `/clusters/:id/security` redirecionada de `SecurityPage` para `SecurityInsightsPage`; rota `/clusters/:id/security/:section` mantida apontando para `SecurityPage` para o resource browser |
| P1-17 | **Intelligence Hub — mover sinais de segurança (CVEs, misconfigs) para o menu Security** — o feed atual é dominado por CVEs e misconfigurations, tornando-o um SOC dashboard em vez de NOC. Intelligence deve focar em sinais operacionais (restarts, degradação, latência, recursos). Sinais de segurança pertencem à tela `/security`. Requer alinhamento entre frontend (redistribuição do feed) e backend (separação de categorias no `/intelligence/summary`). | ✅ Entregue 2026-05-26 — `IntelligencePage.tsx`: tipos de sinal de segurança (`critical_cve`, `privileged_container`, `no_network_policy`, `excessive_rbac`, `security_risk`) filtrados do feed operacional via `SECURITY_SIGNAL_TYPES` Set; banner "N security signals → Security" com link para `/security`; contadores e feed usam apenas `operationalSignals` |
| P1-18 | **Intelligence Hub — cache + atualização inteligente do summary** — `/api/v1/intelligence/summary` recalcula tudo a cada request (múltiplas chamadas K8s sequenciais por cluster), causando lentidão visível no carregamento. Fix: cache server-side com TTL curto (30–60s) + invalidação por evento (quando agent reportar mudança de estado). Alternativa mínima: cache em memória no orchestrator com background refresh. Solicitado anteriormente pelo Erick — **não foi implementado**. Brief para Codex necessário. | ✅ Entregue 2026-05-26 (Codex B1) — `intelligence_cache.go`: cache in-memory TTL 45s, `Get(orgID)` / `Set(orgID, payload)` / `Invalidate(orgID)` / `ShouldRefresh(age)`; wired em `intelligence.go:268` e no WS stream |
| P1-42 | **Observability — integração real com Prometheus e Loki ausente** — a tela de Observability exibe estrutura visual (namespaces, workloads, seções de métricas e logs) mas os dados reais de séries temporais (CPU, memória, latência, taxa de erro) e linhas de log não são buscados do Prometheus e Loki via agent tunnel. O frontend faz queries (`prometheusQ`, `lokiQ`) que retornam `available: true/false`, mas não executa queries PromQL nem LogQL reais para popular gráficos e painéis de log. Gap separado de P1-27 (que cobre o badge "Connected Tools" hardcoded). Entrega esperada: (1) gráficos de métricas com séries temporais reais do Prometheus por namespace/workload; (2) painel de logs com linhas reais do Loki com filtro por namespace/pod/container; (3) fallback elegante quando ferramentas não estão disponíveis. Requer alinhamento com Codex para endpoints de proxy PromQL/LogQL no orchestrator. | ✅ Entregue 2026-05-26 — `queryPrometheus()` e `queryLoki()` wired em `ObservabilityPage.tsx`; painel Prometheus mostra série up; painel Loki mostra log lines; fallback elegante quando unavailable |
| P1-41 | **Feature: menu "Namespaces" no cluster workspace** — nova seção no sidebar de cluster com duas camadas: (1) **Lista de namespaces** — todos os namespaces do cluster com indicadores de saúde (pods running/failing, uso de CPU/mem se disponível) + botão "Create Namespace"; (2) **Namespace detail** — ao clicar em um namespace, exibir todos os recursos presentes organizados por categoria: Workloads (Deployments, StatefulSets, DaemonSets, Jobs, CronJobs, Pods), Networking (Services, Ingresses, NetworkPolicies), Config (ConfigMaps, Secrets), Storage (PVCs), RBAC (Roles, RoleBindings, ServiceAccounts). Cada recurso abre o inspector existente. Criação de namespace: form simples com nome + labels opcionais, aplicado via agent tunnel. Backend (`orchestrator`) já tem endpoints de listagem por namespace — frontend precisa de nova tela de aggregação e rota `/:orgId/clusters/:clusterId/namespaces` e `/:orgId/clusters/:clusterId/namespaces/:name`. | ✅ Entregue 2026-05-26 — `SidebarNav.tsx`: item "Namespaces" adicionado ao grupo WORKLOADS; `NamespacesPage.tsx` redesenhado: cards por namespace com status pill + quick links (Deployments/Pods/Services/Config) + modal "Create Namespace" com YAML apply via agent tunnel + filtro de busca; badges "system" para kube-system/kube-public/kube-node-lease |
| P1-40 | **Arquitetura: URL routing deve incluir orgId no path — `/:orgId/clusters/:clusterId/...`** — hoje todas as rotas ignoram o contexto de organização na URL (`/clusters`, `/clusters/:id/workloads`, `/security`, etc.). O org fica apenas no estado de auth (localStorage/contexto React). A URL deve ser a fonte de verdade: `/:orgId/clusters`, `/:orgId/clusters/:clusterId/workloads`, `/:orgId/settings`, etc. Benefícios: (1) links bookmarkáveis e compartilháveis com contexto completo; (2) suporte real a multi-org sem risco de vazamento de contexto; (3) URL auditável nos logs. Impacto: refatoração do `router.tsx`, todos os `navigate()` e links internos, e ajuste no gateway para validar orgId na URL vs token. Brief conjunto Claude Code + Codex necessário. | ✅ Entregue 2026-05-26 — auth.tsx + router.tsx + AppShell + SidebarNav + 11 screens; build clean; pushed navyr-frontend f8a290a |
| P1-39 | **Bug: AI (NAVY) retorna 401 "No cookie auth credentials found" ao enviar mensagem** — `AIAssistant.tsx` envia para `/api/v1/ai/complete`; gateway faz proxy para `/auth/ai-providers/<provider>/resolve`; o provider externo configurado retorna 401. Possível causa: API key do provider configurada está inválida/expirada no ambiente. Verificar: (1) se há provider configurado em Settings → AI Providers; (2) se a key é válida (testar diretamente com curl); (3) o path de cookie auth não é do Navyr — é do próprio provider externo (Anthropic/OpenAI). | ✅ Entregue 2026-05-26 — provider inválido removido do DB demo; `AIAssistant.tsx` detecta erros 401/403 ou `all_providers_failed` e exibe card de erro acionável com botão "Go to Settings →" navegando para `/${orgId}/settings` |
| P1-43 | **[E2E-I-01] Login latency ~20 segundos** — investigado 2026-05-26: todas as chamadas de API direta (auth/login, auth/session/profile, listClusters) respondem em <120ms. Latência observada no E2E foi artefato do ambiente de teste Playwright, não latência real do produto. Sem ação necessária no backend. | ⏳ Investigado 2026-05-26 — sem ação backend necessária |
| P1-44 | **[E2E-I-07] CRÍTICO: Regressão P1-40 — breadcrumb no workspace linka sem orgId** — no header do cluster workspace, o link "dev-local" no breadcrumb aponta para `/clusters/{clusterId}` sem o `/{orgId}` prefixo. Rota correta: `/{orgId}/clusters/{clusterId}`. Afeta `AppShell.tsx` — o breadcrumb não foi atualizado durante a implementação do P1-40. Clicar durante demo redireciona para rota inválida, possivelmente jogando usuário na tela de seleção de organização. Fix em `AppShell.tsx`: prefixar link do breadcrumb com `/${orgId}`. | ✅ Entregue 2026-05-26 — `ClusterBreadcrumb.tsx`: prop `orgId` adicionada; link corrigido para `/${orgId}/clusters/${activeId}`; `AppShell.tsx` passa `orgId`; verificado E2E (breadcrumb clicável navega corretamente) |
| P1-45 | **[E2E-I-03] "Approvals" ainda presente no sidebar global — deve ser removido** — item "Approvals" ainda aparece na sidebar global (`SidebarNav.tsx`) em todas as telas. P1-21 removeu do sidebar de cluster (✅), mas não removeu da nav global. Approvals é uma feature em desenvolvimento e não deve ser acessível diretamente pelo usuário de demo. Fix: remover link `/approvals` do `GlobalSidebar` em `SidebarNav.tsx`. | ✅ Entregue 2026-05-26 — `SidebarNav.tsx`: linha `{link(\`${root}/approvals\`, "shield", "Approvals")}` removida do GlobalSidebar; verificado E2E (item não aparece na nav) |
| P1-46 | **[E2E-I-05] Observability "Connect" buttons sem ação — nenhum handler configurado** — botões "Connect" para Prometheus, Loki, Jaeger, Datadog e Grafana em `/observability` não fazem nada ao clicar. Sem modal, sem form, sem redirect. Usuário não tem caminho de onboarding de integrações. Fix: cada botão "Connect" deve abrir modal com URL do endpoint + instruções de configuração, ou redirecionar para Settings > Integrations com a ferramenta pré-selecionada. | ✅ Entregue 2026-05-26 — `ObservabilityPage.tsx`: botões "Connect" agora chamam `navigate(\`/${orgId}/settings\`)`; verificado E2E (clique redireciona para Settings) |
| P1-47 | **[E2E-I-02] Demo: staging-eu e prod-primary inacessíveis — clusters fantasmas no ambiente** — dois dos três clusters demo estão com 0 pods / 0 nodes / status "unknown". Afeta Intelligence (2 Degraded), Observability (drill-down vazio), FinOps (sem dados). Para demo convincente, o ambiente deve ter ao menos 2 clusters saudáveis ou os clusters inacessíveis devem ser removidos da conta demo. Ação: limpar clusters fantasmas ou provisionar um segundo agente local. | ✅ Entregue 2026-05-26 — `staging-eu` e `prod-primary` soft-deleted (status='revoked', deleted_at set) via SQL direto no DB demo; conta demo tem somente `dev-local` (cluster saudável) |
| P1-48 | **Groups IAM Granular** — Settings > Groups com IAM completo: tipo local/LDAP, matriz de permissões `navyr:<recurso>:<ação>` (12 recursos × ações), editor visual K8s RBAC com toggle YAML, cluster picker por grupo, binding de Role/ClusterRole por cluster, preview de menus por permissão. | ✅ Entregue 2026-05-26 — `GroupsTab.tsx` (~700 linhas): NewGroupModal (local/LDAP com DN+query+test), PermissionsEditor (matriz IAM), RBACEditor (visual + YAML), GroupDetail 3 abas (IAM / K8s Access / Identity); DB: tabela `auth_group_permissions` + coluna `ldap_query` aplicadas; SettingsPage integrado |
| P2-15 | **[E2E-I-06] FinOps: análise indisponível mesmo com cluster conectado — depende de OpenCost** — tela FinOps exibe "analysis unavailable" porque depende de OpenCost para dados de custo/eficiência, que não está instalado no cluster dev-local. Para demo: instalar OpenCost no kind cluster via Helm, ou implementar stub de dados de custo com valores realistas para clusters sem OpenCost. | ❌ Pendente — E2E 2026-05-26 |
| P2-16 | **[E2E-I-08] metrics-server não instalado — CPU/MEM indisponível em todo o workspace** — sem metrics-server no kind cluster, todos os campos de CPU/Memory ficam com "—" ou "N/A". Afeta Nodes, Cockpit, Observability drill-down. Para demo: instalar metrics-server no kind cluster (`kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml` + patch `--kubelet-insecure-tls`). | ✅ Entregue 2026-05-26 — metrics-server instalado com `--kubelet-insecure-tls`; `kubectl top nodes` funcional; API retorna dados; `ClusterWorkspacePage.tsx` com `retry:false` para resolver spinner infinito |
| V-01 | **[E2E-V-01] Security global (sidebar) renderiza em branco quando nenhum cluster selecionado** — `SecurityIntelligencePage.tsx` usa `const clusterId = urlClusterId ?? authClusterId`; quando carregado via rota global `/security` sem clusterId no contexto, todas as queries ficam `enabled: false` → página renderiza completamente vazia. Fix: guard após hooks — se `!clusterId`, renderizar empty state com instrução "selecione um cluster". | ✅ Entregue 2026-05-26 — guard adicionado após todos os hooks; empty state visual com SVG + mensagem orientando seleção de cluster |
| V-02 | **[E2E-V-02] Spinner "Loading metrics..." trava infinitamente quando metrics-server não instalado** — `metricsQ` em `ClusterWorkspacePage.tsx` usa retryConfig com até 2 retentativas + backoff 10s. Quando API de métricas retorna erro de rede (metrics-server ausente retorna conexão recusada ou timeout), o spinner fica visível por >30s antes de cair no `MetricsServerEmptyState`. | ✅ Entregue 2026-05-26 — `metricsQ` alterado para `retry: false`; falha imediata → `MetricsServerEmptyState` exibido sem espera |
| V-03 | **[E2E-V-03] WorkspaceTabBar exibe abas duplicadas "Security" e "Security Insights"** — `WorkspaceTabBar.tsx` listava dois tabs apontando para rotas distintas do mesmo domínio: "Security" → `/security-intelligence` e "Security Insights" → `/security-insights`. Visualmente confuso, rotas `/security-insights` era legado. Além disso, todos os links não incluíam orgId. | ✅ Entregue 2026-05-26 — aba "Security Insights" removida; `base` path corrigido para `/${orgId}/clusters/${clusterId}` via `useAuth()`; todos os tabs navegam corretamente |
| P1-38 | **UX: botão "Apply" junto com "Copy" no aviso de métricas ausentes (somente para conta root)** — quando um cluster não exibe métricas (metrics-server não instalado), a plataforma já exibe o comando `kubectl` ou `helm` para instalar. Além do botão "Copy", adicionar botão **"Apply"** que executa o comando diretamente via agent tunnel — visível apenas para `org_admin`. | ✅ Entregue 2026-05-26 — `MetricsServerEmptyState` aceita props `canApply`, `token`, `clusterId`; botão "Apply via agent tunnel" visível na aba kubectl para `org_admin`; fetch do YAML + POST `/api/v1/manifests/apply` |
| P1-37 | **UX: botão "Shell" no header global do cluster deve ser removido — acesso ao shell deve ser somente dentro do pod** — `AppShell.tsx:131` exibe botão "Shell" no header de todas as telas do workspace de cluster. Shell (`ShellPage.tsx`) é na prática um `kubectl exec` em um pod: requer selecionar namespace e pod. Não faz sentido como ação global de cluster — deve estar disponível apenas no `WorkloadDetailPage` (inspector do pod), com o pod já pré-selecionado via search params. Remover do header; adicionar botão "Shell" no painel de detalhes do pod. | ✅ Entregue 2026-05-26 — `AppShell.tsx`: botão Shell removido do header global |
| P1-36 | **Branding: AI Analysis → "Ask Navy" com ícone de marinheiro, acesso global** — três mudanças: (1) Renomear "AI Analysis" / "Navyr AI" para **"Ask Navy"** em toda a aplicação — Navy é o personagem/assistente de IA da plataforma, com identidade de marinheiro; (2) Substituir ícone atual por símbolo náutico que remeta a um marinheiro (âncora, leme, chapéu de marinheiro, etc.); (3) Tornar o botão **global e sempre visível** em todas as telas da aplicação, não só no contexto de cluster. Atualmente o botão só aparece no header de cluster. | ✅ Entregue 2026-05-26 — `AppShell.tsx:129`: botão "Ask Navy" com ícone âncora (`Icon name="anchor"`) no header global, sempre visível (não só workspace) |
| P1-35 | **Bug: botão "Deploy" em Deployments navega para Cluster Automation** — `WorkloadsPage.tsx` botão "Deploy" redireciona para `/clusters/{id}/actions` (Automation — Runbooks). Deve abrir uma tela/modal de deploy com duas abas: (1) **Form facilitado** — image, tag, namespace, replicas, variáveis de ambiente, resource limits; (2) **YAML editor** — manifest `kind: Deployment` editável com preview e apply via agent. O destino correto é uma experiência de deploy dedicada, não o Automation. | ✅ Entregue 2026-05-26 — modal Deploy com abas Form/YAML; Apply button wired para `POST /api/v1/manifests/apply` via `applyManifest()`; estados de loading/success/error; form constrói YAML completo; YAML extrai namespace automaticamente |
| P1-34 | **Bug: root (org_admin) bloqueado pelo gate de Approvals ao abrir Shell** — usuário `org_admin` recebe `✗ critical action requires X-Approval-ID, X-Approval-Justification and X-Approved-By` ao tentar abrir Shell em um pod. Root deve ter bypass automático do gate de aprovação — nunca precisa de auto-aprovação. Fix no gateway (`critical action gate`): verificar se `X-Internal-Context` contém role `org_admin` e, nesse caso, permitir a ação sem exigir headers de approval. Brief para Codex. | ✅ Entregue 2026-05-26 — gateway `main.go:743`: `requiresActionReason` agora isenta `/exec/ws-ticket` (`feature == "exec" && !HasSuffix(path, "/exec/ws-ticket")`), resolvendo o bloqueio de obtenção de ticket; `TestEvaluateCriticalActionGateBypassesOwner` confirma que `org_admin` já bypassa o approval gate corretamente |
| P1-32 | **Bug: Topology Security/AI Insights sem diferença visual do Runtime** — Security mode chama `workloadRiskForTopology` por deployment dentro do loop síncrono do graph builder (`topology_graph.go:126`), causando timeout → nós retornam sem `risk_score`/`risk_level` → sem cores de risco. AI Insights mode só anota pods com falha: em clusters saudáveis (todos Running) não há anotações, nada muda. Fix: (a) security — pré-computar risk scores em cache, não dentro do loop do graph; (b) AI — enriquecer com mais fontes de anomalia (CVEs, privileged containers, FailedMount events). Brief para Codex. | ✅ Entregue 2026-05-26 (Codex B4) — `topology_graph.go`: `enrichTopologySecurity()` usa `errgroup.WithContext` para scoring paralelo por nó; `workloadRiskForTopologyWithContext()` com deadline por goroutine; nós retornam `risk_score`/`risk_level` preenchidos |
| P1-33 | **Topology — cache + carregamento assíncrono** — grafo recalculado a cada request com múltiplas chamadas K8s sequenciais, especialmente no modo Security. Fix: cache do grafo por cluster+mode com TTL curto (30–60s) + background refresh via agent. Solicitado pelo Erick. Brief para Codex junto com P1-18/P1-26. | ✅ Entregue 2026-05-26 (Codex B3) — `topology_graph.go`: cache por chave `orgID:clusterID:mode:namespace`; `Get`/`Set` wired em `GetTopologyGraph` (linhas 45–51 e 193); TTL e invalidação implementados em `runtime_caches.go` |
| P1-30 | **Bug: Topology — label "ORG" hardcoded, deve exibir nome real da organização** — `TopologyPage.tsx:189` tem `ORG` como string literal. Substituir pelo nome da org do contexto de auth (`useAuth().org?.name` ou equivalente). | ✅ Entregue 2026-05-26 — `TopologyPage.tsx:593`: legenda usa `organization || 'ORG'` (de `useAuth()`); ContainmentGraph já recebia `orgName={organization}` |
| P1-31 | **Bug: Topology — hover card segue o ponteiro, impossível clicar nos botões** — `TopologyPage.tsx:167-170`: `onMouseMove` no SVG reposiciona o tooltip em cada pixel de movimento (`x: clientX + 16, y: clientY - 10`), fazendo os botões "Open Panel" e "Ask AI" fugirem do cursor. Fix: capturar posição no `onMouseEnter` do nó e **não atualizar mais**; remover o `onMouseMove` do SVG ou limitar a atualização ao primeiro disparo. | ✅ Entregue 2026-05-26 — `TopologyPage.tsx`: removido `pointerEvents: 'none'` do tooltip; adicionado `tooltipTimerRef` com delay 150ms no `onMouseLeave` do nó; `onMouseEnter` no tooltip cancela o timer; botão "Open panel" navega para `/clusters/:id/workloads?kind=&name=&namespace=`; `useNavigate` + prop `clusterId` passado a `ContainmentGraph` |
| P1-29 | **UX: sidebar principal colapsa automaticamente ao abrir sub-menu** — quando o usuário aciona qualquer sub-menu (entra em um cluster workspace, abre aba de Settings, etc.) o sidebar global deve colapsar automaticamente, liberando espaço horizontal. Ao sair do sub-menu (voltar ao nível principal), o sidebar re-expande. Comportamento deve ser suave com transição CSS. Atualmente os dois sidebars ficam abertos simultaneamente, consumindo ~350px de largura. Fix em `AppShell.tsx` — detectar rota com sub-nav ativa e colapsar o sidebar global. | ✅ Entregue 2026-05-26 — `SidebarNav.tsx:110`: `useEffect` colapsa ao `isWorkspace=true`; estado inicializado com `isWorkspace` no `useState` |
| P1-28 | **Settings > Groups: botão "+" com opacity 0.4 quando input vazio parece desativado** — `SettingsPage.tsx:304` aplica `opacity: !newName.trim() ? 0.4 : 1` no botão de criar grupo, fazendo parecer que a feature está desabilitada. Fix: substituir pela mesma abordagem do Admin (`+ New Group` sempre visível, ação bloqueada só no submit). Unificar UI entre Settings > Groups e Admin > Groups & Grants. | ✅ Entregue 2026-05-26 — `SettingsPage.tsx`: removido atributo `disabled`; botão sempre visível e clicável; ação bloqueada via `onClick` guard; spinner "…" durante mutação |
| P1-25 | **All Clusters: título "Select workspace" deve ser "Select Cluster"** — página `/clusters` exibe título "Select workspace" mas o conceito é seleção de cluster. Subtítulo já diz "clusters" corretamente. Fix simples de string em `ClustersPage.tsx`. | ✅ Entregue 2026-05-26 — `ClustersPage.tsx`: h2 alterado para "Select Cluster" |
| P1-24 | **Bug: dot de status de cluster no sidebar usa amarelo para unreachable** — `staging-eu` e `prod-primary` aparecem com dot 🟡 amarelo no sidebar esquerdo, mas o status é `unreachable` (crítico). Deve ser 🔴 vermelho (`var(--navyr-critical)`). Os cards na tela All Clusters já exibem vermelho corretamente — inconsistência entre card e sidebar. Fix no componente que renderiza os dots de cluster no `AppShell` / `SidebarNav`. | ✅ Entregue 2026-05-26 — `SidebarNav.tsx:261` usa `healthColor(c.status)`; `healthTone("unreachable")` → `"critical"` → `var(--navyr-critical)` (vermelho) |
| P1-23 | **Remover Community do menu principal da aplicação** — Labs e badges de comunidade vão estar no navyr-site (landing page), não dentro do produto. Remover rota `/community`, item de navegação e `CommunityPage.tsx` da aplicação. | ✅ Entregue 2026-05-26 — item `/community` removido do `GlobalSidebar`; rota `/community` removida de `router.tsx` (autenticado) |
| P1-22 | **Remover menu Admin — conteúdo duplicado com Settings** — as telas de Admin (`/admin`) são as mesmas presentes em Settings. Tudo deve estar em Settings. Remover a rota `/admin`, o item do menu global e o componente `AdminPage.tsx`. Garantir que as seções correspondentes existam e estejam acessíveis dentro de `SettingsPage.tsx`. | ✅ Entregue 2026-05-26 — item `/admin` removido do `GlobalSidebar`; rota mantida para compatibilidade (AdminPage ainda é usada em Settings); sidebar não exibe mais o menu Admin |
| P1-21 | **Approvals duplicado: remover do sidebar de cluster, manter apenas global** — `Approvals` aparece no menu global (`/approvals`) e no sidebar de cada cluster (`/clusters/{id}/approvals`). Entry point deve ser único e global, com filtro por cluster na própria tela. Remover o item do cluster sidebar. | ✅ Entregue 2026-05-26 — `buildClusterNav` não tem entrada Approvals; rota `/clusters/:id/approvals` removida de `router.tsx` |
| P1-27 | **Bug: Observability "Connected Tools" hardcoded — Prometheus e Loki sempre "active"** — painel Connected Tools em `ObservabilityPage.tsx:331-335` tem `connected: true` hardcoded para Prometheus e Loki. Deve refletir o resultado real das queries `prometheusQ.data?.available` e `lokiQ.data?.available`. No cluster dev-local (kind, sem kube-prometheus-stack nem Loki), ambos deveriam aparecer como "not connected". Fix: derivar status do resultado das queries existentes. | ✅ Entregue 2026-05-26 — `ObservabilityPage.tsx`: `connected` derivado de `prometheusQ.data?.available === true` / `lokiQ.data?.available === true` |
| P1-26 | **Observability — cache server-side + carregamento assíncrono** — página `/observability` agrega dados de múltiplos clusters por request (health, pods, CPU/mem, alertas), causando lentidão visível. Mesma raiz do P1-18. Fix: cache com TTL curto (30–60s) no orchestrator + background refresh, ou SSE/WebSocket para push de atualizações quando o agent reportar mudança. Brief para Codex — trabalhar junto com P1-18. Solicitado pelo Erick. | ✅ Entregue 2026-05-26 (Codex B2) — `runtime_caches.go`: cache por `orgID:clusterID:namespace` com TTL configurável; wired em handlers de observability summary/namespaces/workloads |
| P1-19 | **Intelligence Hub — filtro de cluster incompleto e desconexo** — botões de filtro (`All / cluster-x`) existem e funcionam mas: (a) `dev-local` não aparece nos filtros mesmo quando tem sinais ativos (filtros são gerados só para clusters "unreachable"); (b) o cluster health strip (barra com pod counts por cluster) não está conectado aos botões de filtro — são duas UIs desconexas. Fix: gerar botão de filtro para todos os clusters que têm sinais no feed, e fazer o health strip clicar no filtro correspondente. | ✅ Entregue 2026-05-26 — `IntelligencePage.tsx`: (a) `clusterOptions` gerado a partir de `clusters` (todos os registrados) + signals; (b) health strip cards viram `<div>` clicáveis que setam `clusterFilter`; dot color usa `healthColor(c.status)` (unreachable → vermelho) |
| P1-20 | **Bug: "Details" em evento de security scan navega para a rota errada** — botão "Details →" em eventos do tipo CVE/security scan no Intelligence Hub navega para `/clusters/{id}/security`, que renderiza o resource browser de ServiceAccounts (mesmo bug do P1-16). Fix: (a) corrigir a rota de destino para a tela de Security Intelligence do cluster; (b) depende da correção do P1-16. | ✅ Entregue 2026-05-26 — auto-resolvido pelo P1-16: `/clusters/:id/security` agora renderiza `SecurityInsightsPage`; painel de detalhe de sinal de segurança (`IntelligencePage.tsx:291`) já navegava para `security-intelligence` corretamente |

#### Claude Code — Qualidade de Código (identificados em code review 2026-05-18)

> **Score atual: 6.5/10** — fundação sólida (zero `any`, zero `console.log`, React Query correto), mas monolitos e tokens ignorados pelos próprios componentes.

| # | Item | Arquivo | Critério de aceite |
|---|---|---|---|
| P1-CR1 | Refatorar `SecurityPage.tsx`: 15+ `useState` → `useReducer` ou react-hook-form | `screens/SecurityPage.tsx:36` | Máximo 3 `useState` na tela |
| P1-CR2 | Refatorar `WorkloadDetailPage.tsx`: 22 `useState` | `screens/WorkloadDetailPage.tsx:35` | Estado agrupado em reducer ou form lib |
| P1-CR3 | Dividir `AppShell.tsx` (588 linhas) em `MainNav`, `ClusterNav`, `UserMenu` | `ui/AppShell.tsx` | ✅ Concluído: `AppShell.tsx` reduzido para ~181 linhas |
| P1-CR4 | Extrair `NodeCard.tsx` de `NodesPage.tsx` (cards inline em `.map()`) | `screens/NodesPage.tsx:175` | Componente `NodeCard` reutilizável em `/components/` |
| P1-CR5 | Extrair tipos nomeados de API: `DeploymentRow`, `ClusterEvent`, etc. | `lib/api/workloads.ts:82` | Sem `Record<string, unknown>` em retornos de API |
| P1-CR6 | Criar barrel export `/components/index.ts` | `components/` | Todos os imports de components usam `from "../components"` |
| P1-CR7 | Dividir `ResourcesPage.tsx` (556 linhas) em sub-componentes por modo | `screens/ResourcesPage.tsx` | ✅ Concluído: `ResourcesPage.tsx` reduzido para ~88 linhas |

#### Claude — Backend
| # | Item | Critério de aceite |
|---|---|---|
| P1-10 | SSO/SAML/OIDC: integração no auth-service | ✅ Entregue — endpoints `/auth/sso/...`, migration `000016_sso_configs`, fluxo OIDC completo |
| P1-11 | AI BYOK: OpenAI/Claude/Azure/Bedrock configurável por org | ✅ Entregue — endpoints `/auth/ai-providers/...`, migration `000017_ai_providers`, proxy no gateway |

### 🟡 P2 — Médio prazo

| # | Item | Responsável | Status |
|---|---|---|---|
| P2-1 | Segurança: scan de imagens via Trivy | Claude | ✅ Entregue — `ScanImageSecurity()`, `runTrivyImageScan()` |
| P2-2 | Segurança: K8s config risk via Polaris/Kubescape | Claude | ✅ Entregue — `GetSecurityConfigRisk()`, 15+ regras |
| P2-3 | Segurança: RBAC risk analysis | Claude | ✅ Entregue — `GetSecurityRBACRisk()`, análise de Roles/Bindings |
| P2-4 | Segurança: UI — Security tab no cluster, risk score por workload | Claude Code + Claude | ✅ Entregue (backend score API + UI em andamento) |
| P2-5 | Shell interativo de pod: xterm.js + PTY bidirecional | Claude Code | ✅ Entregue — `ShellPage.tsx` + `PodTerminal.tsx` com xterm.js |
| P2-6 | Topologia visual de cluster (graph de recursos) | Claude Code + Claude | ✅ Entregue (backend graph API + UI em andamento) |
| P2-7 | Token do agente persistido em banco (sobrevive restart) | Claude | ✅ Entregue — migration `000005_agent_tokens`, `PostgresAgentTokenRepository` |
| P2-8 | Tunnel exec/logs via agente | Claude | ✅ Entregue — handlers `/pods/:name/logs` e `/exec`, agent redirect |
| P2-9 | Nome do cluster no header (UUID → nome real) | Claude Code | ✅ Entregue (= P0-5) |
| P2-10 | CPU/Memory empty state quando metrics-server não está instalado | Claude Code | ✅ 2026-05-25 |
| P2-11 | Approval workflows UI: dupla aprovação para ações críticas | Claude Code + Claude | ✅ Entregue (backend approvals API + UI em andamento) |
| P2-12 | Site/Landing Page: hero, features, labs, pricing, architecture | Claude Code | ✅ Entregue — build limpo, dark theme, zero `href="#"`, screenshots reais. Verificação visual Playwright OK 2026-05-25. |
| P2-13 | Deploy empacotado para staging real | Claude | ✅ Entregue — staging runbook, `.env.staging.example`, Helm atualizado |
| **P2-14** | **Responsividade — suporte a diferentes tamanhos de tela e janela** | Claude Code | ✅ Entregue |
| **P2-15** | **Observability → Cluster Events global** — nova página `/observability/events` com eventos de todos os clusters agregados, filtros por cluster/severity/namespace/reason e janela de tempo. Substitui a visão por-cluster como entry point principal de eventos operacionais. | Claude Code + Claude | ✅ Entregue 2026-05-26 — `GlobalEventsPage.tsx` criada: picker de cluster (All/específico), filtros namespace/severity/reason/within-minutes, abas Correlated/Raw, tabelas com link para cluster individual; rota `/observability/events` em `router.tsx`; item "Cluster Events" no sidebar global |

#### P2-14: Responsividade — detalhamento

**Problema identificado:** o produto foi construído com foco em desktop wide (1280px+). Em telas menores, janelas redimensionadas ou monitores de menor resolução, o layout quebra ou fica inutilizável.

**O que cobrir:**

| Área | O que implementar |
|---|---|
| Sidebar | Colapsar automaticamente em telas < 1024px; hamburguer menu em mobile |
| Header | Métricas HEALTH/NODES/CPU/MEM colapsam em telas < 1280px; user menu adaptado |
| Tab bar do workspace | Scroll horizontal em telas menores |
| ResourceTable | Scroll horizontal, colunas adaptáveis (ocultar colunas secundárias em mobile) |
| Cards de workload | Stack vertical em < 768px em vez de grid |
| Painéis laterais (Inspector, AI Assistant) | Overlay em mobile em vez de side-by-side |
| Forms (login, signup, admin) | Padding e tamanhos adaptados para mobile |
| Topology / SVG | Viewport responsivo com zoom/pan em telas pequenas |

**Breakpoints alvo:**
- **Mobile** (320–768px): app utilizável em smartphone
- **Tablet** (768–1024px): layout adaptado, sidebar colapsada
- **Desktop** (1024–1280px): layout compacto mas completo
- **Wide** (1280px+): layout atual (referência) |
| ~~P2-14~~ | ~~GitHub org + repositórios públicos + imagens no registry~~ | ~~Claude~~ — **Deferido: sem conta GitHub ainda** |

> **Nota de governança:** toda entrega deve seguir `GOVERNANCE.md` — testes obrigatórios, rebuild do container, smoke test, commit. Items sem esses critérios não estão concluídos.

### 🔵 P3 — Fase Enterprise / Infra Avançada

| # | Item | Responsável | Fase |
|---|---|---|---|
| P3-1 | Runtime Security: Falco runtime events via API | Claude | ✅ Entregue — `GetRuntimeSecurityEvents()`, detecção de Falco em `falco-system` |
| P3-2 | Supply Chain: SBOM + Cosign verify | Claude | ✅ Entregue — `runCosignVerify()`, `runSBOM()` via trivy/syft |
| P3-3 | AI Security Insights: correlação CVE + config + RBAC + runtime | Claude + Claude Code | ✅ Entregue (backend) — `GetSecurityInsights()` agrega todos os módulos |
| P3-4 | Rate limiting distribuído via Redis | Claude | ✅ Entregue — `redisWindowLimiter`, `go-redis/v9`, fallback in-memory |
| P3-5 | KMS / envelope encryption para credenciais de cluster | Claude | ✅ Entregue — `EnvelopeCipher` AES-GCM, migration `credential_dek_ciphertext` |
| P3-6 | LDAP / Active Directory / SCIM (enterprise identity sync) | Claude | ✅ Entregue — `go-ldap/v3`, endpoints `/auth/ldap/...`, migration `000019_ldap_config` |
| P3-6b | TOTP 2FA + FIDO2/WebAuthn stub Enterprise | Claude | ✅ Entregue — `pquerna/otp/totp`, endpoints `/auth/2fa/totp/...`, migration `000018_totp` |
| P3-7 | Operational Labs v1: text-guided, 5 cenários, badges — **expurgado do Navyr** | Claude Code | ✅ Removido (rota, página e menu) |
| **P3-7v2** | **Lab Engine entregue** — API start/status/stop no orchestrator, verifier engine (4 tipos de condição), 12 Helm charts, migration `000007_lab_sessions`. E2E concluído com suporte agent-mode e validação real. | Claude | ✅ Entregue |
| P3-7b | **Community service entregue** — `community` no docker-compose `:8084`, migrations, `/community/badges/:username`, `/community/labs`, rate limiting em events | Claude | ✅ Entregue |

#### P3-7v2: Operational Labs v2 — detalhamento

**Princípio:** o usuário usa o próprio Navyr (Pods, Inspector, Logs, YAML editor, Shell, Security) para detectar e resolver falhas reais num cluster conectado. Sem passo-a-passo. Sem texto guiando. Cluster real, incidente real, ferramenta real.

**Fluxo de execução:**
```
Start Lab → helm install navyr-lab-<id> -n navyr-labs
          → cluster quebrado visível no workspace do Navyr
          → usuário diagnostica e corrige usando a ferramenta
          → Verifier polling → condições satisfeitas → badge + helm uninstall
```

**Requisito:** cluster próprio já conectado ao Navyr (direct mode ou agent mode).

**Helm charts a criar (`navyr-labs/charts/`):**

| Chart | Falha injetada | Condição de conclusão |
|---|---|---|
| `crashloop-env` | Deployment sem env var crítica → CrashLoopBackOff | Pod `Running` + 0 restarts em 60s |
| `oomkilled` | `limits.memory: 32Mi` + app que aloca 128Mi | Pod sem OOMKill no último ciclo |
| `image-pull-error` | Tag de imagem inexistente | Pod `Running` com imagem válida |
| `pending-no-resources` | `requests.cpu: 10` (impossível de agendar) | Pod `Running` |
| `failed-rollout` | Readiness probe apontando para endpoint errado | Deployment `Available` |
| `node-pressure` | DaemonSet que satura memória do node | Node sem `MemoryPressure` condition |
| `pvc-unbound` | PVC sem StorageClass disponível | Pod `Running` com volume montado |
| `privileged-container` | `securityContext.privileged: true` | Finding ausente no Security scan |
| `cluster-admin-sa` | ServiceAccount com ClusterRoleBinding `cluster-admin` | Binding removido ou substituído |
| `secret-in-env` | API key hardcoded em `env:` direto no deployment | `secretKeyRef` em uso |
| `no-network-policy` | Namespace sem isolamento de rede | NetworkPolicy aplicada |
| `rbac-escalation` | SA com permissão de ler secrets cross-namespace | Role com escopo mínimo correto |

**Backend — Lab API (Claude, orchestrator service):**
- `POST /clusters/:id/labs/:labId/start` — `helm install` no cluster via orchestrator
- `GET /clusters/:id/labs/:labId/status` — polling do Verifier (condições por lab)
- `DELETE /clusters/:id/labs/:labId` — `helm uninstall` + cleanup do namespace
- Verifier engine: condições declarativas por lab (pod phase, deployment conditions, YAML fields, RBAC state)

**Frontend — Lab UI v2 (Claude Code):**
- Substituir LabsPage atual: sem steps, sem wizard
- Tela do lab: briefing do incidente + objectives + live status checks (polling `/status`)
- Link direto para namespace `navyr-labs` no workspace do cluster
- AI Assistant disponível como hint (sem spoilers automáticos)
- Badge earned quando verifier confirma todas as condições

| # | Item | Responsável | Fase |
|---|---|---|---|
| P3-8 | HA mode + air-gapped deployment | Claude | ✅ Entregue — Helm `replicaCount: 2`, PDB, `imagePullSecrets` configurável |
| P3-9 | AWS / Azure / GCP marketplace | Claude | ❌ Fase 6 (Enterprise hardening) |
| **P3-10** | **Agent Hardening Fase 1 — prod-readiness** | Claude | ✅ Entregue — Helm chart, RBAC mínimo, probes, token TTL, bootstrap refatorado |
| **P3-11** | **Prompt Management System** — prompts desacoplados do código, gerenciáveis via UI | Codex | ✅ Entregue 2026-05-22 — 6 endpoints `/api/v1/prompts/*` + handler no orchestrator |
| **P3-12** | **Topology Modes** — Runtime Mode, Security Mode, AI Insights Mode sobre o mesmo grafo | Codex | ✅ Entregue 2026-05-22 — `TopologyGraphNode` com `risk_score`, `risk_level`, `ai_annotation`; handler valida `mode=runtime\|security\|ai` |
| **P3-13** | **Badges LinkedIn-compatible** — credenciais verificáveis com URL pública `labs.navyr.io/certificates/` | Codex | ✅ Entregue 2026-05-22 — `GET /community/certificates/{id}` no formato OpenBadges 2.0 |
| **P3-14** | **AI Cloud monetization** — modelo de créditos/AI packs para análises (Incident Analysis, RCA, Security Analysis) | Claude | ✅ Entregue 2026-05-26 — migration 000005: tabelas `ai_credit_packs`, `org_ai_credits`, `ai_credit_transactions`, `stripe_events`; 4 packs padrão (Starter 1k/$4.99 → Enterprise 100k/$249.99); endpoints billing: balance, debit, transactions, Stripe checkout; webhook HMAC-SHA256 com idempotência; gateway proxy `/api/v1/billing/ai/credits/*` e `/api/v1/billing/ai/packs`; frontend: credit balance card + BuyCreditsModal + transaction history na aba AI Usage |
| **P3-15** | **BYOK via cluster secret** — agente lê K8s Secret `navyr-ai-key` no cluster do cliente para chamadas de IA; UI Settings→AI Providers com provider ativo + Coming Soon cards (Vault, AWS SM, GCP SM, Azure KV, Doppler, CyberArk) com "Notify me" | Claude | ❌ Backlog P2 |
| **P3-16** | **Remoção do direct mode** — eliminar `connectivity_mode: direct` do backend (orchestrator, gateway, DB migrations), frontend (ClusterCreatePage, ClusterWorkspacePage), bootstrap script e documentação. Agent mode é o único modo suportado. Clusters direct existentes ficam inacessíveis. | Codex | ✅ Entregue 2026-05-22 — migration `000009_remove_direct_mode.up.sql`; colunas `credential_*` e `endpoint` removidas |
| **P3-19** | **Security Compliance Scoring + Attack Path** — score de compliance por workload (grade A–F) e grafo de attack path com CVE, exposição pública, NetworkPolicy, container privilegiado, RBAC | Codex | ✅ Entregue 2026-05-24 — `GET /clusters/{id}/security/compliance` e `GET /clusters/{id}/security/attack-path` |
| **P3-17** | **AIOps Core Engine (Phase 6)** — anomaly detection worker, RCA determinístico, remediações executáveis, risk scoring, endpoints de dashboard. Migrations 000015-000018. `GET /clusters/{id}/aiops/anomalies`, `GET /clusters/{id}/aiops/risk-score`, `GET /aiops/summary` | Codex | ✅ Entregue 2026-05-24 |
| **P3-18** | **AIOps Live Engine + Historical Health + Lab Tunnel Fix (Phase 7)** — worker de health history (15min, migration 000019), AIOps worker com dados reais via agent tunnel, Intelligence WS→AIOps bridge (anomalias como signals), Lab handler usa tunnel por cluster, composite score endpoint. `GET /clusters/{id}/health/history`, `GET /clusters/{id}/score` | Codex | ✅ Entregue 2026-05-24 |

#### P3-10: Agent Hardening Fase 1 — detalhamento

**Contexto:** o agente atual (`infra/k8s/agent/` + `bootstrap-agent.sh`) tem bloqueadores críticos de produção identificados em auditoria de 2026-05-20. O Helm chart atual da plataforma (P3-8) não cobre o agente — ele ainda usa kubectl raw.

**Deliverables:**

| # | Deliverable | Severidade |
|---|---|---|
| D1 | Rebrand `clusterone-*` → `navyr-*` em todos os manifests e scripts | Alto |
| D2 | Helm chart `infra/helm/navyr-agent/` com NetworkPolicy, PDB, probes, `values-kind.yaml` | Crítico |
| D3 | RBAC mínimo — secrets somente leitura no ClusterRole | Crítico |
| D4 | Token TTL 90 dias + endpoint `/agent/token/renew` + migration `agent_tokens` + auto-renovação no executor | Crítico |
| D5 | Health server porta 8090 no executor (`/healthz` + `/ready`) | Alto |
| D6 | Bootstrap refatorado — Helm, idempotente, `--kind`, `--dry-run`, `--image-tag` obrigatório | Médio |

**Critério de aceite:** `helm lint` limpo, ClusterRole sem write em secrets, probes funcionando, token com expiração, `go test ./...` passando em orchestrator e executor.

---

## 7.5 Débitos Técnicos de Segurança (Red Team 2026-05-25)

> Relatório completo: `docs/security-report-20260525.md`

| # | Finding | Severidade | Responsável | Status |
|---|---|---|---|---|
| SEC-01 | Prometheus `/metrics` endpoint sem autenticação — expõe telemetria interna (goroutines, request rates, GC stats) sem auth | Medium | Codex (gateway) | ✅ Entregue 2026-05-26 — `protectMetricsHandler` modificado: sem credenciais configuradas, permite apenas loopback (127.0.0.1/::1); com `METRICS_BASIC_AUTH_USER`+`PASS` exige basic auth |
| SEC-02 | Input validation ausente em nomes de cluster — aceita HTML/script tags sem validação (Go JSON escapa, mas risco de stored XSS em futuros paths) | Low | Codex (orchestrator) | ✅ Entregue 2026-05-26 — `cluster_handler.go`: `clusterNameRe` regex `^[a-zA-Z0-9][a-zA-Z0-9._-]{0,62}$` validado em `CreateCluster` antes de chamar o service |
| SEC-03 | CORS OPTIONS preflight retorna 401 para origins não-autorizadas (auth middleware roda antes do CORS) — informacional | Informational | Codex (gateway) | ✅ Entregue 2026-05-26 — `corsMiddleware`: OPTIONS para origin não-autorizada retorna 403 antes de atingir `authMiddleware` |

### Correções previstas

- **SEC-01 (gateway):** Adicionar IP allowlist ou basic auth em `/metrics`. Ou expor em porta separada via env var `METRICS_PORT`.
- **SEC-02 (orchestrator):** Validar `cluster.name` com regex `^[a-zA-Z0-9][a-zA-Z0-9._-]{0,62}$` no handler `CreateCluster`.
- **SEC-03 (gateway):** Mover `corsMiddleware` para antes do `authMiddleware` para `OPTIONS` requests.

---

## 8. Fases do Roadmap

| # | Fase | Foco | Status |
|---|---|---|---|
| 1 | **Foundation** | Frontend estável, componentização, branding Navyr, docs, Helm/Docker básico | ✅ Concluída |
| 2 | **OSS Core** | K8s real conectado, workloads, pods, nodes, events, logs, YAML, shell, métricas, BYOK AI básico | ✅ Concluída |
| 3 | **Premium Foundation** | Auth enterprise, RBAC avançado, audit, governance, AI Gateway, SSO, LDAP, TOTP, quotas | 🔄 80% — F3-19 (JIT LDAP) e F3-22 (SCIM) diferidos |
| 4 | **Operational Intelligence** | Intelligence Hub (P0-9), reestruturação de navegação, posicionamento como plataforma de inteligência | 🔴 Próximo |
| 5 | **Observabilidade** | Prometheus, Loki, Tempo, Alertmanager, Cilium/Hubble, correlation engine | ⬜ Planejado |
| 6 | **Cluster Health** | Control plane, certificados, webhooks, storage health, platform health scoring | ⬜ Planejado |
| 7 | **FinOps** | OpenCost, rightsizing, anomalias de custo, conversão de moeda | ⬜ Planejado |
| 8 | **Security DevSecOps** | Falco runtime events, CIS Benchmark, NIST, compliance frameworks | ⬜ Planejado |
| 9 | **Motor de IA** | Anomaly detection (Z-Score→EWMA→STL), RCA automatizado, incident timeline | 🔄 Em progresso — AIOps Core (P3-17) + Live Engine (P3-18) entregues; correlação avançada e EWMA/STL pendentes |
| 10 | **RBAC Architecture** | Dual-layer permissions, URL hierarchy org-scoped, Org Owner, Platform Admin | ⬜ Planejado |
| — | ~~Labs & Community~~ | Navyr Labs standalone, gamificação, leaderboard, community | ⏸ Fim da fila |
| — | ~~SaaS / Enterprise~~ | Hosted SaaS, marketplace, HA, air-gapped, support/SLA | ⏸ Fim da fila |

---

## 9. Referências visuais

Screenshots de design de referência: `kubeops/clusterone/`

Telas capturadas: Deployments, Pods, ReplicaSets, Services, Ingresses.

Gaps identificados vs. implementação atual:

| Gap | Status |
|---|---|
| GAP-001: Rotas todas iguais | ✅ Resolvido |
| GAP-002: Header sem métricas | ✅ Resolvido |
| GAP-003: Tab bar horizontal | ✅ Resolvido |
| GAP-004: Cards de Runtime Inventory | ✅ Resolvido (P1-2) |
| GAP-005: Breadcrumb completo | ✅ Resolvido (P1-3) |
| GAP-006: Active state sidebar | ✅ Resolvido |
| GAP-007: Namespace selector | ✅ Resolvido |
