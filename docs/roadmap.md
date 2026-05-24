# Product Roadmap

**Last updated: 2026-05-24**  
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
| P0-8 | Align UI to HTML prototypes | Visual source of truth: `frontend-proposals/01`–`10`. Screen-by-screen mapping in CLAUDE.md. | 🔄 Partial |
| P0-9 | Intelligence Hub + sidebar restructure | `IntelligencePage.tsx` (AI summary, stat tiles, cross-cluster signal feed). New sidebar: Intelligence first, Observability/Security global, Clusters second. Prototype: `11-intelligence.html`. | ❌ Not started |
| P0-4 | Full componentization | Remaining workload screens → `RuntimeCard`; `OperationalSummary` on every screen; hard-coded colors → `var(--navyr-*)`; inline `ProgressBar` → central component | 🔄 Partial |
| P0-7 | Dark theme across all cards | No `bg-white` / `bg-slate-50` visible in workspace screens | 🔄 Partial |

#### P0-8: UI prototype alignment — screen map

| HTML | Screen | Status |
|---|---|---|
| `01-overview.html` | `OverviewPage.tsx` | ❌ |
| `02-workspace.html` | `ClusterWorkspacePage.tsx` + `AppShell.tsx` | ❌ |
| `03-workload-panel.html` | `WorkloadsPage.tsx` | ✅ |
| `04-pod-inspector.html` | `WorkloadDetailPage.tsx` + `InspectorPanel.tsx` | ✅ |
| `05-topology.html` | `TopologyPage.tsx` | 🔄 Partial |
| `06-security.html` | `SecurityPage.tsx` + `SecurityInsightsPage.tsx` | 🔄 Partial |
| `07-observability.html` | `ObservabilityPage.tsx` | 🔄 Partial |
| `08-signup.html` | `SignupPage.tsx` | ✅ |
| `09-select-org.html` | `ClustersPage.tsx` | ✅ |
| `10-deployments.html` | `WorkloadsPage.tsx` (deployments) | ✅ |
| `11-intelligence.html` | `IntelligencePage.tsx` *(new)* | ❌ |

---

### P1 — Next sprint (frontend, Claude Code)

| # | Item | Detail |
|---|---|---|
| P1-2 | Runtime Inventory cards | `RuntimeCard` + `OperationalSummary` aligned to reference screenshots |
| P1-3 | Full breadcrumb | name + env + provider + region + K8s version |
| P1-6 | Security screen | `RiskBadge` + risk score per workload — operational visual, not a static table |
| P1-7 | Nodes heat map | CPU/mem/disk pressure heat map per node |
| P1-8 | Automation visual state | Execution state + run history |
| P1-9 | Empty states | Designed empty states on all screens |
| P1-10 | Admin UI | Groups, members, grants CRUD using existing endpoints |
| P1-12 | User menu | Editable profile, plan indicator |
| P1-13 | Status color semantics | Standardize across all screens using `lib/status.ts` + `StatusBadge` |
| P1-CR2 | Refactor `WorkloadDetailPage.tsx` | 22 `useState` → reducer (grouped state) |
| P1-CR4 | Extract `NodeCard.tsx` | From inline `.map()` in `NodesPage.tsx` |
| P1-CR5 | Named API types | `DeploymentRow`, `ClusterEvent`, etc. — no `Record<string, unknown>` in API returns |
| P1-CR7 | Split `ResourcesPage.tsx` (556 lines) | Sub-components per resource mode |

---

### P2 — Medium term

| # | Item | Owner | Status |
|---|---|---|---|
| P2-10 | CPU/Memory empty state when metrics-server not installed | Claude Code | ❌ |
| P2-11 | Approvals UI integration with dual-approval backend | Claude Code | 🔄 Partial |
| F3-19 | JIT LDAP provisioning — auto-create user on first LDAP login | Claude Code + Codex | ❌ |
| D1 | Historical health score — 15-min worker + `metrics/history` endpoint | Codex | ❌ |
| D6 | Lab service via agent tunnel — smoke test `POST /labs/crashloop-env/start` with tunnel | Codex | ❌ |

---

## Codex next brief — D1–D7

| # | Deliverable | Origin |
|---|---|---|
| D1 | Historical health score — migration + 15-min worker + `GET /clusters/{id}/metrics/history` | Brief 22b |
| D2 | Lab service agent tunnel validation — smoke test per-cluster lab ops via tunnel | Bug fix |
| D3 | Kind Provisioner service — create/destroy kind clusters per session | navyr-labs |
| D4 | Scenario Engine — multi-fault scenarios in declarative YAML (4 scenarios minimum) | navyr-labs |
| D5 | Timer + Scoring engine — real-time scoring with time penalties | navyr-labs |
| D6 | Leaderboard per scenario | navyr-labs |
| D7 | Certificates expanded — score + time + rank at completion | navyr-labs / D4 |

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
| Attack path visualization | 🗓 Planned |
| Predictive risk scoring (AI-assisted) | 🗓 Planned |

---

## Phase 5 — Intelligence platform (planned)

Five strategic epics, sequenced. Architectural decision (integration vs. new service) required before any backend work.

| Epic | Description | Status |
|---|---|---|
| E1 — Observability | Cross-cluster metrics, logs, traces, SLOs, Prometheus/Loki/Tempo integration | 🗓 Planned |
| E2 — Cluster Health | Historical health score, trending, degradation detection, proactive alerting | 🗓 Planned |
| E3 — Security intelligence | Attack path, runtime correlation, compliance scoring | Partial (Phase 4) |
| E4 — FinOps | Resource efficiency, cost estimation, right-sizing recommendations | 🗓 Planned |
| E5 — AI motor | Incident correlation, RCA, remediation suggestions, runbook generation | 🗓 Planned |

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
| Kind Provisioner (ephemeral clusters per session) | 🗓 Planned |
| Scenario Engine (multi-fault YAML) | 🗓 Planned |
| Timer + scoring | 🗓 Planned |
| Per-scenario leaderboard | 🗓 Planned |
| Certificates with score + time + rank | 🗓 Planned |

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
