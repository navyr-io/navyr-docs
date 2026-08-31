# Failure Handling

## Kubernetes Failures

- cluster unreachable → mark cluster as "error"
- API timeout → retry once, then fail
- permission denied → return RBAC error

## System Behavior

- degraded cluster should not break UI
- partial data must be allowed
- errors must not crash request

## User Feedback

- show clear error messages
- never expose raw Kubernetes errors
