# Data Model

## Tenant
- id
- name
- plan
- created_at

## User
- id
- tenant_id
- email
- password_hash
- role

## Cluster
- id
- tenant_id
- name
- kubeconfig (encrypted)
- status
- created_at

## UsageEvent
- id
- tenant_id
- type
- metadata
- created_at
