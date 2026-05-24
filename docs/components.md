# Components

**Last updated: 2026-05-24**

## navyr-gateway

**Repo:** https://github.com/navyr-io/navyr-gateway  
**Port:** `8080`  
**Language:** Go  
**Image:** `ghcr.io/navyr-io/navyr-gateway`

The single entry point for all client traffic. The gateway is responsible for authentication, authorization, plan enforcement, rate limiting, and routing. No client-facing service other than the gateway communicates with the outside world.

**Key behaviors:**
- Validates every JWT by calling `navyr-auth /auth/token/validate`
- Checks plan entitlements via `navyr-billing /billing/v1/enforcement/check` before proxying writes
- Attaches a signed `X-Internal-Context` header to every proxied request
- Emits audit events to `navyr-billing` for all write operations (async)
- Proxies AI BYOK completions after resolving the active provider config from `navyr-auth`
- Rate limits by org/IP — in-memory by default, Redis for multi-instance deployments
- Exposes Prometheus metrics on `GET /metrics`

**Dependencies:** navyr-auth, navyr-billing, navyr-orchestrator, navyr-community, Redis (optional)

---

## navyr-auth

**Repo:** https://github.com/navyr-io/navyr-auth  
**Port:** `8081`  
**Language:** Go  
**Image:** `ghcr.io/navyr-io/navyr-auth`  
**Database:** PostgreSQL — 21 migrations

The identity layer for the platform. Manages everything related to who can do what and under what conditions.

**Key behaviors:**
- Issues and validates JWTs (HS256, shared secret with gateway)
- Refresh token rotation with sliding window
- Organization registration (creates org + owner user in a single transaction)
- Full user lifecycle: invite, accept, role change, deactivation, session revocation
- RBAC with four roles (`owner`, `admin`, `operator`, `viewer`) plus fine-grained grants per cluster and namespace
- Groups: local groups and LDAP-synced groups with `ldap_group_dn`
- SSO via OIDC providers; SCIM v2 stub for directory provisioning
- TOTP 2FA with AES-256 encrypted secrets at rest
- LDAP/AD configuration, connectivity test, and group sync
- Outbound webhooks per organization (dispatched by orchestrator on cluster events)
- AI provider BYOK (Bring Your Own Key): stores and encrypts API keys for OpenAI, Anthropic, Azure, Bedrock, Vertex, Ollama

**Dependencies:** PostgreSQL, SMTP server (optional), LDAP/AD (optional)

---

## navyr-billing

**Repo:** https://github.com/navyr-io/navyr-billing  
**Port:** `8082`  
**Language:** Go  
**Image:** `ghcr.io/navyr-io/navyr-billing`  
**Database:** PostgreSQL — 4 migrations

The enforcement and observability layer for plan limits and audit requirements. Called by the gateway — never directly by clients.

**Key behaviors:**
- `POST /billing/v1/enforcement/check` — the gateway calls this before every proxied request; returns `allow`, `deny`, or `degraded` based on the org's plan and current usage counters
- Records every API call as a usage event (feature, org, timestamp, outcome)
- Records security-relevant actions as audit events (actor, resource, action, outcome, IP, user-agent)
- Configurable audit retention policy per org; supports on-demand purge and export (JSON/CSV)
- Exposes authenticated billing summary and usage history to org users via the gateway

**Dependencies:** PostgreSQL

---

## navyr-orchestrator

**Repo:** https://github.com/navyr-io/navyr-orchestrator  
**Port:** `8083`  
**Language:** Go  
**Image:** `ghcr.io/navyr-io/navyr-orchestrator`  
**Database:** PostgreSQL — 8 migrations  
**Bundled tools:** Trivy `0.70.0`, Helm `3.18.3`

The Kubernetes operations engine. All cluster-side operations go through the orchestrator, which relays them over the agent tunnel. The orchestrator never holds kubeconfig files or calls the Kubernetes API directly.

**Key behaviors:**
- Cluster registry: stores encrypted cluster credentials (AES-GCM, envelope encryption with DEK/KEK, optional AWS KMS)
- Agent tunnel: maintains WebSocket connections from in-cluster agents; multiplexes K8s API calls over the tunnel
- Full Kubernetes CRUD for all standard resource kinds
- Pod exec via single-use WebSocket tickets (prevents replay attacks); exec sessions are audited
- Node operations: cordon, uncordon, drain, taint management
- Security scanning: image CVEs (Trivy), config risk, RBAC risk, image signing (Cosign), SBOM, Falco runtime events, aggregated security insights
- Topology graph: builds a live graph of cluster resources and their relationships (owns, runs_on, selects, routes_to)
- Approval workflows: dual-approval gate for critical operations (delete, scale to 0, rollback)
- Operational Labs: installs Helm-based fault scenarios; runs a verifier loop to check resolution conditions
- Automation: runbooks (parameterized shell + kubectl), schedules, execution history
- AIOps: generates incident remediation plans using the org's AI provider; ingests operational signals; maintains cluster baselines
- Observability: SLO status, alert feeds, unified topology, correlated event streams
- Internal RBAC: reconcile, drift detection, snapshot, simulate endpoints for the gateway

