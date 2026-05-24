# Deployment

**Last updated: 2026-05-24**

## Option 1: Docker Compose (quick-start)

The fastest path to a running Navyr instance. See [navyr-deploy](https://github.com/navyr-io/navyr-deploy) for the full configuration.

```bash
# 1. Clone the deployment config
git clone https://github.com/navyr-io/navyr-deploy
cd navyr-deploy

# 2. Authenticate with GHCR
docker login ghcr.io -u YOUR_GITHUB_USERNAME -p YOUR_PAT

# 3. Configure
cp .env.example .env
# Fill in required secrets (see below)

# 4. Start
docker compose up -d

# 5. Check health
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
# Add the chart repo (or use the local chart)
helm install navyr ./navyr-platform \
  --namespace navyr \
  --create-namespace \
  -f navyr-platform/values-prod.example.yaml \
  --set global.jwtSecret=<JWT_SECRET> \
  --set global.databaseUrl=postgresql://navyr:<PW>@postgres:5432/navyr \
  --set global.internalContextSecret=<INTERNAL_SECRET> \
  --set global.credentialEncryptionKey=<CRED_KEY> \
  --set global.wsTicketSecret=<WS_SECRET>
```

### Key Helm values

```yaml
global:
  jwtSecret: ""               # required — must match across gateway, auth, orchestrator
  internalContextSecret: ""   # required
  credentialEncryptionKey: "" # required
  wsTicketSecret: ""          # required
  databaseUrl: ""             # required

gateway:
  replicas: 2
  image:
    tag: latest
  rateLimiting:
    enabled: false
    redisUrl: ""

auth:
  smtp:
    enabled: false
    host: ""
    port: 587
    username: ""
    password: ""
    fromEmail: ""
  totp:
    encryptionKey: ""
  aiProvider:
    secretKey: ""

ingress:
  enabled: true
  host: navyr.example.com
  tls:
    enabled: true
    secretName: navyr-tls

postgresql:
  enabled: true   # set false to use external postgres
  auth:
    password: ""
```

---

## Option 3: Kustomize

```bash
kubectl apply -k k8s/overlays/prod
```

Overlays available:
- `k8s/overlays/dev` — local kind/minikube, no TLS, relaxed limits
- `k8s/overlays/staging` — staging environment
- `k8s/overlays/prod` — production-hardened, PDB, resource limits, NetworkPolicy

---

## Connecting clusters

After the platform is running, connect Kubernetes clusters via the agent:

```bash
# 1. In the Navyr UI: Clusters → Add Cluster → Copy install command
# The command includes a pre-generated agent token

# 2. Install the agent in the target cluster
helm install navyr-agent oci://ghcr.io/navyr-io/navyr-agent \
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
| `latest` | Latest build from `main` branch |
| `main` | Alias for `latest` |
| `sha-<commit>` | Pinned to a specific commit (recommended for production) |

Example to pin:
```bash
NAVYR_VERSION=sha-b67bf1e docker compose up -d
```
