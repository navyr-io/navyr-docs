# Kubernetes Interaction Model

## Modes

1. Read Mode (default)
- list resources
- describe resources
- fetch logs/events

2. Action Mode
- exec into pod
- restart deployment
- scale workload

3. Watch Mode (future)
- real-time updates via watch API

## Rules

- All operations must be namespace-scoped when possible
- Cluster-wide resources require explicit permission
- Timeouts must be enforced (max 5s default)
- Errors from Kubernetes must be normalized
