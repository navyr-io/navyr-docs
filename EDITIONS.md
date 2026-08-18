# Navyr — Editions & Licensing

## Modalidades

| Modalidade | Para quem | Custo |
|---|---|---|
| **Free** | Self-hosted, não comercial, educação pública, OSS | Gratuito |
| **Enterprise (EE)** | Empresas que self-hostam com features avançadas | Licença por org/cluster |
| **Partner** | Consultorias e MSPs que redistribuem ou hospedam para clientes | Licença de parceria |
| **Royalty** | Cloud providers e ISVs que vendem como SaaS/managed service | Royalty por uso |

---

## Casos de uso permitidos por modalidade

| Cenário | Free | EE | Partner | Royalty |
|---|---|---|---|---|
| Desenvolvedor individual (local, estudo) | ✅ | — | — | — |
| Startup / empresa (uso interno self-hosted) | ✅ | — | — | — |
| Enterprise self-hosted com features avançadas | — | ✅ | — | — |
| Consultoria instala na infra do cliente | ✅ | — | — | — |
| Consultoria hospeda e vende acesso para clientes | — | — | ✅ | — |
| MSP de K8s (Navyr como painel do serviço) | — | — | ✅ | — |
| ISV redistribui para cliente self-hostar | — | — | ✅ | — |
| ISV hospeda e vende como SaaS | — | — | — | ✅ |
| Cloud provider (AWS, GCP, Azure, DigitalOcean) | — | — | — | ✅ |
| Fork concorrente | ❌ | ❌ | ❌ | ❌ |
| Universidade pública / pesquisa sem fins lucrativos | ✅ | — | — | — |
| Repartição pública, órgão estatal, administração federal/estadual/municipal | — | ✅ | ✅ | — |
| Banco público, empresa estatal | — | ✅ | ✅ | — |
| Universidade privada com fins lucrativos | — | ✅ | — | — |
| Projeto OSS / comunidade | ✅ | — | — | — |
| Contributor Partner (contribui ativamente) | — | — | ✅ desconto | — |

---

## Features por edição

### Autenticação e identidade

| Feature | Free | EE | Partner/Royalty |
|---|---|---|---|
| Login email + senha | ✅ | ✅ | ✅ |
| Reset de senha | ✅ | ✅ | ✅ |
| Convites de usuário | ✅ | ✅ | ✅ |
| Sessões e refresh tokens | ✅ | ✅ | ✅ |
| TOTP / 2FA (obrigatório para root account) | ✅ | ✅ | ✅ |
| SSO / OIDC / SAML | ❌ | ✅ | ✅ |
| LDAP / Active Directory | ❌ | ✅ | ✅ |
| SCIM (provisionamento automático) | ❌ | ✅ | ✅ |
| FIDO2 / WebAuthn | ❌ | ✅ | ✅ |
| API Keys (automação / CI-CD) | ❌ | ✅ | ✅ |

### Clusters e conectividade

| Feature | Free | EE | Partner/Royalty |
|---|---|---|---|
| Conexão direta (kubeconfig) | ✅ | ✅ | ✅ |
| Agente (tunnel WebSocket) | ✅ | ✅ | ✅ |
| Multi-cluster | ✅ | ✅ | ✅ |
| Agent token com TTL e renovação automática | ✅ | ✅ | ✅ |
| KMS / envelope encryption de credenciais | ❌ | ✅ | ✅ |
| HA deployment do agente | ❌ | ✅ | ✅ |
| Air-gapped deployment | ❌ | ✅ | ✅ |

### Operações K8s (core)

| Feature | Free | EE | Partner/Royalty |
|---|---|---|---|
| Pods — list, logs, exec, delete | ✅ | ✅ | ✅ |
| Deployments — list, scale, rollout, rollback | ✅ | ✅ | ✅ |
| StatefulSets, DaemonSets, ReplicaSets | ✅ | ✅ | ✅ |
| Jobs e CronJobs | ✅ | ✅ | ✅ |
| Nodes — list, pressão, métricas | ✅ | ✅ | ✅ |
| Namespaces, Services, Ingresses | ✅ | ✅ | ✅ |
| ConfigMaps, Secrets (read-only) | ✅ | ✅ | ✅ |
| PVCs e Volumes | ✅ | ✅ | ✅ |
| Shell interativo no pod | ✅ | ✅ | ✅ |
| Events do cluster | ✅ | ✅ | ✅ |
| Topology graph | ✅ | ✅ | ✅ |

### Segurança

| Feature | Free | EE | Partner/Royalty |
|---|---|---|---|
| Image scan via Trivy (básico) | ✅ | ✅ | ✅ |
| Config risk — Polaris/Kubescape (básico) | ✅ | ✅ | ✅ |
| RBAC risk analysis | ✅ | ✅ | ✅ |
| Workload risk score | ✅ | ✅ | ✅ |
| Falco runtime security events | ❌ | ✅ | ✅ |
| SBOM + Cosign verify | ❌ | ✅ | ✅ |
| AI Security Insights | ❌ | ✅ | ✅ |

### Governança e controle

| Feature | Free | EE | Partner/Royalty |
|---|---|---|---|
| RBAC interno (admin / member / viewer) | ✅ | ✅ | ✅ |
| Grupos de usuários + grants por cluster/namespace | ❌ | ✅ | ✅ |
| Approval workflows (dupla aprovação) | ❌ | ✅ | ✅ |
| Audit log (básico) | ✅ | ✅ | ✅ |
| Audit retention policy e export | ❌ | ✅ | ✅ |
| Rate limiting | ✅ | ✅ | ✅ |

### Observabilidade e AI Ops

| Feature | Free | EE | Partner/Royalty |
|---|---|---|---|
| Métricas CPU/memory em tempo real | ✅ | ✅ | ✅ |
| Observability page | ✅ | ✅ | ✅ |
| AI Ops (análise básica) | ✅ | ✅ | ✅ |
| AI BYOK (OpenAI, Claude, Bedrock, Azure) | ❌ | ✅ | ✅ |

### Plataforma e comunidade

| Feature | Free | EE | Partner/Royalty |
|---|---|---|---|
| Operational Labs | ✅ | ✅ | ✅ |
| Community badges | ✅ | ✅ | ✅ |
| Hosted SaaS (Navyr Cloud) | — | — | — |
| Suporte SLA | ❌ | ✅ | ✅ |
| Acesso antecipado ao roadmap | ❌ | ✅ | ✅ |

---

## Contributor Partner

Empresas que contribuem ativamente com o projeto (PRs aceitos, bugs relatados, documentação) têm acesso ao programa **Contributor Partner**:

- Desconto na licença Partner proporcional à contribuição
- Co-marketing: logo no site e repositório
- Acesso antecipado a features EE
- Voz no roadmap trimestral

Critério de elegibilidade: avaliado trimestralmente pela equipe Navyr.

---

## Licenciamento

Ver `LICENSE` para os termos completos.
Para licenças EE, Partner ou Royalty: **contact@navyr.io**
