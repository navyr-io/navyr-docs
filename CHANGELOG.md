# Changelog

Formato baseado em [Keep a Changelog](https://keepachangelog.com/pt-BR/1.1.0/),
versionamento em [SemVer](https://semver.org/lang/pt-BR/).

> Antes da 0.1.0 não havia release versionada: as imagens saíam apenas como
> `sha-<commit>` e `latest`, sem como saber o que estava no ar nem reverter
> para uma versão conhecida. Esta primeira entrada consolida o estado atual.

## [0.1.0] - 2026-08-19

### Documentação
- Initial navyr-docs — full technical architecture documentation
- Add integrated architecture diagram (full system Mermaid)
- Add product roadmap as source of truth
- Reorder phases — Intelligence Platform to 5, Enterprise to 6, Labs to 7 (site only)
- Phase 5 — Labs Kind Provisioner, FinOps, Intelligence WS
- Phase 6+7 — AIOps engine, intelligence WS bridge, lab tunnel fix
- Atualizar status — Phase 5/6/7 entregues pelo Codex
- Mark P0-8 and P0-9 as delivered (2026-05-25)
- Mark FinOps UI, AIOps UI and Security Compliance as delivered
- Add Phase 8 deliverables and Phase 9 backlog to roadmap
- Mark P2-10, P2-11, P1-CR2/CR4/CR5, F3-19 as delivered
- Update roadmap — P0-8 all screens ✅, P1 items ✅, Phase 9 D0+D1+D2+SEC-01 ✅
- Mark P1-13 status color semantics ✅ 2026-05-25
- Mark P1-6, P2-12 ✅; add P2-12 site verification entry
- Phase 9 ✅ complete — all D0-D10 delivered; SEC-02/03 deferred to Phase 10
- P2-14 ✅, E1 observability ✅, AIOps correlations UI ✅ — 2026-05-25
- Phase 10 ✅ complete — SEC-02/03 + D1-D6 delivered
- Phase 11 ✅ — Security Module + Enterprise Hardening complete
- Marca P1-18/26/32/33 entregues (B1-B4) + registra brief apply-binding
- Mark apply-binding D1/D2/D3 as delivered 2026-05-26
- P1-25/27 marked delivered, apply-binding D1/D2/D3 log updated
- P1-19/21/22/23/24/29/30/36/37 marked delivered
- Mark P1-40 done — /:orgId URL routing delivered 2026-05-26
- Registrar achados do E2E 2026-05-26 no backlog
- Close P1-44/45/46 V-01/02/03 P2-16 — sprint E2E fixes 2026-05-26
- Mark P1-47, P1-39, P1-48, P3-14 as delivered (2026-05-26)
- E2E vistoria 2026-05-26 — 10 bugs, 7 design issues, 11 security vulns
- Close P1-49..54 — E2E audit bugs fixed and investigated
- Close P1-55..58 — E2E bugs sprint 2 completo
- Registrar P1-40 extensão (UserMenu/SharedViewsPage) e P1-59 (AdminPage GroupDetail IAM)
- Resgata LICENSE, GOVERNANCE.md e EDITIONS.md do monorepo
- Corrige a contradição de licenciamento no repositório público
- Registra os achados em aberto e fecha o backlog SEC
- Substitui a meta de 60% de cobertura por critério de risco
- Registra a divisão dos arquivos-deus e o que ficou pendente
- Registra a divisão do auth_service.go
- Registra o fechamento do E2E e a cobertura perdida com as telas removidas
- Registra o fechamento da fase 4.2 e corrige a instrução de instalação
- Registra que o CI nunca rodou em PR do Dependabot
- Registra que o bump do Go quebrou as ferramentas de analise
- Registra os quatro defeitos da NetworkPolicy e a validacao que nao validava
- Adiciona licença, política de segurança, guia e CODEOWNERS
- Registra os arquivos de governanca como resolvidos
- ADRs das decisões estruturais e runbooks de incidente
- Indexa as specs OpenAPI em api.md
- ADR 0006 e registro do fim do caminho Kustomize
