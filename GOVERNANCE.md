# Navyr — Governança de Engenharia

**Aplica-se a:** Claude Code, Codex e qualquer agente de desenvolvimento.  
**Versão:** 2026-05-18  
**Autoridade:** Erick (decisor final em qualquer conflito de escopo ou prioridade)

---

## Regra 1 — Definition of Done (mandatório)

Uma entrega SÓ é considerada concluída quando TODOS os itens abaixo estiverem satisfeitos:

| Critério | Obrigatório? | Exceção permitida? |
|---|---|---|
| Código compila sem erros | ✅ Sempre | Nunca |
| Testes unitários passam (`go test ./...` ou `npm test`) | ✅ Sempre | Documentar justificativa no commit |
| Build Docker do serviço alterado passa (`docker compose build <serviço>`) | ✅ Sempre | Nunca |
| Container sobe sem erro (`docker compose up -d <serviço>`) | ✅ Sempre | Nunca |
| Smoke test ou teste funcional básico executado e evidenciado | ✅ Sempre | Documentar justificativa no brief/status |
| Commit local criado com mensagem descritiva | ✅ Sempre | Nunca |

**Não existe "quase pronto". Se faltar um critério, não está pronto.**

---

## Regra 2 — Testes são obrigatórios em qualquer mudança de código

- Toda nova função de serviço (`auth_service`, `cluster_service`, etc.) deve ter ao menos um teste unitário.
- Toda nova rota de API deve ter ao menos um teste funcional ou de contrato.
- Toda mudança de comportamento existente deve ter o teste correspondente atualizado.
- Frontend: novos componentes devem ter pelo menos um render test básico quando aplicável.

**Se um agente entrega código sem teste, o entregável é rejeitado** até que o teste seja adicionado.

---

## Regra 3 — Containerização obrigatória

- Qualquer mudança em `services/*` exige rebuild do container correspondente.
- Qualquer mudança em `frontend/` exige rebuild do container `frontend`.
- Qualquer mudança em `gateway/` exige rebuild do container `gateway`.
- O agente deve executar `docker compose build <serviço>` e confirmar sucesso antes de fechar o brief.

**O ambiente de validação é sempre o Docker Compose local — nunca só o binário local.**

---

## Regra 4 — Escopo controlado pelo Erick

- Novos requisitos funcionais entram no `NAVYR.md` (backlog), não na sessão corrente.
- Nenhum agente altera o escopo de uma entrega sem aprovação explícita do Erick.
- Se um agente identificar trabalho adicional durante a implementação, ele documenta em `NAVYR.md` e continua o escopo original.
- Claude Code age como orquestrador: debate premissas, aponta riscos, mas executa a decisão do Erick.

---

## Regra 5 — Comunicação Claude Code ↔ Codex

O canal de comunicação é `.agents/`. Protocolo completo em `.agents/PROTOCOL.md`.

| Arquivo | Quem escreve | Quando |
|---|---|---|
| `claude-brief-YYYYMMDD.md` | Claude Code | Ao enviar tarefa para o Codex |
| `codex-status-YYYYMMDD.md` | Codex | Ao concluir qualquer tarefa |
| `claude-review-YYYYMMDD.md` | Claude Code | Ao validar entrega do Codex |
| `.agents/CURRENT.md` | Quem agiu por último | Sempre — resume estado atual |

**`NAVYR.md` é atualizado pelo Claude Code após cada sprint.**  
**`.agents/CURRENT.md` é operacional — para retomada de sessão.**

---

## Regra 6 — Arquitetura e contratos

- Nenhuma rota nova sem entrada no `spec/openapi.yaml`.
- Nenhuma migration sem ser registrada na lista de startup do serviço.
- Nenhuma feature flag ou backwards-compatibility shim sem justificativa documentada.
- Mudanças no modelo de dados exigem migration versionada (`000XXX_descricao.up.sql`).

---

## Regra 7 — Segurança

- Nenhum segredo hardcoded em código ou config.
- Nenhum endpoint sem autenticação exceto rotas públicas explicitamente designadas.
- Toda ação mutável crítica (exec, delete, cluster create) deve ter razão auditável (`X-Action-Reason`).
- Mudanças no gateway ou no auth-service exigem security review antes de merge.

---

## Regra 8 — Gestão do backlog

Quando o Erick trouxer um novo requisito:

1. Claude Code registra em `NAVYR.md` na prioridade correta.
2. Claude Code confirma com o Erick se é P0/P1/P2 ou deferido.
3. Se for P0, interrompe o sprint atual e realinha.
4. Se for P1/P2, entra na fila e não afeta o sprint corrente.

**Nenhum requisito novo entra em implementação imediata sem passar por esse fluxo.**

---

## Checklist pré-entrega (Claude Code e Codex)

Antes de declarar uma tarefa concluída:

```
[ ] go build ./... ou npm run build — sem erros
[ ] go test ./... ou npx playwright test — passando
[ ] docker compose build <serviço alterado> — sucesso
[ ] docker compose up -d <serviço> — container sobe
[ ] Smoke: requisição básica ao endpoint novo retorna 2xx
[ ] Commit criado com mensagem descritiva
[ ] NAVYR.md atualizado (se item do backlog foi fechado)
[ ] .agents/CURRENT.md atualizado
```
