# FinOps

**Last updated: 2026-05-24**

## Overview

The FinOps module provides cross-cluster cost efficiency visibility across all clusters registered to an org. It aggregates resource utilization, estimates cloud spend, identifies waste, and surfaces right-sizing recommendations — without requiring cloud billing API access.

Efficiency is calculated from actual pod resource requests vs. node allocatable capacity, giving a utilization-based approximation of cloud spend efficiency.

## Endpoint

### `GET /api/v1/finops/summary`

Returns the FinOps summary for all `ready` clusters in the authenticated org.

**Request headers:**
- `Authorization: Bearer <token>`
- `X-Internal-Context: <gateway-injected>` (org_id, user_id, role)

**Response:**

```json
{
  "org_id": "uuid",
  "generated_at": "2026-05-24T14:00:00Z",
  "total_clusters": 4,
  "total_nodes": 18,
  "total_pods": 142,
  "estimated_monthly_cost_usd": 2140.50,
  "efficiency_score": 71,
  "waste_usd": 412.00,
  "savings_opportunity_usd": 320.00,
  "clusters": [
    {
      "cluster_id": "uuid",
      "name": "prod-us-east",
      "nodes": 8,
      "pods": 64,
      "cpu_utilization_pct": 58,
      "mem_utilization_pct": 71,
      "estimated_monthly_cost_usd": 980.00,
      "efficiency_score": 65,
      "waste_usd": 280.00
    }
  ],
  "top_waste_clusters": [
    {
      "cluster_id": "uuid",
      "name": "prod-us-east",
      "waste_usd": 280.00,
      "primary_cause": "over-provisioned memory"
    }
  ],
  "recommendations": [
    "Right-size payments-api: 3 replicas → 2 sufficient at p99 load (saves ~$80/mo)",
    "notification-worker requests 200m CPU but uses <10m consistently (saves ~$40/mo)",
    "dev cluster is idle 16h/day — consider autoscaling to zero"
  ]
}
```

## Efficiency score

The efficiency score (0–100) is calculated per cluster:

```
cpu_util = sum(pod.cpu_requests) / node.cpu_allocatable
mem_util = sum(pod.mem_requests) / node.mem_allocatable
efficiency = (cpu_util * 0.5 + mem_util * 0.5) * 100
```

A score above 80 is considered well-optimized. Below 50 indicates significant over-provisioning.

## Waste estimation

Waste is estimated as the cost of unused allocatable capacity:

```
waste_usd = (1 - efficiency/100) * estimated_monthly_cost_usd
```

Cost estimation uses a blended node-hour rate based on the node's CPU and memory capacity (configurable; defaults to `$0.048/vCPU-hour` + `$0.006/GB-hour`).

## Configuration

| Variable | Description | Default |
|---|---|---|
| `FINOPS_CPU_COST_PER_VCPU_HOUR` | Cost per vCPU-hour (USD) | `0.048` |
| `FINOPS_MEM_COST_PER_GB_HOUR` | Cost per GB-hour (USD) | `0.006` |
| `FINOPS_CACHE_TTL` | How long to cache the FinOps summary | `5m` |

## Frontend integration

The `FinOpsPage` consumes `GET /api/v1/finops/summary` via React Query with a 5-minute stale time, matching the server-side cache TTL.
