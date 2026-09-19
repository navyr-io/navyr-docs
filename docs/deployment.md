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

## Option 3: ECS on Fargate (AWS)

> **Status: complete in code, never applied.** The module passes
> `terraform validate` and its contract gate, but has never run against AWS.
> `validate` accepts things `apply` rejects. Treat the first install as the test
> that is still missing. See navyr-io/navyr-deploy#17.

### Why this path exists

**The control plane does not run on the thing it manages.** Running Navyr inside
Kubernetes creates the classic circular dependency: the day the tool is most
needed is the day it may be down with the cluster. For an AWS team that wants to
manage EKS without standing up an extra cluster just to host the tool, this
removes an adoption objection.

### What it creates

Nine services in the contract become eight Fargate tasks plus two managed
services — Postgres becomes RDS, Redis becomes ElastiCache. The eighth task is a
`trivy server`, which exists only on this path (see below).

Nothing is hand-written: every service, port, image and health path is derived
from `contrato/plataforma.yaml`, and the gate
`tests/contract/test_ecs_cobre_o_contrato.py` fails when the module stops
covering it.

### Before applying: mirror to ECR

Tasks run in private subnets **with no NAT Gateway**, so nothing is pulled from
`ghcr.io` at runtime. This is ADR 0007 ("the application fetches nothing from
outside") applied to infrastructure; it also avoids roughly US$32/month of NAT,
which costs more than the Trivy task.

```bash
# Creates nothing in AWS — copies images and the vulnerability database.
# Needs `crane`: the database is an OCI artifact, and `docker buildx
# imagetools` cannot copy it.
scripts/ops/espelhar-para-ecr.sh <account>.dkr.ecr.<region>.amazonaws.com 0.1.0
```

**Put the database mirror on a schedule, not just at install time.** A stale
vulnerability database does not error — it produces false negatives. Upstream
publishes daily.

### Install

```bash
cd terraform/ecs
terraform init

terraform apply \
  -var 'regiao=us-east-1' \
  -var 'dominio=navyr.example.com' \
  -var 'arn_do_certificado=arn:aws:acm:...' \
  -var 'registro_ecr=<account>.dkr.ecr.us-east-1.amazonaws.com' \
  -var 'versao_das_imagens=0.1.0'
```

Then finish three steps the module deliberately does not do:

1. **Create the ECR repositories.** Output `repositorios_ecr_necessarios` lists
   the eight names.
2. **Fill the three external secrets.** Output `segredos_a_preencher` lists them
   with reasons. They are created empty on purpose: putting a value in Terraform
   puts it in the state file, which someone eventually commits. Tasks do not
   start until the values are there — a visible failure instead of a leaked
   credential.
3. **Point DNS at the load balancer**, using outputs `dns_do_alb` and
   `zona_do_alb`.

### Key variables

| Variable | Description |
|---|---|
| `versao_das_imagens` | Image tag. Rejects `latest`: a moving tag makes two installs of the same module version run different code. |
| `registro_ecr` | ECR host. The module never pulls from `ghcr.io`. |
| `kms.id_da_chave` / `kms.arn_da_chave` | KMS envelope encryption for cluster credentials (ADR 0002). Empty disables it and the orchestrator uses the local key. The key is **not** created by the module: destroying the stack must not take the key that makes database backups readable. |
| `ia.usar_bedrock` | Grants the gateway `bedrock:InvokeModel` for Navy. Off by default — the alternative is an external provider via `AI_PROVIDER_SECRET_KEY`. |
| `postgres.protegido_contra_remocao` | On by default. `skip_final_snapshot` is convenient in a lab and deletes a customer database with no safety net. |
| `permitir_ecs_exec` | Off by default: it is a shell inside the process that holds customer cluster credentials. |
| `dimensionamento` | CPU and memory per service. The orchestrator gets more — it runs the Trivy client and holds the agent WebSockets. |

### Two things this path does differently, and why

**Redis is encrypted and authenticated.** `transit_encryption_enabled`,
`at_rest_encryption_enabled` and an AUTH token. This only became possible when
navyr-collector learned to read `REDIS_URL`: it was the one service that opened
Redis by bare address, with no password and no TLS, so enabling AUTH would have
taken it down alone. Redis holds user sessions and multi-tenant operational
state; leaving it unauthenticated and relying on network isolation alone was the
alternative, and it was worse.

**Trivy runs as a server.** Measured on the binary embedded in
`navyr-orchestrator:0.1.0` (Trivy 0.74.0): the vulnerability database is
**1.3 GB** and takes about 53 seconds to download, and it always lives on the
filesystem — `--cache-backend redis://` governs the *scan* cache, not the
database. On Fargate, where the disk is discarded on every deploy, embedding it
in the orchestrator would cost that download after every rollout.

### The limitation you must read before selling this

**Every orchestrator deploy interrupts the agent tunnels.**

`tunnel.Registry` is a `map[uuid.UUID]*Conn` in process memory with no
coordination (`internal/tunnel/tunnel.go:255`). With two instances, an agent
connects to one and a request routed to the other gets `agent not connected` —
the product breaks rather than degrades. So `desired_count` is 1, and
`deployment_minimum_healthy_percent` must be 0, because ECS has nowhere to place
a second task before removing the first.

During that window, cluster actions fail. Agents reconnect on their own
afterwards. Output `janela_de_indisponibilidade_no_deploy` states this.

The argument for this path is that the control plane survives cluster
degradation. It does — it does not survive its own deploy. Taking `Registry` out
of process memory is what would unlock HA, on every path, and it is tracked
separately.

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
helm install navyr-agent oci://ghcr.io/navyr-io/charts/navyr-agent --version 0.1.1 \
  --namespace navyr-agent \
  --create-namespace \
  --set image.tag=0.1.0 \
  --set agent.orchestratorUrl=wss://<NAVYR_HOST> \
  --set agent.agentSecret=<SECRET_FROM_UI> \
  --set agent.orgId=<ORG_ID> \
  --set agent.clusterId=<CLUSTER_ID>

# 3. Verify
kubectl -n navyr-agent get pods
# navyr-agent-xxxx   1/1   Running
```

> **Three things changed in this command on 2026-09-19**, after measuring that it
> did not work as written:
>
> - **`--set image.tag=` is required.** Without it the chart refuses to render:
>   `image.tag is required`. The command above did not set it, so it never got
>   as far as installing.
> - **`agent.token` was never a value of this chart** — the real name is
>   `agent.agentSecret`. The flag was silently ignored and the Secret was
>   created with an empty `agent-secret`, so the agent came up and never
>   authenticated. The chart now fails with a named error instead.
> - **The URL is the edge, not port 8083.** The tunnel enters through `/api` at
>   the edge and crosses the gateway, which is what Helm and ECS already did;
>   compose was the odd one out until navyr-deploy#39.
>
> The chart's default `image.repository` also pointed at `ghcr.io/navyr/executor`,
> which does not exist — `docker manifest inspect` returns `denied`. It is
> `ghcr.io/navyr-io/navyr-agent`. `helm template` and `helm lint` do not pull
> images, which is why rendering the chart passed while installing it could not.

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
