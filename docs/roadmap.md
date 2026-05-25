# Product Roadmap

**Last updated: 2026-05-25 (Phase 8 ✅ | Phase 9 in progress — D0+D1+D2+SEC-01 delivered | P1-13 ✅)**  
**Source of truth:** this document supersedes all other roadmap references.

---

## Current status — what is working today

As of 2026-05-24 the full backend stack is production-ready and deployed. The frontend covers all primary screens with real API integration.

| Area | Status |
|---|---|
| Auth: registration, login, session, RBAC, invites, password reset | ✅ |
| SMTP email (invites + reset, fail-open, BYOK-ready) | ✅ |
| Billing: plans, enforcement, history, audit | ✅ |
| Orchestrator: full K8s operations (pods, deployments, nodes, storage, RBAC, exec) | ✅ |
| Gateway: JWT, RBAC, rate limiting, audit, critical action gate | ✅ |
| Security headers, CORS, HSTS, ASVS checklist, SAST/secret scanning | ✅ |
| Agent tunnel WebSocket (clusters without public IP — kind, private EKS) | ✅ |
| Multi-cluster (agent mode) | ✅ |
| User groups + grants per cluster/namespace + JWT scope impact | ✅ |
| SSO/OIDC (OIDC discovery, 4 endpoints, migration 000016) | ✅ |
| AI BYOK (OpenAI, Claude, Azure, Bedrock, Vertex, Ollama — per org) | ✅ |
| TOTP 2FA (setup, verify, disable, backup codes) | ✅ |
| LDAP config, test, group sync, JIT provisioning stub | ✅ |
| Webhooks persistence (PostgreSQL, outbound dispatch) | ✅ |
| AI quota enforcement per plan (100K / 500K / 2M / unlimited tokens/month) | ✅ |
| Approval workflows API (dual-approval for critical actions) | ✅ |
| Audit log export (JSON/CSV/PDF), retention policy, purge by date | ✅ |
| Security scanning: Trivy image scan, config risk, RBAC risk, security insights | ✅ |
| Topology graph API (runtime / security / AI modes) | ✅ |
| Operational Labs: 12 fault scenarios, verifier engine, badge on pass | ✅ |
| Community: badges, leaderboard, GitHub OAuth, LinkedIn certificates | ✅ |
| AIOps Core Engine: anomaly detection worker, RCA, remediations, risk scoring (migrations 000015-000018) | ✅ |
| AIOps Live Engine: real cluster data via agent tunnel, health history worker, Intelligence WS bridge, composite score (migration 000019) | ✅ |
| Frontend: all primary screens with real API integration | ✅ |
| Docker Compose, Helm chart, Kustomize overlays, CI pipeline (multi-arch GHCR images) | ✅ |
| navyr.io landing page (Next.js, Raspberry Pi + Cloudflare tunnel) | ✅ |

---

## Phase overview

| Phase | Name | Status |
|---|---|---|
| Phase 1 | Core platform (auth, gateway, orchestrator, billing, frontend shell) | ✅ Complete |
| Phase 2 | Kubernetes coverage (full workload CRUD, shell, topology, security scanning, agent tunnel) | ✅ Complete |
| Phase 3 | Premium foundation (SSO/OIDC, LDAP, TOTP, groups/grants, AI BYOK, webhooks, labs) | ✅ Complete |
| Phase 4 | Security module (runtime security, SBOM, Cosign, policy enforcement, compliance) | 🔄 In progress |
| Phase 5 | Intelligence platform (observability, cluster health, FinOps, AI motor) | 🗓 Planned |
| Phase 6 | Enterprise hardening (HA, air-gapped, KMS, SCIM, advanced governance) | 🗓 Planned |
| Phase 7 | Labs & community (kind provisioner, scenario engine, scoring — site feature, not core system) | 🗓 Planned |

---

## Active backlog

### P0 — Current sprint (frontend, Claude Code)

