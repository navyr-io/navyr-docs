# Agent tunnel

**Last updated: 2026-05-24**

## Problem it solves

Traditional Kubernetes management tools require direct access to the cluster's API server — either via kubeconfig with the API server URL exposed publicly, or via VPN. This creates two problems:

1. **Security**: exposing the K8s API server to the internet is a significant attack surface
2. **Operations**: organizations with private VPCs, NAT, or corporate firewalls cannot manage clusters remotely without complex network configuration

Navyr solves this with an **agent-based tunnel**: a small process (`navyr-agent`) running inside the customer cluster that initiates an outbound WebSocket connection to `navyr-orchestrator`. The orchestrator multiplexes all Kubernetes API calls over this tunnel. No inbound firewall rules are required.

## Architecture

```
Customer cluster (private VPC)        Navyr control plane (public)
┌─────────────────────────────┐       ┌──────────────────────────┐
│                             │       │                          │
│  navyr-agent                │       │  navyr-orchestrator      │
│  ┌───────────────────────┐  │       │  :8083                   │
│  │ WebSocket client      │──┼──────►│  /api/v1/clusters/{id}/ │
│  │ (outbound connection) │  │  wss  │  agent/tunnel            │
│  └──────────┬────────────┘  │       │                          │
│             │               │       │  TunnelRegistry          │
│  ┌──────────▼────────────┐  │       │  maps cluster_id →       │
│  │ Kubernetes API client │  │       │  WebSocket connection     │
│  │ (in-cluster SA token) │  │       │                          │
│  └──────────┬────────────┘  │       └──────────────────────────┘
│             │               │
│  kube-apiserver             │
└─────────────────────────────┘
```

## Connection lifecycle

```
1. AGENT STARTUP
   navyr-agent starts → reads ORCHESTRATOR_URL, AGENT_SECRET, ORG_ID, CLUSTER_ID

2. REGISTRATION
   Agent → POST /api/v1/clusters/{id}/agent/heartbeat (HTTP)
   Orchestrator records LastAgentSeenAt

3. TUNNEL ESTABLISHMENT
   Agent → GET /api/v1/clusters/{id}/agent/tunnel (WebSocket upgrade)
   Orchestrator adds connection to TunnelRegistry[cluster_id]

4. HEARTBEAT LOOP
   Every 30s: Agent sends WebSocket ping
   Orchestrator records LastAgentSeenAt
   Cluster status = "healthy" if LastAgentSeenAt < 90s ago

5. REQUEST FORWARDING
   Browser → Gateway → Orchestrator: "list pods for cluster X"
   Orchestrator → TunnelRegistry.Get(cluster_id) → WebSocket message
   Agent receives message → calls kube-apiserver → returns response
   Orchestrator receives response → returns to gateway → browser

6. DISCONNECTION
   Agent loses connection → reconnects with exponential backoff
   During disconnect: cluster shows as "unreachable"
   LastAgentSeenAt is not updated → status derived as unreachable after 90s
```

## Agent token lifecycle

```
1. User registers cluster in Navyr UI
2. Orchestrator generates agent token: POST /api/v1/clusters/{id}/agent/token
   → Token is a JWT signed with CLUSTER_CREDENTIAL_ENCRYPTION_KEY
   → TTL: 90 days

3. User installs agent via Helm, passing the token

4. Agent authenticates with token on every WebSocket connection

5. Auto-renewal:
   POST /api/v1/clusters/{id}/agent/token/renew
   → Called by agent 7 days before expiry
   → Returns new token
   → Agent swaps token without reconnecting

6. Revocation:
   DELETE /api/v1/clusters/{id} in UI → token is invalidated
   Agent is rejected on next connection attempt
```

## Message protocol

Messages over the tunnel are length-prefixed JSON frames. Each frame has a type and a payload:

```json
// Request (orchestrator → agent)
{
  "id":      "req-uuid",
  "type":    "k8s_request",
  "method":  "GET",
  "path":    "/apis/apps/v1/namespaces/default/deployments",
  "headers": {}
}

// Response (agent → orchestrator)
{
  "id":      "req-uuid",
  "type":    "k8s_response",
  "status":  200,
  "body":    "<base64-encoded response body>"
}

// Exec frame (bidirectional)
{
  "id":      "exec-uuid",
  "type":    "exec_stdin" | "exec_stdout" | "exec_stderr" | "exec_resize",
  "data":    "<base64>"
}

// Heartbeat
{
  "type": "ping"
}
```

## Cluster status derivation

Cluster status is **not stored in the database**. It is derived from `LastAgentSeenAt` at query time:

```
Now - LastAgentSeenAt < 90s  → "healthy"
Now - LastAgentSeenAt < 5min → "degraded"
Now - LastAgentSeenAt >= 5min → "unreachable"
Cluster never connected       → "pending"
```

This means status is always live — there is no background job that writes a stale status to the database.

## Security properties

| Property | Mechanism |
|---|---|
| No inbound firewall rules | Agent initiates outbound WebSocket |
| Agent authentication | JWT token validated on every connection |
| Transport encryption | TLS (wss://) in production |
| Token rotation | Auto-renew 7 days before expiry |
| Token revocation | Immediate on cluster delete |
| Secret isolation | Agent reads secrets metadata only; raw values never forwarded |
| Exec audit | All pod exec sessions logged with actor, command, cluster, namespace |

## Installation

See [navyr-agent](https://github.com/navyr-io/navyr-agent) for Helm installation instructions. Tokens are issued from the Navyr UI under **Clusters → Add Cluster**.
