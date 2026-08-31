# Non-Functional Requirements

## Performance
- API responses < 300ms (p95) for non-K8s calls
- K8s operations < 2s (best effort)

## Scalability
- Support 100 tenants initially
- Each tenant up to 20 clusters

## Availability
- Target: 99.5% uptime (MVP)
- Graceful degradation if K8s unavailable

## Security
- All endpoints require authentication (except signup/login)
- Sensitive data must be encrypted at rest

## Multi-tenancy
- strict tenant isolation
- no cross-tenant data leakage

## Cost
- minimize unnecessary LLM/API calls