| # | Item | Detail | Status |
|---|---|---|---|
| P0-8 | Align UI to HTML prototypes | Visual source of truth: `frontend-proposals/01`–`11`. 11/11 HTMLs aligned. | ✅ 2026-05-25 |
| P0-9 | Intelligence Hub + sidebar restructure | `IntelligencePage.tsx` — AI Summary narrative, stat tiles, signal feed, cluster health strip. | ✅ 2026-05-25 |
| FinOps UI | FinOps page with cluster efficiency scoring | `FinOpsPage.tsx` — per-cluster efficiency rings (A–F), idle workloads, savings opportunities, top consumers. | ✅ 2026-05-25 |
| AIOps UI | AIOps anomaly analysis page | `AIOpsPage.tsx` — org summary, cluster risk scores, severity-sorted anomaly feed, inline RCA + remediations. | ✅ 2026-05-25 |
| Security Compliance + Attack Path | SecurityIntelligencePage enhancements | Attack Path tab: risk nodes (blast radius, risk factors), attack vectors, summary stats. Compliance tab: grade A–F, per-workload breakdown (already shipped). | ✅ 2026-05-25 |
| P0-4 | Full componentization | Remaining workload screens → `RuntimeCard`; `OperationalSummary` on every screen; hard-coded colors → `var(--navyr-*)` | ✅ 2026-05-25 |
| P0-7 | Dark theme across all cards | No `bg-white` / `bg-slate-50` visible in workspace screens — all screens use design tokens | ✅ 2026-05-25 |

#### P0-8: UI prototype alignment — screen map

| HTML | Screen | Status |
|---|---|---|
| `01-overview.html` | `OverviewPage.tsx` | ✅ 2026-05-25 |
| `02-workspace.html` | `ClusterWorkspacePage.tsx` + colored Jump To icons | ✅ 2026-05-25 |
| `03-workload-panel.html` | `WorkloadsPage.tsx` | ✅ |
| `04-pod-inspector.html` | `WorkloadDetailPage.tsx` + `InspectorPanel.tsx` | ✅ |
| `05-topology.html` | `TopologyPage.tsx` | ✅ 2026-05-25 |
| `06-security.html` | `SecurityIntelligencePage.tsx` — ScoreRing + tabs (config/rbac/images/runtime/compliance/attack-path) | ✅ 2026-05-25 |
| `07-observability.html` | `ObservabilityPage.tsx` — drill-down org→cluster→namespace→workload + integration badges | ✅ 2026-05-25 |
| `08-signup.html` | `SignupPage.tsx` | ✅ |
| `09-select-org.html` | `ClustersPage.tsx` | ✅ |
| `10-deployments.html` | `WorkloadsPage.tsx` (deployments) | ✅ |
| `11-intelligence.html` | `IntelligencePage.tsx` | ✅ |

---

### P1 — Next sprint (frontend, Claude Code)

| # | Item | Detail | Status |
|---|---|---|---|
| P1-1 | Tab bar in workspace | Horizontal tab bar in `ClusterWorkspacePage` | ✅ |
| P1-2 | Runtime Inventory cards | `RuntimeCard` + `OperationalSummary` in all workload screens | ✅ |
| P1-3 | Full breadcrumb | name + env + provider + region + K8s version | ✅ |
| P1-5 | Billing redesign | Dark hierarchy with `MetricCard`, plan tier strip, ROI tiles | ✅ |
| P1-6 | Security screen | `RiskBadge` + risk score per workload — operational visual | 🔄 In progress |
| P1-7 | Nodes heat map | CPU/mem/disk pressure heat map per node | ✅ |
| P1-8 | Automation visual state | Execution state + run history | ✅ |
| P1-9 | Empty states | Designed empty states on all screens | ✅ |
| P1-10 | Admin UI | Groups, members, grants CRUD — LDAP groups, sync, member management | ✅ |
| P1-11 | Jobs & CronJobs | Correct columns + inspector | ✅ |
| P1-12 | User menu | Editable profile, plan indicator | ✅ |
| P1-13 | Status color semantics | Standardize across all screens using `lib/status.ts` + CSS design tokens | ✅ 2026-05-25 |
| P1-14 | Audit log critical events | Critical events highlighted with `var(--navyr-critical)` | ✅ |
| P1-15 | Audit log time filter | Time range filter on audit log | ✅ |
| P1-CR2 | Refactor `WorkloadDetailPage.tsx` | ✅ 22 `useState` → reducer (grouped state) | ✅ |
| P1-CR3 | Split `AppShell.tsx` | `AppShell.tsx` → ~181 lines, `MainNav`/`ClusterNav`/`UserMenu` | ✅ |
| P1-CR4 | Extract `NodeCard.tsx` | Standalone component in `components/` | ✅ |
| P1-CR5 | Named API types | `DeploymentRow`, `ClusterEvent` in `lib/api/workloads.ts` | ✅ |
| P1-CR7 | Split `ResourcesPage.tsx` (556 lines) | Reduced to ~88 lines with sub-components | ✅ |