**Dependencies:** PostgreSQL, navyr-auth (webhook dispatch), navyr-community (badge grants on lab completion), navyr-agent (cluster tunnel)

---

## navyr-community

**Repo:** https://github.com/navyr-io/navyr-community  
**Port:** `8084`  
**Language:** Go  
**Image:** `ghcr.io/navyr-io/navyr-community`  
**Database:** PostgreSQL — 2 migrations

The community and gamification layer. Operates with a separate identity (GitHub OAuth) from the main platform auth.

**Key behaviors:**
- GitHub OAuth login for a distinct community identity (not linked to org users)
- Issues achievement badges on lab completion — triggered by `navyr-orchestrator` after verifier passes
- Public badge profiles per user (accessible without auth)
- Rate-limited community event recording
- Leaderboard entries: tracks labs completed and badges earned per user
- Admin panel for feature toggles, secrets, metrics, and user management

**Dependencies:** PostgreSQL, GitHub OAuth app (optional)

---

## navyr-frontend

**Repo:** https://github.com/navyr-io/navyr-frontend  
**Language:** TypeScript / React 19  
**Image:** `ghcr.io/navyr-io/navyr-frontend`  
**Served on:** `:5173` (dev Vite) / `:8080` (production nginx)

The single-page application. Communicates exclusively with `navyr-gateway`. Holds a JWT in `localStorage` and attaches it to all API requests.

**Key behaviors:**
- Route guard: requires `token` + confirmed `organization` before rendering any protected page
- Auto-enter for single-org users: `OrganizationSelectPage` detects the user's org from the session profile and navigates automatically
- Cluster workspace: all cluster-scoped pages receive `X-Cluster-ID` via the API layer; the cluster ID is extracted from the route param
- Pod exec shell: opens a WebSocket to `/api/v1/pods/{name}/exec/ws` using a single-use ticket
- Real-time streams: SSE (`/api/v1/pods/watch`, `/api/v1/events/watch`) for live updates
- Dark-mode design system built entirely on CSS custom properties (`--navyr-*`); no Tailwind classes in production paths

**Dependencies:** navyr-gateway

---

## navyr-agent

**Repo:** https://github.com/navyr-io/navyr-agent  
**Language:** Go  
**Image:** `ghcr.io/navyr-io/navyr-agent`  
**Health:** HTTP `:8090` (`/healthz`, `/ready`)

The in-cluster relay. Runs inside the customer's Kubernetes cluster and maintains a persistent outbound WebSocket tunnel to `navyr-orchestrator`. This is the only component that actually calls the Kubernetes API.

**Key behaviors:**
- Establishes outbound WebSocket to orchestrator on startup; reconnects automatically on disconnect
- Authenticates with a JWT agent token (TTL 90 days, auto-renew)
- Forwards kubectl-equivalent operations (get, list, watch, exec, logs, apply, delete)
- Executes Helm install/uninstall for Operational Labs
- Sends heartbeats every 30s; orchestrator derives live cluster status from `LastAgentSeenAt`
- Minimal ClusterRole: full read/write on workload resources; read-only on secrets (values never forwarded)

**Deployed via:** `navyr-helm` Helm chart (`helm/navyr-agent/`)

**Dependencies:** Kubernetes API server (in-cluster), navyr-orchestrator (outbound WebSocket)

---

## navyr-helm

**Repo:** https://github.com/navyr-io/navyr-helm  
**Contents:** Helm charts for platform, agent, and Operational Labs; Kustomize overlays

Contains all Kubernetes packaging for production deployments:

| Path | Description |
|---|---|
| `navyr-platform/` | Full platform chart (all services + PostgreSQL) |
| `navyr-agent/` | In-cluster agent chart |
| `labs/` | 12 fault injection charts used by the Lab Engine |
| `k8s/` | Kustomize base + dev/staging/prod overlays |

**Dependencies:** None (pure Helm/Kustomize)

---

## navyr-deploy

**Repo:** https://github.com/navyr-io/navyr-deploy  
**Contents:** Docker Compose quick-start, `.env.example`, operational guidance

The fastest path to a running Navyr instance. Pulls all images from `ghcr.io/navyr-io/*` and starts them with a single `docker compose up -d`.

**Dependencies:** Docker Engine 24+, Docker Compose v2
