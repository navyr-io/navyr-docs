# Architecture overview

**Last updated: 2026-05-24**

## Integrated architecture diagram

The diagram below shows all Navyr components and every integration point between them — services, storage, in-cluster infrastructure, and external dependencies.

```mermaid
graph TB
    %% ── External clients ──────────────────────────────────────
    subgraph CLIENTS["External clients"]
        BROWSER["Browser / API clients"]
    end

    %% ── Presentation ──────────────────────────────────────────
    subgraph PRESENTATION["Presentation"]
        FE["navyr-frontend :5173\nReact · TypeScript · Vite\nghcr.io/navyr-io/navyr-frontend"]
    end

    %% ── Gateway ───────────────────────────────────────────────
    subgraph GATEWAY_SG["API Gateway  (only public entry point)"]
        GW["navyr-gateway :8080\nJWT validation · RBAC enforcement\nPlan enforcement · Rate limiting\nAudit emission · AI BYOK proxy\nghcr.io/navyr-io/navyr-gateway"]
    end

    %% ── Core services ─────────────────────────────────────────
    subgraph CORE["Core services  (internal network only)"]
        AUTH["navyr-auth :8081\nIdentity · JWT issuance\nSSO/OIDC · LDAP sync\nTOTP 2FA · SCIM stub\nGroups · Grants · Webhooks\nBYOK AI providers\n21 migrations"]
        BILL["navyr-billing :8082\nPlan enforcement\nUsage tracking\nAudit log + export\nRetention policy\n4 migrations"]
        ORCH["navyr-orchestrator :8083\nKubernetes CRUD\nPod exec · Node ops\nSecurity scanning\nTopology graph\nApproval workflows\nLab Engine · AIOps\nAutomation · Observability\nAgent tunnel registry\n8 migrations"]
        COMM["navyr-community :8084\nBadge engine\nLab completions\nLeaderboard\nGitHub OAuth\nAdmin panel\n2 migrations"]
    end

    %% ── Storage ───────────────────────────────────────────────
    subgraph STORAGE["Storage"]
        DB[("PostgreSQL :5432\n35+ tables · 35 migrations\n4 independent schemas")]
        REDIS[("Redis :6379\nRate limit state\noptional")]
    end

    %% ── In-cluster (customer) ─────────────────────────────────
    subgraph INCLUSTER["Customer Kubernetes cluster  (private VPC / NAT)"]
        AGT["navyr-agent\nWebSocket tunnel client\nHealth :8090\nghcr.io/navyr-io/navyr-agent"]
        K8SAPI["kube-apiserver\nKubernetes API"]
        HELMRT["Helm 3\nfault lab charts"]
    end

    %% ── External services ─────────────────────────────────────
    subgraph EXTERNAL["External services"]
        SMTP["SMTP server\ninvites · password reset"]
        LDAP["LDAP / Active Directory\ngroup sync"]
        GITHUB["GitHub OAuth\ncommunity identity"]
        AIPROVIDERS["AI providers\nOpenAI · Anthropic · Azure\nBedrock · Vertex · Ollama"]
        KMS["AWS KMS\ncredential KEK encryption\noptional"]
    end

    %% ── Client → frontend → gateway ──────────────────────────
    BROWSER -->|"HTTPS"| FE
    FE -->|"REST + WebSocket\nAuthorization: Bearer JWT"| GW

    %% ── Gateway → core services ───────────────────────────────
    GW -->|"POST /auth/token/validate\n/auth/* proxy"| AUTH
    GW -->|"POST /enforcement/check\nPOST /audit/events\n/api/v1/billing/* proxy"| BILL
    GW -->|"/api/v1/* K8s + labs + aiops\nX-Internal-Context header"| ORCH
    GW -->|"/community/* proxy"| COMM
    GW <-->|"rate limit counters"| REDIS

    %% ── Gateway → AI (BYOK completion proxy) ─────────────────
    GW -->|"POST /api/v1/ai/complete\n(resolves provider via auth)"| AIPROVIDERS

    %% ── Auth dependencies ─────────────────────────────────────
    AUTH -->|"identity · sessions\ngroups · grants\nSSO · webhooks"| DB
    AUTH -->|"invites\npassword reset"| SMTP
    AUTH <-->|"group sync\nuser lookup"| LDAP
    AUTH -->|"resolve BYOK config\nfor gateway AI proxy"| AIPROVIDERS

    %% ── Billing dependencies ──────────────────────────────────
    BILL -->|"plans · usage events\naudit log"| DB

    %% ── Orchestrator dependencies ─────────────────────────────
    ORCH -->|"cluster registry\nlab sessions\napprovals\nexec audit"| DB
    ORCH <-->|"WebSocket tunnel\nK8s ops relay\n(outbound from cluster)"| AGT
    ORCH -->|"badge grant\non lab pass"| COMM
    ORCH -->|"DEK envelope encryption\nKEK rotation"| KMS

    %% ── Community dependencies ────────────────────────────────
    COMM -->|"badges · completions\nleaderboard · events"| DB
    COMM <-->|"OAuth login + callback"| GITHUB

    %% ── In-cluster ────────────────────────────────────────────
    AGT -->|"in-cluster ServiceAccount\nall K8s API calls"| K8SAPI
    AGT -->|"helm install/uninstall\nlab fault scenarios"| HELMRT
```

