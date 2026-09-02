# Deployment

**Last updated: 2026-09-02**

## Option 1: Docker Compose (quick-start)

The fastest path to a running Navyr instance. The configuration lives in
[navyr-deploy](https://github.com/navyr-io/navyr-deploy).

> **Access note.** The service images are **public** — `docker pull` needs no
> login. The `navyr-deploy` repository is currently **private**, so step 1 needs
> access to the `navyr-io` organization. Ask for access if the clone returns 404.

```bash
# 1. Clone the deployment config  (needs org access — see note above)
git clone https://github.com/navyr-io/navyr-deploy
cd navyr-deploy

# 2. Configure
cp .env.example .env
# Fill in required secrets (see below)

# 3. Start
docker compose up -d

# 4. Check health
curl http://localhost:8080/health
```

Access the workspace at **http://localhost:5173**.

### Required secrets

Generate each with `openssl rand -hex 32`:

| Variable | Purpose |
|---|---|
| `NAVYR_POSTGRES_PASSWORD` | PostgreSQL password |
| `NAVYR_JWT_SECRET` | Shared by gateway, auth, orchestrator |
| `NAVYR_INTERNAL_SECRET` | Signs `X-Internal-Context` header |
| `NAVYR_CREDENTIAL_KEY` | Encrypts cluster credentials (AES-256) |
| `NAVYR_WS_TICKET_SECRET` | Signs WebSocket exec tickets |
| `COMMUNITY_SECRETS_KEY` | Community service token signing |

### Optional: Redis rate limiting

```bash
docker compose --profile redis up -d
```

Then set in `.env`:
```
RATE_LIMIT_ENABLED=true
REDIS_URL=redis://navyr-redis:6379
```

---

## Option 2: Helm (production)

Uses the `navyr-platform` chart from [navyr-helm](https://github.com/navyr-io/navyr-helm).

```bash
# 1. Create the Secret with real values, outside the chart.
kubectl create namespace navyr
kubectl create secret generic navyr-secrets -n navyr \
  --from-literal=jwt_secret="$(openssl rand -hex 32)" \
  --from-literal=internal_context_signing_secret="$(openssl rand -hex 32)" \
  --from-literal=ws_exec_ticket_secret="$(openssl rand -hex 32)" \
  --from-literal=community_secrets_key="$(openssl rand -hex 16)" \
  --from-literal=cluster_credential_encryption_key="$(openssl rand -hex 16)" \
  --from-literal=postgres_user=postgres \
  --from-literal=postgres_password="$(openssl rand -hex 16)" \
  --from-literal=postgres_db=navyr \
  --from-literal=auth_database_url="postgres://postgres:<PW>@navyr-postgres:5432/navyr?sslmode=disable" \
  --from-literal=billing_database_url="postgres://postgres:<PW>@navyr-postgres:5432/navyr?sslmode=disable" \
  --from-literal=community_database_url="postgres://postgres:<PW>@navyr-postgres:5432/navyr?sslmode=disable" \
  --from-literal=orchestrator_database_url="postgres://postgres:<PW>@navyr-postgres:5432/navyr?sslmode=disable"

# 2. Install pointing at it.
helm install navyr ./navyr-platform \
  --namespace navyr \
  --set secrets.existingSecret=navyr-secrets \
  --set ingress.host=navyr.example.com \
  --set ingress.tls.enabled=true \
  --set ingress.tls.secretName=navyr-tls
```

> `cluster_credential_encryption_key` must be **exactly 32 bytes** — the
> `openssl rand -hex 16` above produces that. The chart validates before install.

For a throwaway local cluster the example values work, but must be accepted
explicitly:

```bash
helm install navyr ./navyr-platform --set secrets.allowInsecureDefaults=true
```

### Key Helm values

| Value | Description |
|---|---|
| `secrets.existingSecret` | Name of a Secret created outside the chart. **Recommended.** When set, the chart generates no Secret at all. |
| `secrets.allowInsecureDefaults` | Accept the example values. Install fails without this if any secret was left unchanged. Throwaway clusters only. |
| `secrets.jwtSecret` | HS256 signing secret shared by gateway and auth |
| `secrets.internalContextSigningSecret` | Signs the `X-Internal-Context` header |
| `secrets.clusterCredentialEncryptionKey` | AES-256 key for cluster credentials — exactly 32 bytes |
| `secrets.wsExecTicketSecret` | Signs the WebSocket exec ticket |
| `databaseUrls.<service>` | Postgres DSN per service |
| `images.<service>` | Image per service |
| `autoscaling.gateway.enabled` | Gateway HPA. Requires metrics-server. When on, `replicaCount.gateway` is ignored. |
| `autoscaling.orchestrator.enabled` | Same for the orchestrator |
| `ingress.enabled` | Expose via Ingress |
| `ingress.host` | Public hostname |
| `ingress.tls.enabled` | Terminate TLS at the Ingress. Without it the JWT travels in the clear. |
| `ingress.tls.secretName` | `kubernetes.io/tls` Secret, created outside the chart |

---

## Removed: Kustomize

The `k8s/` Kustomize path was retired on 2026-08-19. Helm is the single
deployment path for Kubernetes — see
[ADR 0006](adr/0006-helm-como-caminho-unico.md).

This page previously described `k8s/overlays/prod` as "production-hardened,
PDB, resource limits, NetworkPolicy". **None of the four existed there.** The
prod overlay set three environment variables and nothing else — the hardening
lived in the Helm chart the whole time.

Environment differences that lived in the overlays are chart values now:

| Overlay literal | Chart value |
|---|---|
| `APP_ENV` | `global.appEnv` |
| `CLUSTER_VALIDATION_MODE` | `orchestrator.clusterValidationMode` |
| `BILLING_DB_FALLBACK` | `billing.dbFallback` |

If you need plain manifests without Helm installed in the cluster:

```bash
helm template navyr navyr-platform/ -f your-values.yaml | kubectl apply -f -
```

This gives up release history and `helm rollback`, which the
[rollback runbook](runbooks/reverter-release.md) relies on.

---

## Connecting clusters

After the platform is running, connect Kubernetes clusters via the agent:

```bash
# 1. In the Navyr UI: Clusters → Add Cluster → Copy install command
# The command includes a pre-generated agent token

# 2. Install the agent in the target cluster
helm install navyr-agent oci://ghcr.io/navyr-io/charts/navyr-agent --version 0.1.0 \
  --namespace navyr-agent \
  --create-namespace \
  --set agent.orchestratorUrl=wss://<NAVYR_HOST>:8083 \
  --set agent.token=<TOKEN_FROM_UI> \
  --set agent.orgId=<ORG_ID> \
  --set agent.clusterId=<CLUSTER_ID>

# 3. Verify
kubectl -n navyr-agent get pods
# navyr-agent-xxxx   1/1   Running
```

The cluster will appear as **healthy** in the Navyr UI within 30 seconds.

---

## Production checklist

### Secrets
- [ ] All secrets generated with `openssl rand -hex 32` (minimum 32 bytes)
- [ ] Secrets stored in a secret manager (Vault, AWS Secrets Manager, K8s Secrets)
- [ ] `TOTP_ENCRYPTION_KEY` and `AI_PROVIDER_SECRET_KEY` set (required to enable 2FA and BYOK)
- [ ] `JWT_SECRET` is identical across gateway, auth, and orchestrator

### Database
- [ ] PostgreSQL running with a dedicated user per environment
- [ ] Connection pooling configured (PgBouncer recommended for >100 concurrent users)
- [ ] Backups scheduled (daily at minimum)
- [ ] `DATABASE_URL` uses `sslmode=require` in production

### Network
- [ ] Only `navyr-gateway :8080` and `navyr-frontend :5173` exposed externally
- [ ] All inter-service communication on an internal network
- [ ] TLS termination at load balancer or ingress (gateway does not terminate TLS itself)
- [ ] `CORS_ALLOWED_ORIGINS` set to the exact frontend domain

### Auth
- [ ] `SMTP_ENABLED=true` with a working SMTP relay (otherwise password reset and invites are broken)
- [ ] `AUTH_EXPOSE_INVITE_TOKEN=false` and `AUTH_EXPOSE_RESET_TOKEN=false` (dev-only flags)
- [ ] Strong `NAVYR_JWT_SECRET` — at least 256 bits of entropy

### Billing enforcement
- [ ] `BILLING_ENFORCEMENT_MODE=enforce` (default is `log` which does not block over-limit requests)

### Rate limiting
- [ ] Redis deployed and `RATE_LIMIT_ENABLED=true` for multi-instance gateway deployments

### Monitoring
- [ ] Prometheus scraping `GET /metrics` on gateway (`:8080/metrics`)
- [ ] Liveness probes configured: `GET /health` on all services
- [ ] Alert on cluster `last_agent_seen_at` lag > 5 minutes

---

## Image tags

| Tag | Meaning |
|---|---|
| `latest` | the most recent published build |
| `<short-sha>` | a specific commit, e.g. `7f21918` — **the tag to pin** |
| `sha-<short-sha>`, `main` | produced by an older CI pipeline; frozen since 2026-08-19 |

Example to pin:
```bash
NAVYR_VERSION=7f21918 docker compose up -d
```

To list what actually exists, use the package page for each image on GHCR
(`github.com/orgs/navyr-io/packages`) — do not guess a SHA.
