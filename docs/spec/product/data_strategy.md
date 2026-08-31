# Data Strategy

## Approach

MVP:
- on-demand fetch from Kubernetes API
- no local cache

Future:
- caching layer (Redis)
- background sync

## Rules

- never store full cluster state
- only store metadata (clusters, users, etc.)