---

### P2 — Medium term

| # | Item | Owner | Status |
|---|---|---|---|
| P2-10 | CPU/Memory empty state when metrics-server not installed | Claude Code | ✅ 2026-05-25 |
| P2-11 | Approvals UI — nav + route + gateway fix (Phase 9 D0) | Claude Code + Codex | ✅ 2026-05-25 |
| F3-19 | JIT LDAP provisioning — auto-create user on first LDAP login | Codex | ✅ Phase 8 D5 |

---

## Phase 4 — Security module

| Item | Status |
|---|---|
| Trivy image scan | ✅ |
| K8s config risk (Polaris/Kubescape rules) | ✅ |
| RBAC risk analysis | ✅ |
| Security insights (CVE + config + RBAC + runtime correlation) | ✅ |
| Runtime events (Falco integration, detection fallback) | ✅ |
| Cosign image verification | ✅ |
| SBOM generation (syft/trivy fallback) | ✅ |
| Runtime security (eBPF, Tetragon) | 🗓 Planned |
| Policy enforcement (Kyverno / OPA Gatekeeper) | 🗓 Planned |
| Compliance reports (CIS Benchmark, SOC 2, PCI-DSS) | 🗓 Planned |
| Attack path visualization (`GET /clusters/{id}/security/attack-path`) | ✅ |
| Predictive risk scoring (AIOps risk score per cluster) | ✅ |
| Security compliance scoring per workload (`GET /clusters/{id}/security/compliance`) | ✅ |

---

## Phase 5 — Intelligence platform

Five strategic epics. Backend foundation delivered; frontend integration ongoing.

| Epic | Description | Status |
|---|---|---|
| E1 — Observability | Cross-cluster metrics, logs, traces, SLOs, Prometheus/Loki proxy via agent tunnel | 🔄 In progress (Phase 9 D1+D9) |
| E2 — Cluster Health | Historical health score (15-min worker, 30d retention), composite score per cluster | ✅ Backend complete |
| E3 — Security intelligence | Attack path, runtime correlation, compliance scoring | ✅ Backend complete |
| E4 — FinOps | Resource efficiency per cluster + cross-cluster summary, top consumers, savings opportunities | ✅ Backend complete (stub) |
| E5 — AI motor | Anomaly detection, RCA engine, remediation generator, risk scoring, Intelligence WS stream | ✅ Backend complete |

---

## Phase 6 — Enterprise hardening (planned)

| Item | Notes |
|---|---|
| HA mode (multi-instance orchestrator with sticky WebSocket routing) | Requires message broker or sticky LB |
| Air-gapped deployment (no outbound internet required) | Helm chart + private registry support in place |
| AWS KMS credential encryption (KEK rotation) | Backend prepared (envelope encryption local), KMS integration deferred |
| SCIM v2 provisioning | Stub returns 501 — full implementation planned |
| Advanced governance (policy-as-code, workspace isolation) | 🗓 Planned |
| Azure AD / Okta / Keycloak connectors | 🗓 Planned |
| Executive dashboards + compliance PDF export | 🗓 Planned |

---

## Phase 9 — Observability Foundation + AIOps Correlation + Enterprise Hardening

**Status: 🔄 In progress (Codex)**

| Deliverable | Description | Status |
|---|---|---|
| D0 HOTFIX | Gateway `resolveFeature` — register `/api/v1/approvals/*` routes | ✅ 2026-05-25 |
| SEC-01 | Protect `/metrics` endpoint — basic auth via `METRICS_BASIC_AUTH_USER/PASS` env vars | ✅ 2026-05-25 |
| D1 | Prometheus query proxy via agent tunnel — `GET /clusters/{id}/observability/prometheus/query` | ✅ 2026-05-25 |
| D2 | Cross-cluster anomaly correlation engine — `correlated_anomalies` table + worker 5min | ✅ 2026-05-25 |
| SEC-02 | Input validation on `cluster.name` — regex `^[a-zA-Z0-9][a-zA-Z0-9._-]{0,62}$` | ❌ Pending |
| SEC-03 | CORS middleware before auth for OPTIONS requests | ❌ Pending |
| D3 | Capacity forecasting — linear regression 7d health_history | ❌ Pending |
| D4 | LDAP group-to-role mapping — `ldap_group_role_mappings` table + JIT auto-assign | ❌ Pending |
| D5 | Worker leader election — `worker_locks` table + TryAcquireLock/Renew/Release | ❌ Pending |
| D6 | Health score export JSON/CSV | ❌ Pending |
| D7 | AlertManager webhook ingest | ❌ Pending |
| D8 | Webhook delivery history API | ❌ Pending |
| D9 | Loki log proxy via agent tunnel | ❌ Pending |
| D10 | Community weekly challenges | ❌ Pending |

