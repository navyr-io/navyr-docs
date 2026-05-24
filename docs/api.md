# API conventions

**Last updated: 2026-05-24**

## Base URL

All client-facing API traffic goes through `navyr-gateway`:

```
http(s)://<host>:8080
```

In local development this is `http://localhost:8080`. The frontend proxies requests via Vite's dev proxy to the same address.

## Authentication

All endpoints except public auth paths require a Bearer JWT:

```
Authorization: Bearer <jwt>
```

JWTs are issued by `navyr-auth /auth/login` and have a short TTL (typically 1 hour). Use `POST /auth/token/refresh` with the refresh token to obtain a new access token without re-logging in.

**Public endpoints (no auth):**
- `POST /auth/register`
- `POST /auth/login`
- `POST /auth/token/refresh`
- `POST /auth/invites/accept`
- `POST /auth/password/reset/request`
- `POST /auth/password/reset/confirm`
- `GET /auth/sso/{provider}/login`
- `GET /auth/sso/{provider}/callback`
- `GET /community/profile/{username}`
- `GET /community/badges`
- `GET /health`

## Versioning

API paths use version prefixes:

| Prefix | Services | Notes |
|---|---|---|
| `/api/v1/` | orchestrator, billing | Stable client-facing API |
| `/auth/` | auth | Auth service paths (unversioned by design) |
| `/community/` | community | Community paths |
| `/billing/v1/` | billing | Internal service-to-service only |
| `/internal/v1/` | orchestrator | Internal RBAC endpoints (gateway → orchestrator) |

## Cluster-scoped requests

Most Kubernetes operation endpoints require the target cluster to be specified via a header:

```
X-Cluster-ID: <cluster-uuid>
```

The cluster UUID is the `id` field from `GET /api/v1/clusters`. In the frontend, it is extracted from the route parameter `/clusters/:id`.

## Namespace scoping

By default, workload endpoints return resources across all namespaces. To scope to a specific namespace:

```
GET /api/v1/pods?namespace=production
```

## Pagination

List endpoints support cursor-based pagination:

```
GET /api/v1/pods?limit=50&continue=<token>
```

The `continue` token is returned in the response as `metadata.continue` (mirrors the Kubernetes API convention). An empty `continue` means the last page.

## Error format

All errors return a JSON body with a consistent shape:

```json
{
  "error": "resource not found",
  "code": "NOT_FOUND",
  "details": {}
}
```

Common error codes:

| HTTP | Code | Meaning |
|---|---|---|
| `400` | `INVALID_REQUEST` | Malformed body or missing field |
| `401` | `UNAUTHORIZED` | Missing or invalid JWT |
| `403` | `FORBIDDEN` | Valid JWT but insufficient permissions |
| `403` | `PLAN_LIMIT` | Action denied by billing plan enforcement |
| `404` | `NOT_FOUND` | Resource does not exist |
| `409` | `CONFLICT` | Duplicate resource |
| `429` | `RATE_LIMITED` | Too many requests |
| `502` | `TUNNEL_UNAVAILABLE` | Cluster agent not connected |
| `503` | `SERVICE_UNAVAILABLE` | Downstream service unreachable |

## Rate limiting

Rate limiting is applied per organization and per IP at the gateway level.

| Tier | Limit | Window |
|---|---|---|
| Default (OSS) | 500 req/min per org | Rolling 60s |
| Enterprise | Configurable | — |
| Internal service-to-service | Unlimited | — |

When `RATE_LIMIT_ENABLED=false` (default), no rate limiting is applied. Enable it with Redis for multi-instance deployments:

```
RATE_LIMIT_ENABLED=true
REDIS_URL=redis://navyr-redis:6379
```

## CORS

The gateway enforces CORS. Allowed origins are configured via:

```
CORS_ALLOWED_ORIGINS=https://app.navyr.io,http://localhost:5173
```

## Response conventions

- Successful creates: `201 Created` with the created resource body
- Successful updates: `200 OK` with the updated resource body
- Successful deletes: `204 No Content`
- Long-running operations (drain, scan): `202 Accepted` with a polling URL

## WebSocket endpoints

Two types of WebSocket connections exist:

| Endpoint | Purpose | Auth |
|---|---|---|
| `GET /api/v1/pods/{name}/exec/ws?ticket=<ticket>` | Pod shell | Single-use ticket |
| `GET /api/v1/clusters/{id}/agent/tunnel` | Agent tunnel | Agent JWT token |

WebSocket connections are established after the HTTP upgrade handshake. The ticket for pod exec is obtained via `POST /api/v1/pods/{name}/exec/ws-ticket` (requires full JWT auth).

## Server-Sent Events

Real-time list updates use SSE streams:

| Endpoint | Events |
|---|---|
| `GET /api/v1/pods/watch` | Pod add / update / delete |
| `GET /api/v1/events/watch` | Kubernetes event stream |

SSE streams require `Accept: text/event-stream` and a valid JWT. They reconnect automatically on the client side (browser `EventSource` handles reconnection).
