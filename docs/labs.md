# Operational Labs

**Last updated: 2026-05-24**

## What are Operational Labs

Operational Labs are hands-on failure scenarios that run inside ephemeral Kubernetes clusters provisioned on-demand by Navyr. Each lab injects a genuine fault condition — not a simulation — via a declarative YAML manifest applied by `navyr-orchestrator`. Users are expected to diagnose and resolve the fault using Navyr's tooling (or kubectl directly). When the resolution condition is met, the verifier marks the session as passed and issues a community badge.

Labs are designed to build operational muscle memory for the most common Kubernetes failure patterns.

## Architecture

```
User clicks "Start Lab" in Navyr UI
        │
        ▼
navyr-frontend → POST /api/v1/labs/clusters        (provision ephemeral Kind cluster)
        │
        ▼
navyr-orchestrator (Kind Provisioner)
  1. Runs: kind create cluster --name lab-<session-id>
  2. Inserts row in lab_clusters (status: creating)
  3. Polls until control-plane ready → status: ready
  4. Returns kubeconfig (encrypted, stored in DB)
        │
        ▼
navyr-frontend → POST /api/v1/labs/clusters/{id}/scenarios/{scenarioId}/inject
        │
        ▼
navyr-orchestrator (Scenario Engine)
  1. Loads scenario YAML manifest from embedded FS
  2. Applies manifest to ephemeral cluster via kubeconfig
  3. Creates lab_sessions_v2 row (status: running, started_at: now)
  4. Starts background verifier loop (polls every 10s)
        │
        ▼
Verifier loop checks resolution condition
  Pass → session status = "passed"
       → score calculated: 1000 − (elapsed_min × 10) − (hints × 50)
       → lab_leaderboard upsert
       → POST /community/badges/grant (navyr-community)
  Fail → session status = "failed"  (TTL exceeded, default 90min)
        │
        ▼
Garbage collector (every 5min): destroy clusters past TTL
  kind delete cluster --name lab-<session-id>
  lab_clusters row → status: destroyed
```

## Kind Provisioner

Each lab session runs in a **dedicated ephemeral Kind cluster** — completely isolated from the user's registered clusters. This means:

- Faults can affect node stability without risk to production
- Users have full cluster-admin in their lab cluster
- Clusters are automatically destroyed after the TTL (default 90 minutes) or when the user stops the lab

### Cluster lifecycle states

```
creating → ready → active → destroyed
                          → expired   (TTL exceeded, GC ran)
                          → failed    (kind create failed)
```

### API

| Method | Path | Description |
|---|---|---|
| `POST` | `/api/v1/labs/clusters` | Provision ephemeral Kind cluster |
| `GET` | `/api/v1/labs/clusters/{id}/status` | Provisioning status |
| `DELETE` | `/api/v1/labs/clusters/{id}` | Destroy cluster immediately |

## Scenario Engine

Scenarios are declarative YAML manifests embedded in the orchestrator binary. Each scenario defines:

- **Fault manifest** — what to apply to inject the fault
- **Verify function** — Go function that checks the resolution condition via K8s API
- **Hints** — up to 3 progressive hints (each costs 50 points)

### Built-in scenarios

| ID | Fault injected | What you must do | Resolution condition |
|---|---|---|---|
| `crashloop-env` | Missing required env var → CrashLoopBackOff | Add the missing env var to the deployment | Pod `Running` + 0 restarts in last 60s |
| `oom-killer` | Memory limit 32Mi but app allocates 128Mi | Increase memory limit to allow the app to run | No OOMKilled event in last GC cycle |
| `pending-pods` | CPU request `10` cores (impossible to schedule) | Reduce CPU request to schedulable value | Pod `Running` |
| `rbac-lockout` | ServiceAccount bound to nonexistent Role | Create the missing Role or rebind to a valid one | ServiceAccount has valid working Role binding |

### Scenario API

| Method | Path | Description |
|---|---|---|
| `POST` | `/api/v1/labs/clusters/{id}/scenarios/{scenarioId}/inject` | Inject fault |
| `POST` | `/api/v1/labs/clusters/{id}/scenarios/{scenarioId}/verify` | Run resolution verifier |
| `GET` | `/api/v1/labs/clusters/{id}/scenarios/{scenarioId}/hint` | Get progressive hint (costs 50 pts) |

## Scoring

```
score = 1000 − (elapsed_minutes × 10) − (hints_used × 50)
```

- Maximum score: **1000** (solve immediately, no hints)
- Each minute costs 10 points
- Each hint costs 50 points
- Minimum score: 0 (no negative scores)

Scores are persisted in `lab_leaderboard` and visible on the global leaderboard per scenario.

### Score & Leaderboard API

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/v1/labs/score/{sessionId}` | Current session score |
| `GET` | `/api/v1/labs/leaderboard/{scenarioId}` | Top-10 leaderboard |

## Community integration

On lab pass:
- `navyr-orchestrator` calls `POST /community/badges/grant` on `navyr-community`
- The badge is associated with the user's GitHub identity (if linked)
- The badge appears on the user's public profile at `/community/profile/{username}`
- The user's `lab_leaderboard` entry is updated with score, time, and scenario

## Safety

Labs run in completely isolated ephemeral clusters — they have **no access** to the user's registered production clusters. The orchestrator never grants lab kubeconfigs to the gateway; they are used only internally for fault injection and verification.

## Agent-mode Labs (Phase 7)

Labs can also run against **existing clusters already connected to Navyr via agent mode** — not just Kind-provisioned ephemeral clusters. In this mode:

- `POST /api/v1/clusters/{id}/labs/{labId}/start` provisions the lab in the specified registered cluster via agent tunnel.
- The lab handler (`NewLabHandlerWithTunnel`) obtains a per-cluster K8s client via `tunnelRegistry.GetClient(clusterID)` instead of the global k8sService.
- If the cluster is not reachable via tunnel, the endpoint returns `503 Service Unavailable` with message `cluster not reachable`.
- `GET /api/v1/clusters/{id}/labs/{labId}/status` polls the verifier using the same tunnel client.
- `DELETE /api/v1/clusters/{id}/labs/{labId}` cleans up lab resources on the correct cluster.

This ensures lab operations always target the cluster selected by the user, regardless of how many clusters are registered in the org.
