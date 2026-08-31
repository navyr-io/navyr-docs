# Workloads

## Pods
- list
- describe
- logs
- exec
- delete

## Deployments
- list
- describe
- rollout status
- rollout restart
- scale

## StatefulSets
- list
- describe
- scale
- restart

## DaemonSets
- list
- describe
- rollout status

## Jobs
- list
- describe
- delete

## CronJobs
- list
- describe
- trigger manually (future)

# Namespaces

- list namespaces
- switch namespace context
- create namespace (admin)
- delete namespace (admin)

Rules:
- all resources are namespace-scoped unless cluster-wide

# Networking

## Services
- list
- describe
- show endpoints
- show type (ClusterIP, NodePort, LoadBalancer)

## Ingress
- list
- describe
- show routes (host/path)

## NetworkPolicies (future)
- list
- describe

# Configuration

## ConfigMaps
- list
- view data
- edit (future)

## Secrets
- list (masked)
- view metadata only
- never expose raw values

# Storage

## PersistentVolumeClaims (PVC)
- list
- describe
- show bound PV

## PersistentVolumes (PV)
- list
- describe

# Security

## ServiceAccounts
- list
- describe

## Roles / RoleBindings
- list
- describe

## ClusterRoles (future)
- read-only view