---

## Phase 7 — Labs & community (site feature)

> **Scope note:** Operational Labs are a feature of the navyr.io website and community experience — not part of the core platform. Labs run against the user's own cluster via the agent tunnel; the lab engine backend (verifier, Helm charts) lives in `navyr-orchestrator` as a support service, but the product surface (UI, scoring, leaderboard, certificates) belongs to the site.

| Item | Status |
|---|---|
| 12 operational fault scenarios (Helm charts) | ✅ |
| Verifier engine (poll every 10s, TTL 60min) | ✅ |
| Lab lifecycle: pending → running → passed / failed / stopped | ✅ |
| Community badges (GitHub identity, leaderboard) | ✅ |
| LinkedIn certificate (OpenBadges 2.0) | ✅ |
| Lab UI v2 (catalog, active lab, AI hint, badge earned) | ✅ |
| Kind Provisioner (ephemeral clusters per session, `POST /api/v1/labs/clusters`) | ✅ |
| Scenario Engine (4 fault scenarios: crashloop-env, oom-killer, pending-pods, rbac-lockout) | ✅ |
| Timer + scoring (formula: 1000 − elapsed_min×10 − hints×50) | ✅ |
| Per-scenario leaderboard (`lab_leaderboard` table, ranking endpoint) | ✅ |
| Certificates with score + time + rank | ✅ |

---

## Open operational items

| Item | Detail |
|---|---|
| GHCR_PAT as GitHub secret | Add PAT with `read:packages` to `github.com/navyr-io/site/settings/secrets/actions` |
| CI deploy workflow | Must use `sudo docker compose` instead of `docker run` |
| `NEXT_PUBLIC_APP_URL` / `NEXT_PUBLIC_COMMUNITY_URL` | Configure as secrets in navyr-io/site |
| Revoke old PAT | Revoke at `github.com/settings/tokens` |
| kind cluster unreachable | Re-register cluster using navyr-agent in agent mode (direct mode removed in migration 000009) |

---

## Recent deliveries — Phase 8: Reliability, Enterprise Identity & Intelligence Expansion (2026-05-25)

| Delivery | Description |
|---|---|
| Intelligence summary cache | In-memory cache per org, TTL 45s, invalidated on new anomaly. Reused by WS stream. |
| AI Gateway fallback chain | Model routing with priority ordering. Tries providers in sequence, returns structured error on all fail. Auth migration 000022. |
| Approval email notifications | Internal endpoint enqueues approval request/outcome emails to auth outbox. |
| Webhook delivery queue | `webhook_delivery_attempts` table (auth migration 000023). Worker with exponential backoff retry (3 attempts). |
| JIT LDAP provisioning | Auto-create user on first valid LDAP login. `users.source` column + `ldap_configs.jit_provisioning` flag. Auth migration 000024. |
| SCIM v2 core | Users + Groups provisioning (real implementation). `ServiceProviderConfig` and `Schemas` endpoints. `scim_handler.go`. |
| Cert expiry detection | `cert_expiry` anomaly kind. Detects TLS certs expiring within 30 days via cluster secret inspection. |
| BYOK via cluster secret | Agent reads K8s Secret `navyr-ai-key` from cluster. Source type `cluster_secret` exposed to gateway. |
| Observability namespace enrichment | Namespaces endpoint now includes `cpu_requests`, `memory_requests`, and `top_consumers` per namespace via `observability_hierarchy.go`. |
| Runbook generation engine | Deterministic runbook generation by `anomaly_kind`. `GET /clusters/{id}/aiops/anomalies/{id}/runbook`. `aiops_runbook.go`. |

---

## Phase 9 backlog (in progress — Codex)

