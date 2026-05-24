# Data model

**Last updated: 2026-05-24**

## Database topology

All services share a single PostgreSQL instance (in the default Docker Compose deployment). Each service manages its own schema via embedded migrations — there are no cross-service foreign keys.

| Service | Migration prefix | Table count |
|---|---|---|
| `navyr-auth` | `000001` – `000021` | ~30 tables |
| `navyr-billing` | `000001` – `000004` | ~8 tables |
| `navyr-orchestrator` | `000001` – `000008` | ~10 tables |
| `navyr-community` | `001_init`, `000011` | ~6 tables |

Migrations run automatically on service startup using an embedded migration library. The migration state is tracked in a `schema_migrations` table per service (or a named table to avoid conflicts).

In production, a dedicated database per service is recommended for isolation and independent scaling.

## navyr-auth schema

```
organizations
  id            UUID PK
  name          TEXT UNIQUE
  slug          TEXT UNIQUE
  created_at    TIMESTAMPTZ

users
  id            UUID PK
  org_id        UUID FK → organizations
  email         TEXT UNIQUE
  password_hash TEXT
  role          TEXT (owner|admin|operator|viewer)
  active        BOOLEAN
  created_at    TIMESTAMPTZ

sessions / refresh_tokens
  id            UUID PK
  user_id       UUID FK → users
  token_hash    TEXT
  expires_at    TIMESTAMPTZ
  revoked_at    TIMESTAMPTZ

auth_groups
  id            UUID PK
  org_id        UUID FK → organizations
  name          TEXT
  group_type    TEXT (local|ldap)
  ldap_group_dn TEXT

user_grants / group_grants
  id            UUID PK
  {user_id|group_id}  UUID FK
  cluster_id    UUID (nullable — org-wide if null)
  namespace     TEXT (nullable — cluster-wide if null)
  role          TEXT

sso_configs
  id            UUID PK
  org_id        UUID FK → organizations
  provider      TEXT (oidc|saml)
  config        JSONB (encrypted)

ai_provider_configs
  id            UUID PK
  org_id        UUID FK → organizations
  provider      TEXT (openai|anthropic|azure|bedrock|vertex|ollama)
  config        JSONB (api key and params — AES-GCM encrypted)

totp_configs
  user_id       UUID PK FK → users
  secret        TEXT (AES-GCM encrypted)
  enabled       BOOLEAN

ldap_configs
  org_id        UUID PK FK → organizations
  host          TEXT
  config        JSONB

auth_webhooks
  id            TEXT PK
  org_id        UUID FK → organizations
  url           TEXT
  secret        TEXT
  events        TEXT[]
  enabled       BOOLEAN
```

## navyr-billing schema

```
org_billing
  org_id        UUID PK
  plan          TEXT (oss|starter|pro|enterprise)
  status        TEXT (active|trial|suspended)
  limits        JSONB

usage_events
  id            UUID PK
  org_id        UUID
  feature       TEXT
  action        TEXT
  outcome       TEXT
  recorded_at   TIMESTAMPTZ

audit_events
  id            UUID PK
  org_id        UUID
  actor_id      UUID
  actor_role    TEXT
  actor_ip      TEXT
  action        TEXT
  resource_kind TEXT
  resource_name TEXT
  cluster_id    UUID
  namespace     TEXT
  outcome       TEXT
  recorded_at   TIMESTAMPTZ

audit_retention_policies
  org_id                UUID PK
  retention_days        INT
  last_purged_at        TIMESTAMPTZ
```

## navyr-orchestrator schema

```
clusters
  id                    UUID PK
  org_id                UUID
  name                  TEXT
  provider              TEXT
  region                TEXT
  connectivity_mode     TEXT (agent)    ← only mode since migration 000002
  credential_encrypted  BYTEA           ← AES-GCM(DEK(credential))
  dek_encrypted         BYTEA           ← KEK(DEK)
  kek_id                TEXT
  last_agent_seen_at    TIMESTAMPTZ     ← used to derive live status
  created_at            TIMESTAMPTZ

agent_tokens
  id                    UUID PK
  cluster_id            UUID FK → clusters
  token_hash            TEXT
  expires_at            TIMESTAMPTZ
  created_at            TIMESTAMPTZ

exec_audits
  id                    UUID PK
  cluster_id            UUID
  namespace             TEXT
  pod_name              TEXT
  container             TEXT
  command               TEXT
  actor_id              UUID
  actor_role            TEXT
  actor_ip              TEXT
  started_at            TIMESTAMPTZ
  ended_at              TIMESTAMPTZ

lab_sessions
  id                    UUID PK
  cluster_id            UUID
  lab_id                TEXT
  status                TEXT (running|passed|failed|stopped)
  helm_release          TEXT
  started_at            TIMESTAMPTZ
  resolved_at           TIMESTAMPTZ

action_approvals
  id                    UUID PK
  cluster_id            UUID
  action                TEXT
  requested_by          UUID
  approved_by           UUID
  status                TEXT (pending|approved|rejected|expired)
  payload               JSONB
  created_at            TIMESTAMPTZ
```

## navyr-community schema

```
community_users
  id            UUID PK
  github_id     TEXT UNIQUE
  username      TEXT
  avatar_url    TEXT
  created_at    TIMESTAMPTZ

badges
  id            UUID PK
  name          TEXT
  description   TEXT
  icon_url      TEXT

user_badges
  user_id       UUID FK → community_users
  badge_id      UUID FK → badges
  awarded_at    TIMESTAMPTZ

lab_completions
  user_id       UUID FK → community_users
  lab_id        TEXT
  completed_at  TIMESTAMPTZ

leaderboard_entries
  user_id       UUID FK → community_users
  labs_completed INT
  badges_earned  INT
  updated_at    TIMESTAMPTZ

community_events
  id            UUID PK
  user_id       UUID
  event_type    TEXT
  payload       JSONB
  created_at    TIMESTAMPTZ
```

## Migration strategy

Each service uses embedded migrations that run automatically on startup. The migration library applies pending migrations in order and tracks applied migrations in a state table.

**Rules:**
- Migrations are **never rolled back automatically** in production
- Every migration has a corresponding `.down.sql` file for manual rollback
- No cross-service migrations — each service owns its schema
- Migrations use `IF NOT EXISTS` guards to be idempotent where possible
- Breaking schema changes use multi-step migrations (add column → backfill → drop old column)

**Applying manually:**

```bash
migrate -path migrations -database "$DATABASE_URL" up
migrate -path migrations -database "$DATABASE_URL" down 1  # roll back one step
```
