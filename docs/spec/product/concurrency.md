# Concurrency

## Rules

- exec sessions are isolated per user
- multiple requests per tenant allowed
- limit concurrent K8s calls per tenant

## Limits

- max 5 concurrent exec sessions per tenant
- max 10 parallel K8s requests
