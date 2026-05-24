# Operational Labs

**Last updated: 2026-05-24**

## What are Operational Labs

Operational Labs are hands-on failure scenarios that run inside a real Kubernetes cluster. Each lab injects a genuine fault condition — not a simulation — using a Helm chart deployed by `navyr-orchestrator`. Users are expected to diagnose and resolve the fault using Navyr's tooling (or kubectl directly). When the resolution condition is met, the verifier marks the lab as passed and issues a community badge.

Labs are designed to build operational muscle memory for the most common Kubernetes failure patterns.

## Architecture

```
User clicks "Start Lab" in Navyr UI
        │
        ▼
navyr-frontend → POST /api/v1/clusters/{id}/labs/{labId}/start
        │
        ▼
navyr-orchestrator (Lab Engine)
  1. Resolves the Helm chart for labId from navyr-helm/labs/
  2. Calls navyr-agent tunnel → helm install <fault-chart> in cluster
  3. Creates lab_session record (status: running)
  4. Starts verifier loop (polls every 10s via agent tunnel)
        │
        ▼
navyr-agent → Helm install → fault workload deployed in cluster
        │
        ▼
Verifier loop checks resolution condition
  Pass → lab_session status = "passed"
       → POST /community/badges/grant (navyr-community)
       → Community badge awarded
  Fail → lab_session status = "failed" (if TTL exceeded)
        │
        ▼
User clicks "Stop Lab" → helm uninstall → fault workload removed
```

## Lab catalog

| Lab ID | Fault injected | What you must do | Resolution condition |
|---|---|---|---|
| `crashloop-env` | Missing required env var → CrashLoopBackOff | Add the missing env var to the deployment | Pod `Running` + 0 restarts in last 60s |
| `oomkilled` | Memory limit 32Mi but app allocates 128Mi | Increase memory limit to allow the app to run | Pod without OOMKill event in last GC cycle |
| `image-pull-error` | Non-existent image tag (`app:does-not-exist`) | Fix the image tag to an existing one | Pod `Running` with valid image |
| `pending-no-resources` | CPU request `10` cores (impossible to schedule) | Reduce CPU request to schedulable value | Pod `Running` |
| `failed-rollout` | Readiness probe HTTP path returns 404 | Fix the readiness probe path | Deployment `Available` (all replicas ready) |
| `node-pressure` | DaemonSet allocating all node memory | Remove or limit the DaemonSet | Node without `MemoryPressure` condition |
| `pvc-unbound` | PVC requests a non-existent StorageClass | Create or fix the StorageClass binding | Pod `Running` with volume mounted |
| `privileged-container` | Pod with `securityContext.privileged: true` | Remove the privileged flag | Finding absent in next Navyr security scan |
| `cluster-admin-sa` | ServiceAccount bound to `cluster-admin` ClusterRole | Remove or replace the ClusterRoleBinding | Binding absent from cluster |
| `secret-in-env` | Hardcoded API key in `env:` plain text | Replace with a `secretKeyRef` reference | Secret referenced correctly, no plain value in spec |
| `no-network-policy` | Namespace with unrestricted pod communication | Apply a NetworkPolicy to restrict ingress/egress | NetworkPolicy present and enforced in namespace |
| `rbac-escalation` | ServiceAccount can read secrets across namespaces | Scope the Role to minimum required namespace | Role bound only to required namespace and resources |

## Lab lifecycle states

```
pending  → running  → passed
                   → failed  (TTL exceeded without resolution)
                   → stopped (user clicked Stop)
```

Stopped labs are cleaned up immediately (helm uninstall). Failed labs are cleaned up after a configurable TTL.

## Verifier design

Each lab has a dedicated verifier function in `navyr-orchestrator/internal/service/lab_service.go`. The verifier:

1. Polls the cluster via the agent tunnel at a fixed interval (10s)
2. Checks the exact resolution condition for that lab
3. Is idempotent — checking multiple times does not change state
4. Times out after the lab TTL (configurable, default 60 minutes)

Verifier checks use the same K8s API calls available to users — there is no special privileged check path.

## Community integration

On lab pass:
- `navyr-orchestrator` calls `POST /community/badges/grant` on `navyr-community`
- The badge is associated with the user's GitHub identity (if linked)
- The badge appears on the user's public profile at `/community/profile/{username}`
- The user's leaderboard entry (`leaderboard_entries`) is updated

## Running labs in production clusters

Labs are safe to run in non-production namespaces. By default, lab Helm charts deploy into an isolated namespace (`navyr-lab-<lab-id>`). This namespace is created on lab start and deleted on lab stop.

**Never run labs in production namespaces.** The `node-pressure` lab in particular can affect node stability.
