# Security architecture

**Last updated: 2026-05-24**

## Authentication flow

```
Client                Gateway              Auth               Billing
  │                     │                   │                    │
  │  POST /auth/login   │                   │                    │
  │────────────────────►│                   │                    │
  │                     │  POST /auth/login │                    │
  │                     │──────────────────►│                    │
  │                     │  {jwt, refresh}   │                    │
  │                     │◄──────────────────│                    │
  │  {jwt, refresh}     │                   │                    │
  │◄────────────────────│                   │                    │
  │                     │                   │                    │
  │  GET /api/v1/... Bearer <jwt>           │                    │
  │────────────────────►│                   │                    │
  │                     │  POST /auth/token/validate             │
  │                     │──────────────────►│                    │
  │                     │  {org_id,user,role,permissions}        │
  │                     │◄──────────────────│                    │
  │                     │  POST /billing/v1/enforcement/check    │
  │                     │───────────────────────────────────────►│
  │                     │  {allow}                               │
  │                     │◄───────────────────────────────────────│
  │                     │  X-Internal-Context: <signed payload>  │
  │                     │──────────────────► downstream          │
  │  200 response       │◄───────────────── downstream          │
  │◄────────────────────│                   │                    │
```

## JWT structure

All JWTs are HS256 signed with a shared secret between `navyr-gateway` and `navyr-auth`. The secret must match exactly across both services.

**Claims:**
```json
{
  "sub": "<user_uuid>",
  "org": "<org_uuid>",
  "role": "admin",
  "email": "user@example.com",
  "exp": 1748131200,
  "iat": 1748044800,
  "jti": "<unique token id>"
}
```

Access tokens have a short TTL (typically 1 hour). Refresh tokens are stored in the database and rotated on each use (sliding window). Refresh tokens are revoked on logout.

## RBAC model

Navyr uses a layered RBAC model:

```
Platform level:
  Platform Admin  — full access to all orgs (SaaS/self-hosted admin)

Org level:
  Owner           — full control of org settings, billing, member management
  Admin           — manage users, clusters, settings; cannot delete org
  Operator        — cluster operations (scale, restart, exec, apply)
  Viewer          — read-only across all clusters

Cluster/Namespace level (fine-grained grants):
  Grant{user_id or group_id, cluster_id, namespace, role}
  — Grants override the org-level role downward or upward for that scope
```

Groups support two types:
- `local` — created and managed within Navyr
- `ldap` — synced from LDAP/AD using `ldap_group_dn`; membership is refreshed on sync

## Internal context header

Every service-to-service call carries an `X-Internal-Context` header signed by the gateway. Services validate the HMAC signature before processing the request, preventing any client from forging internal context.

```
Format: base64(json_payload) + "." + hmac_sha256(base64(json_payload), INTERNAL_SECRET)

Payload:
{
  "org_id":      "uuid",
  "user_id":     "uuid",
  "role":        "operator",
  "cluster_id":  "uuid",        // present only for cluster-scoped routes
  "permissions": ["pods:read", "deployments:write"],
  "issued_at":   1748044800
}
```

The signing secret (`INTERNAL_CONTEXT_SIGNING_SECRET`) is independent from `JWT_SECRET` — rotating it invalidates all in-flight internal requests without affecting client tokens.

## Cluster credential encryption

Cluster credentials (service account tokens, kubeconfig fragments for kubectl download) are encrypted at rest using AES-GCM with a two-layer envelope:

```
DEK (Data Encryption Key)
  → AES-256-GCM encrypts the credential payload
  → DEK is itself encrypted by the KEK

KEK (Key Encryption Key)
  → Local mode: AES-256 key stored in CLUSTER_CREDENTIAL_ENCRYPTION_KEY env var
  → AWS KMS mode: DEK is envelope-encrypted with a KMS CMK

Rotation:
  POST /api/v1/clusters/{id}/credentials/rotate      → rotates the DEK
  POST /api/v1/clusters/{id}/credentials/rotate-kek  → re-encrypts all DEKs with a new KEK
```

## Pod exec security

Pod exec sessions use a single-use WebSocket ticket to prevent replay attacks:

```
1. Client → POST /api/v1/pods/{name}/exec/ws-ticket
   → Gateway validates JWT + RBAC
   → Orchestrator issues a signed ticket (TTL: 30s, single use)
   → Returns {ticket_id}

2. Client → GET /api/v1/pods/{name}/exec/ws?ticket=<ticket_id>
   → Orchestrator validates ticket (TTL, not consumed)
   → Marks ticket consumed
   → Upgrades to WebSocket
   → Relays stdin/stdout/stderr through agent tunnel to pod

3. Session is audited in exec_audits table (command, user, cluster, pod, namespace, timestamp)
```

Raw pod exec without a ticket is not permitted — the WebSocket endpoint requires a valid, unconsumed ticket.

## Secret handling

Navyr never returns raw Kubernetes secret values through the API or the agent tunnel. The secrets endpoints (`GET /api/v1/secrets`, `GET /api/v1/secrets/{name}`) return metadata only (name, namespace, labels, annotations, data keys — not data values).

## AI provider keys (BYOK)

When users configure a Bring Your Own Key (BYOK) AI provider, the API key is:
1. Encrypted with AES-256-GCM using `AI_PROVIDER_SECRET_KEY`
2. Stored in the `ai_provider_configs` table
3. Never returned in plaintext via the API — only used internally when the gateway proxies AI completion requests

## TOTP 2FA

TOTP secrets are encrypted with AES-256-GCM using `TOTP_ENCRYPTION_KEY` before storage. The plaintext seed is only ever returned once — during setup — and never again. Verification is done server-side using the stored ciphertext.

## Audit trail

Every write operation emits an audit event to `navyr-billing` asynchronously. Audit events are immutable, append-only, and include:

| Field | Description |
|---|---|
| `org_id` | Organization |
| `actor_id` | User UUID |
| `actor_role` | Role at time of action |
| `actor_ip` | Client IP |
| `action` | e.g. `deployment.scale`, `pod.delete`, `cluster.revoke` |
| `resource_kind` | Resource type |
| `resource_name` | Resource name |
| `cluster_id` | Target cluster |
| `namespace` | Target namespace |
| `outcome` | `success` / `denied` / `error` |
| `timestamp` | UTC |

Audit events can be exported as JSON or CSV for compliance requirements. Retention is configurable per org.

## Security scanning capabilities

The orchestrator integrates the following security tools:

| Tool | Capability | Mode |
|---|---|---|
| **Trivy** | Image CVE scanning | Bundled in image (`trivy` binary) |
| **Trivy** | Kubernetes config risk (misconfigured pods, privileges, etc.) | Bundled |
| **Cosign** | Image signature verification (supply chain) | Via API call |
| **SBOM** | Software bill of materials generation | Via Trivy |
| **Falco** | Runtime event feed | Reads from cluster via agent tunnel |
| **RBAC analysis** | Dangerous permission combinations, escalation paths | Internal logic |
