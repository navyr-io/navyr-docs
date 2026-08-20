# Editions

**Last updated: 2026-08-19**

## Licensing

Navyr is **source-available under a commercial license** — see [LICENSE](../LICENSE).
Free use is granted for self-hosted, non-commercial, educational and open-source
scenarios; commercial redistribution and hosting-as-a-service require a separate
agreement. The full matrix of permitted scenarios is in [EDITIONS.md](../EDITIONS.md).

> **Open core — decidido e executado em 20/08.** LDAP, SSO, SCIM, grupos e
> grants são recursos da edição Enterprise, sob licença comercial
> (`navyr-auth/LICENSE-EE`). O núcleo segue sob a licença do repositório.
>
> A separação é de **compilação**, não de configuração: `go build -tags oss`
> produz um binário que não contém o código Enterprise. É verificável — zero
> símbolos de `go-ldap` contra 309 na edição completa. Chamada a recurso
> Enterprise na edição livre responde 402. Detalhes em
> `navyr-auth/EDITIONS.md`.
>
> Revisões anteriores desta página descreviam a edição livre como Apache 2.0.
> Nunca foi verdade: o código sempre saiu sob a Navyr Software License.

## Overview

Navyr ships as a single codebase with three editions. Edition capabilities are
detected at runtime based on the license key and environment configuration —
there are no separate branches or forks.

| Edition | Target | Distribution | Licensing |
|---|---|---|---|
| **Free** | Self-hosted, non-commercial, education, OSS projects | Docker Compose / Helm, no cost | Navyr Software License |
| **Enterprise** | Teams with compliance and SSO requirements | Self-hosted with EE license key | Navyr Software License, commercial terms |
| **SaaS** | Managed service | navyr.io hosted | Subscription |

## Feature matrix

| Feature | Free | Enterprise | SaaS |
|---|---|---|---|
| Cluster management (agent tunnel) | ✅ | ✅ | ✅ |
| Full Kubernetes CRUD | ✅ | ✅ | ✅ |
| Pod exec (WebSocket shell) | ✅ | ✅ | ✅ |
| Security scanning (Trivy, RBAC risk) | ✅ | ✅ | ✅ |
| Topology graph | ✅ | ✅ | ✅ |
| Operational Labs | ✅ | ✅ | ✅ |
| AIOps (incident plan, baseline) | ✅ | ✅ | ✅ |
| AI BYOK (bring your own key) | ✅ | ✅ | ✅ |
| Basic RBAC (4 roles) | ✅ | ✅ | ✅ |
| Audit log | ✅ | ✅ | ✅ |
| Community badges and leaderboard | ✅ | ✅ | ✅ |
| SSO (OIDC) | — | ✅ | ✅ |
| LDAP/AD sync | — | ✅ | ✅ |
| SCIM v2 provisioning | — | ✅ | ✅ |
| Groups + fine-grained grants | — | ✅ | ✅ |
| Approval workflows (dual-approval) | — | ✅ | ✅ |
| Outbound webhooks | — | ✅ | ✅ |
| Audit export (JSON/CSV) | — | ✅ | ✅ |
| Configurable audit retention | — | ✅ | ✅ |
| Cosign image verification | — | ✅ | ✅ |
| SBOM generation | — | ✅ | ✅ |
| AWS KMS credential encryption | — | ✅ | ✅ |
| Multi-region clusters | — | ✅ | ✅ |
| SLA support | — | ✅ | ✅ |
| Managed infrastructure | — | — | ✅ |
| Navyr-hosted AI (no BYOK required) | — | — | ✅ |

## Edition detection

The edition is determined at runtime by `navyr-auth` based on:

1. **License key** (environment variable `NAVYR_LICENSE_KEY`) — if absent or invalid, edition defaults to `oss`
2. **Feature flags** — individual features can be enabled/disabled independently of the edition (for gradual rollout or custom enterprise agreements)

The billing enforcement endpoint (`/billing/v1/enforcement/check`) gates feature access based on the org's resolved plan. The gateway calls this endpoint before every write operation.

## Running in OSS mode

No configuration required. Simply do not set `NAVYR_LICENSE_KEY`. All Enterprise features return `HTTP 403` with `code: PLAN_LIMIT`.

```bash
# OSS — no license key needed
docker compose up -d
```

## Running in Enterprise mode

```bash
# Set the license key in .env
NAVYR_LICENSE_KEY=<your-license-key>

# Or as an environment variable
docker run -e NAVYR_LICENSE_KEY=<key> ghcr.io/navyr-io/navyr-auth
```

The license key is validated by `navyr-auth` on startup. An invalid key logs a warning and falls back to OSS mode.

## Design principle

All editions are built from the same source. This means:

- There are no OSS-specific hacks or removed features — the full feature set is compiled into every image
- Enterprise features are gated by plan enforcement, not by different binaries
- Upgrading from OSS to Enterprise requires only setting a license key and restarting — no data migration needed
- Security fixes and improvements reach all editions simultaneously
