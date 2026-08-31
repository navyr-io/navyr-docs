# Extensibility

## Resource Handling

- each Kubernetes resource must have its own module
- shared interface for resource handlers

## Example

ResourceHandler:
- list()
- describe()
- actions()

## Goal

- easy addition of new resource types