---

## System map

```
┌─────────────────────────────────────────────────────────────┐
│                        Browser / API clients                │
└───────────────────────────┬─────────────────────────────────┘
                            │ HTTP / WebSocket
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                    navyr-gateway :8080                      │
│  JWT validate · RBAC enforce · plan enforce · audit emit   │
│  Rate limit (in-memory or Redis) · proxy + AI complete     │
└──────┬──────────┬──────────┬──────────┬────────────────────┘
       │          │          │          │
       ▼          ▼          ▼          ▼
  navyr-auth  navyr-billing  navyr-  navyr-community
   :8081       :8082        orchestrator  :8084
                             :8083
       │          │          │          │
       └──────────┴──────────┴──────────┘
                            │
                     PostgreSQL :5432
                    (shared database,
                     separate schemas
                     per service via
                     migration prefix)

                            │ WebSocket tunnel (outbound from cluster)
                            ▼
              ┌─────────────────────────┐
              │      navyr-agent        │
              │  (in customer cluster)  │
              └────────────┬────────────┘
                           │ in-cluster ServiceAccount
                           ▼
                    Kubernetes API server
```

## Request lifecycle

Every request from the browser goes through this sequence:

```
1. Browser → navyr-gateway
   - Attaches JWT as Authorization: Bearer <token>

2. Gateway → navyr-auth /auth/token/validate
   - Validates JWT signature and expiry
   - Returns org_id, user_id, role, permissions

3. Gateway → navyr-billing /billing/v1/enforcement/check
   - Checks if the org's plan allows this feature/action
   - Returns allow / deny / degraded

4. Gateway → downstream service (auth / orchestrator / billing / community)
   - Attaches X-Internal-Context (signed HMAC-SHA256 header)
   - Contains: org_id, user_id, role, cluster_id, permissions

5. Downstream service processes request
   - Reads X-Internal-Context instead of re-validating JWT
   - No service calls the Kubernetes API directly

6. Gateway → navyr-billing (async, non-blocking)
   - Emits audit event for write operations

7. Response flows back to browser
```

## Service responsibilities

| Service | Port | Primary role |
|---|---|---|
| `navyr-gateway` | `8080` | Entry point, auth enforcement, routing |
| `navyr-auth` | `8081` | Identity, JWT, RBAC, SSO, LDAP, 2FA, BYOK AI |
| `navyr-billing` | `8082` | Plan enforcement, usage tracking, audit log |
| `navyr-orchestrator` | `8083` | All Kubernetes operations via agent tunnel |
| `navyr-community` | `8084` | Labs, badges, GitHub identity, leaderboard |
| `navyr-frontend` | `5173` | React SPA, served via nginx in production |
| `navyr-agent` | outbound | In-cluster relay — connects to orchestrator |

## Network topology

```
Internet
    │
    ▼
navyr-gateway :8080          ← only service exposed externally
    │
Internal Docker/K8s network:
    ├── navyr-auth :8081
    ├── navyr-billing :8082
    ├── navyr-orchestrator :8083
    │       │
    │       │ WebSocket (outbound from cluster)
    │       ◄─── navyr-agent (in customer cluster)
    │                │
    │                └── Kubernetes API (in-cluster)
    └── navyr-community :8084
            │
            └── GitHub OAuth (external)

Shared:
    PostgreSQL :5432
    Redis :6379 (optional, rate limiting)
```

## Horizontal scaling considerations

| Service | Stateful? | Scale notes |
|---|---|---|
| `navyr-gateway` | No | Scales horizontally; Redis required for distributed rate limiting |
| `navyr-auth` | No | Scales horizontally; sticky sessions not required (JWT is stateless) |
| `navyr-billing` | No | Scales horizontally |
| `navyr-orchestrator` | Partially | Agent tunnel WebSocket connections are per-process; use sticky routing or a message broker for multi-instance |
| `navyr-community` | No | Scales horizontally |
| PostgreSQL | Yes | Primary + read replicas; use PgBouncer for connection pooling |

## Inter-service communication

All services communicate over plain HTTP/1.1 within the internal network. Authentication between services uses the `X-Internal-Context` header — a compact signed JSON payload that carries the authorized identity context from the gateway.

```
X-Internal-Context: <base64(json)>.<hmac-sha256-signature>

Payload fields:
  org_id      string   — organization UUID
  user_id     string   — user UUID
  role        string   — effective role (owner/admin/operator/viewer)
  cluster_id  string   — target cluster UUID (if scoped)
  permissions []string — allowed action list
  issued_at   int64    — Unix timestamp
```

Services **reject** requests without a valid `X-Internal-Context`. This prevents bypassing the gateway entirely.
