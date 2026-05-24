# Development guide

**Last updated: 2026-05-24**

## Prerequisites

| Tool | Version | Purpose |
|---|---|---|
| Go | 1.25+ | Backend services |
| Node.js | 22+ | Frontend |
| Docker + Compose v2 | any | Run infrastructure |
| PostgreSQL | 15+ | Database (or via Docker) |
| Helm | 3.18+ | Operational Labs |
| Trivy | 0.70+ | Security scanning |
| gh CLI | any | GitHub operations |

## Repository layout

```
navyr-io/
  navyr-gateway        API gateway
  navyr-auth           Identity service
  navyr-billing        Billing and enforcement
  navyr-orchestrator   Kubernetes operations engine
  navyr-community      Community and labs
  navyr-frontend       React SPA
  navyr-agent          In-cluster agent
  navyr-helm           Helm charts
  navyr-deploy         Docker Compose quick-start
  navyr-docs           This documentation
```

## Running the full stack locally

The fastest path uses Docker Compose. Backend services are pulled from GHCR; only the service you're working on needs to be run locally.

```bash
# 1. Clone navyr-deploy
git clone https://github.com/navyr-io/navyr-deploy
cd navyr-deploy
cp .env.example .env
# Fill in .env (see deployment.md for secret generation)

# 2. Start all services
docker compose up -d

# 3. Verify
curl http://localhost:8080/health
# {"status":"ok"}
```

## Running a single service locally

Replace the containerized service with a local process:

```bash
# Example: run navyr-auth locally instead of the Docker image
git clone https://github.com/navyr-io/navyr-auth
cd navyr-auth
go mod download

DATABASE_URL=postgres://navyr:navyr@localhost:5432/navyr \
  JWT_SECRET=dev-secret-32chars-minimum-here \
  PORT=8081 \
  go run ./cmd/server
```

Then update the gateway's `AUTH_SERVICE_URL` to point to `http://host.docker.internal:8081`.

## Environment variables reference

### navyr-gateway

```bash
PORT=8080
JWT_SECRET=dev-secret-32chars-minimum-here
INTERNAL_CONTEXT_SIGNING_SECRET=another-32char-secret
AUTH_SERVICE_URL=http://localhost:8081
BILLING_SERVICE_URL=http://localhost:8082
ORCHESTRATOR_SERVICE_URL=http://localhost:8083
CORS_ALLOWED_ORIGINS=http://localhost:5173
RATE_LIMIT_ENABLED=false
BILLING_ENFORCEMENT_MODE=log     # log|enforce|off
```

### navyr-auth

```bash
PORT=8081
DATABASE_URL=postgres://navyr:navyr@localhost:5432/navyr
JWT_SECRET=dev-secret-32chars-minimum-here
FRONTEND_URL=http://localhost:5173
CORS_ALLOWED_ORIGINS=http://localhost:5173
SMTP_ENABLED=false
AUTH_EXPOSE_INVITE_TOKEN=true    # dev only — returns token in response
AUTH_EXPOSE_RESET_TOKEN=true     # dev only — returns token in response
```

### navyr-orchestrator

```bash
PORT=8083
DATABASE_URL=postgres://navyr:navyr@localhost:5432/navyr
JWT_SECRET=dev-secret-32chars-minimum-here
INTERNAL_CONTEXT_SIGNING_SECRET=another-32char-secret
AUTH_SERVICE_URL=http://localhost:8081
CLUSTER_CREDENTIAL_ENCRYPTION_KEY=32chardevkeyforlocaldevelopment!
WS_EXEC_TICKET_SECRET=yet-another-32char-secret-here!!
CLUSTER_VALIDATION_MODE=lenient
CORS_ALLOWED_ORIGINS=http://localhost:5173
```

### navyr-frontend

```bash
# .env.local
VITE_API_URL=http://localhost:8080
```

## Test users

| Email | Password | Role | Org | Notes |
|---|---|---|---|---|
| `dev@clusterone.io` | `Dev@12345!` | org_admin | ClusterOne | Primary frontend test user |
| `admin@test.com` | `Test1234!` | org_admin | QA | Used for auth flow testing |
| `admin@navyr.io` | `Navyr@2026!` | org_admin | Navyr Demo | Has 3 test clusters (prod-primary, staging-eu, dev-local) |

## Running tests

```bash
# Backend — any service
go test ./...
go test ./... -v -run TestSpecificName

# Frontend — unit tests
cd navyr-frontend
npm run test

# Frontend — E2E (requires stack running)
npx playwright test
npx playwright test --headed  # with browser visible
npx playwright test --ui      # interactive test runner
```

## Database operations

```bash
# Connect to local postgres
psql postgres://navyr:navyr@localhost:5432/navyr

# Run migrations manually (any service)
cd navyr-auth
migrate -path migrations -database "$DATABASE_URL" up

# Roll back one step
migrate -path migrations -database "$DATABASE_URL" down 1

# Check migration status
migrate -path migrations -database "$DATABASE_URL" version
```

## Debugging the agent tunnel

When developing cluster operations locally, you need an agent connected:

```bash
# 1. Register a cluster in the UI (Clusters → Add Cluster)
# 2. Get the agent token from the response

# 3. Run the agent locally against your kind cluster
cd navyr-agent
ORCHESTRATOR_URL=ws://localhost:8083 \
  AGENT_SECRET=<TOKEN> \
  ORG_ID=<ORG_UUID> \
  CLUSTER_ID=<CLUSTER_UUID> \
  KUBECONFIG=$HOME/.kube/config \
  go run ./cmd/executor

# 4. The cluster should show as "healthy" in the UI within 30s
```

## Useful commands

```bash
# Check which clusters are connected (via DB)
psql $DATABASE_URL -c "SELECT id, name, last_agent_seen_at FROM clusters ORDER BY last_agent_seen_at DESC;"

# Tail gateway logs
docker logs -f navyr-gateway

# Tail all service logs
docker compose logs -f

# Restart a single service
docker compose restart navyr-orchestrator

# Rebuild and restart frontend (after code change)
cd navyr-frontend && npm run build
docker compose build frontend && docker compose up -d frontend
```

## Adding a new API endpoint

1. **navyr-orchestrator** (or appropriate service): add handler function in `internal/handler/`
2. Register the route in `cmd/server/main.go` with `mux.HandleFunc`
3. **navyr-gateway**: add the route to the proxy rules if it needs auth enforcement
4. **navyr-frontend**: add the API function in `src/lib/api/`
5. Update the service README endpoint table
6. Update this docs repo

## Code style

- Go: standard `gofmt` + `golint`; no generics where a simple interface suffices
- TypeScript: strict mode enabled; no `any` except at external API boundaries
- Commit messages: Conventional Commits (`feat:`, `fix:`, `docs:`, `refactor:`)
- No `bg-white` or hardcoded hex colors in frontend — use `var(--navyr-*)` tokens only
