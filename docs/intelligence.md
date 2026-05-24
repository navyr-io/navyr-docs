# Intelligence

**Last updated: 2026-05-24 (Phase 7 — AIOps bridge)**

## Overview

The Intelligence module provides a continuously-updated view of threats, anomalies, and AI-generated recommendations across all clusters in an org. It combines signals from security scanning, RBAC analysis, image CVE data, Falco runtime events, and the AIOps baseline engine into a single prioritized feed.

## Architecture

```
Signal sources
  ├── Security scanner (Trivy, Falco, RBAC risk, config risk)
  ├── AIOps engine — Anomaly Detection Worker (real cluster data via agent tunnel)
  │     └── pod_crash_loop · pod_oomkill · pod_high_restarts
  │         deployment_unavailable · node_pressure · node_not_ready
  ├── Compliance scoring engine
  └── Topology change events
          │
          ▼
navyr-orchestrator (Intelligence Aggregator)
  ├── POST /api/v1/aiops/signals/ingest  ← inbound signals
  ├── GET  /api/v1/intelligence/summary  ← snapshot (HTTP polling fallback)
  └── WS   /api/v1/intelligence/stream   ← real-time push (preferred)
        ├── snapshot includes active anomalies (field: "anomalies")
        └── AIOps anomalies emitted as type="signal" on each WS tick
          │
          ▼
navyr-frontend (SecurityIntelligencePage)
  ├── Connects to WS on mount
  ├── Receives snapshot on connect, then incremental signals
  └── Falls back to 30s HTTP polling if WS unavailable
```

## REST Endpoint

### `GET /api/v1/intelligence/summary`

Returns the current intelligence snapshot for the authenticated org.

```json
{
  "org_id": "uuid",
  "generated_at": "2026-05-24T14:00:00Z",
  "threat_level": "high",
  "open_threats": 3,
  "anomalies": 1,
  "recommendations": 5,
  "top_signals": [
    {
      "id": "uuid",
      "cluster_id": "uuid",
      "cluster_name": "prod-us-east",
      "kind": "threat",
      "severity": "critical",
      "title": "Privileged container detected in production",
      "description": "payments-api pod running with securityContext.privileged=true",
      "workload": "payments-api",
      "namespace": "production",
      "detected_at": "2026-05-24T13:55:00Z",
      "remediation": "Remove privileged flag from pod spec"
    }
  ]
}
```

## WebSocket Stream

### `WS /api/v1/intelligence/stream`

Persistent WebSocket connection for real-time intelligence updates. Replaces the previous 30-second polling approach.

**Authentication:** Bearer token in `Authorization` header (or `?token=` query param for WS clients that don't support headers).

### Message flow

```
Client → Server: subscribe
Server → Client: snapshot     (full current state)
Server → Client: signal       (incremental updates as they arrive)
Server → Client: heartbeat    (every 15s to keep connection alive)
```

### Client subscribe message

```json
{
  "type": "subscribe",
  "cluster_id": "uuid-or-empty-for-all",
  "org_id": "uuid"
}
```

### Server message types

**`snapshot`** — sent immediately after subscribe, contains the full intelligence state:
```json
{
  "type": "snapshot",
  "payload": { /* same shape as GET /intelligence/summary */ }
}
```

**`signal`** — sent when a new signal arrives or an existing one changes. AIOps anomalies are emitted in this format:
```json
{
  "type": "signal",
  "payload": {
    "id": "anomaly-uuid",
    "severity": "critical",
    "signal_type": "pod_crash_loop",
    "title": "CrashLoopBackOff detected",
    "detail": "Pod api-server-xyz has restarted 12 times in the last 10 minutes",
    "cluster_id": "cluster-uuid",
    "cluster_name": "kind-prod",
    "workload": "api-server-xyz",
    "namespace": "production",
    "detected_at": "2026-05-24T21:00:00Z"
  }
}
```

**`heartbeat`** — sent every 15 seconds:
```json
{
  "type": "heartbeat",
  "ts": "2026-05-24T14:00:15Z"
}
```

## Frontend integration

The `SecurityIntelligencePage` connects to the WebSocket stream on mount:

```tsx
const ws = new WebSocket(`ws://localhost:8080/api/v1/intelligence/stream`);
ws.onopen = () => ws.send(JSON.stringify({ type: "subscribe", org_id, cluster_id }));
ws.onmessage = (e) => {
  const msg = JSON.parse(e.data);
  if (msg.type === "snapshot") setIntelligence(msg.payload);
  if (msg.type === "signal")   updateSignal(msg.payload);
};
```

If the WebSocket connection fails or is unavailable, the page falls back to polling `GET /api/v1/intelligence/summary` every 30 seconds.
