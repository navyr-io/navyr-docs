# Error Model

All errors follow this format:

{
  "error": {
    "code": "string",
    "message": "string",
    "details": {}
  }
}

## Error Codes

AUTH_INVALID_CREDENTIALS
AUTH_UNAUTHORIZED
TENANT_NOT_FOUND
CLUSTER_INVALID_CONFIG
FEATURE_NOT_ALLOWED
K8S_CONNECTION_FAILED
INTERNAL_ERROR
