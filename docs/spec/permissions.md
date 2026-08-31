# Permissions

## Actions

- read:workloads
- read:logs
- exec:pods
- restart:deployments
- manage:clusters
- manage:users

## Mapping

Viewer:
- read:workloads
- read:logs

Operator:
- read:workloads
- exec:pods
- restart:deployments

Admin:
- manage:clusters
- manage:users