| # | Item | Service | Status |
|---|---|---|---|
| D1 | Prometheus query proxy via agent tunnel | orchestrator | 🔄 |
| D2 | Cross-cluster anomaly correlation engine | orchestrator | 🔄 |
| D3 | Capacity forecasting endpoint (7d linear regression) | orchestrator | 🔄 |
| D4 | LDAP group-to-role mapping | auth | 🔄 |
| D5 | Worker leader election (HA-safe locks) | orchestrator | 🔄 |
| D6 | Health score export (JSON/CSV) | orchestrator | 🔄 |
| D7 | AlertManager webhook ingest | orchestrator | 🔄 |
| D8 | Webhook delivery history API | auth | 🔄 |
| D9 | Loki log proxy via agent tunnel | orchestrator | 🔄 |
| D10 | Community weekly challenges | community | 🔄 |

---

## Recent deliveries — Phase 5 / 6 / 7 (2026-05-24)

| Delivery | Description |
|---|---|
| Labs Kind Provisioner | `POST /api/v1/labs/clusters` — provision ephemeral kind cluster per session. `lab_clusters` table (migration 000012), GC worker destroys expired clusters. |
| Scenario Engine | 4 embedded fault scenarios (crashloop-env, oom-killer, pending-pods, rbac-lockout). `lab_sessions_v2` table (migration 000013). Inject → verifier loop → pass/fail. |
| Timer + Scoring + Leaderboard | Score formula, `lab_leaderboard` upsert on pass (migration 000014), ranking endpoint. |
| FinOps cross-cluster | `GET /api/v1/finops/summary` — aggregate efficiency score, top consumers, idle workloads, savings opportunities across all ready clusters. |
| Intelligence WebSocket | `WS /api/v1/intelligence/stream` — snapshot on connect, incremental signal push, heartbeat, 50-conn/org cap, inactivity close. Gateway proxy with session validation. |
| AIOps Core Engine | Anomaly detection worker (2-min interval, 5 anomaly kinds). RCA engine (deterministic rule-based, cached). Remediation generator with apply endpoint. Predictive risk scoring (0–100 formula). AIOps dashboard endpoints. Migrations 000015–000018. |
| AIOps Live Engine | Health history worker (15-min, 30d retention, migration 000019). AIOps worker reads real cluster data via agent tunnel. Intelligence WS bridge: active anomalies emitted as signals. Lab handler uses per-cluster tunnel. Composite score `GET /clusters/{id}/score`. |
| Security compliance scoring | `GET /clusters/{id}/security/compliance` — per-workload runAsNonRoot, readOnlyRootFilesystem, allowPrivilegeEscalation, resource limits, pinned tag, non-privileged. Grade A–F. |
| Security attack path | `GET /clusters/{id}/security/attack-path` — nodes + edges for CVE, public exposure, missing NetworkPolicy, privileged container, ClusterRole binding. |

---

## Recent deliveries — Phase 3 (2026-05-24)

| Delivery | Description |
|---|---|
| F3-13 LDAP group type | `group_type local\|ldap` + `ldap_group_dn`. Migration 000020. AdminPage toggle + UI. |
| F3-14 Sync Now | LDAP panel: Sync Now button + tiles (groups synced / members added / members removed). Idempotent. |
| F3-15 Billing redesign | PLANS array + UsageBar + plan tier strip (4 cards) + ROI tiles. Dark theme. |
| F3-16 Profile plan indicator | ProfilePage: plan badge + Upgrade button for non-enterprise plans. |
| F3-17 Edition gate SSO | SettingsPage SSO tab: "Pro Feature" gate card with Upgrade button; LDAP open to all plans. |
| F3-18 Webhooks persistence | `auth_webhooks` PostgreSQL table (migration 000021). Removes in-memory map+mutex. Pool dispatch. |
| F3-20 AI quota enforcement | `AITokenLimitPerMonth()` per plan. Billing summary includes `quota_exceeded`. Gateway returns 402 when exceeded. Frontend: adaptive color quota bar. |
| F3-21 Prompt Management UI | PromptsTab in SettingsPage — full CRUD for AI prompts. |
| Org-select bypass fix | `router.tsx`: guard block locks all routes to `/organization` when token exists but org is empty. |
| Single-org auto-enter | `OrganizationSelectPage.tsx`: auto-enters when profile returns exactly one org. |
| navyr-docs repository | Created at `github.com/navyr-io/navyr-docs` — full technical documentation (10 docs). |
| Architecture diagram | Integrated Mermaid diagram showing all components and every integration point in one view. |
| Monorepo → individual repos | 9 repos under `navyr-io` org. Multi-arch GHCR images (linux/amd64 + linux/arm64). |
