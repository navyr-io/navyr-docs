# SPEC-022 — Controles que parecem funcionar e não fazem nada

**Cards:** `navyr-io/navyr-frontend#26` e `#27` · **Severidade:** Médio
**Descoberto:** varredura de interface ponta a ponta, Fase 4 (28/08/2026)

## O defeito

Dois sintomas da mesma doença: a interface afirma capacidade que não tem.

### `#26` — nove controles clicáveis sem handler

Medido lendo `__reactProps$.onClick` do elemento no DOM. Sem handler, o clique é
inerte — e isso **não aparece em leitura estática do JSX** quando o botão é
montado por um wrapper, o que foi o motivo de a medição precisar ser feita no
navegador.

Nenhum sinal visual os distinguia de um botão funcional: `disabled` ausente,
`cursor: pointer`, opacidade 1, sem `title`.

### `#27` — a aba Features afirma três coisas incompatíveis

Presentes ao mesmo tempo, a poucos centímetros:

> *"Disabled modules are **hidden from all members** of your organization"*
> *"Changes are **applied immediately** to all members"*
> *"**Backend persistence coming** in Enterprise edition"*

Medido: **8 toggles, 0 requisições** ao alternar. São `useState` puro; recarregar
descarta.

O defeito não é a funcionalidade em construção — é a **contradição**. Quem lê um
painel de configuração lê o rótulo do controle, não a nota de rodapé.

## Decisão

Resolver cada controle pelo caminho mais barato que seja **honesto**: implementar
quando o backend existe, desabilitar com explicação quando não existe.

O produto já tem o padrão honesto — o "Export JSON" da tela de Security usa
`cursor: not-allowed`, opacidade reduzida e `title`. Faltava aplicá-lo, e
faltava o `disabled`, sem o qual o botão continua clicável e não é anunciado
como indisponível.

| Controle | Resolução | Por quê |
|---|---|---|
| **Re-scan** (Security) | **implementado** | refaz a consulta da aba ativa; `refetch` do react-query é literalmente o que "re-scan" significa aqui |
| AI Analysis (Security) | desabilitado | não existe rota de análise de segurança por IA; `lib/api/ai.ts` só tem copiloto de incidente e linha de base do AIOps |
| Export JSON (Security) | `disabled` acrescentado | já tinha `title` e opacidade, mas continuava clicável |
| CTA de Network | desabilitado | não há formulário de criação para Service, Ingress, Gateway ou Policy |
| Ask AI (Topology) | desabilitado | o painel existe (`NavyAnalysis.tsx`) mas é estado local do `AppShell`, não exposto por contexto; ligar exigiria elevar esse estado |
| Upgrade (Billing ×2) | desabilitado | não há rota de upgrade, checkout ou assinatura em lugar nenhum do `navyr-billing` |
| Notify me (AI Providers) | desabilitado | não há lista de notificação; o rótulo ao lado já dizia "Coming soon", o botão prometia mais |
| 8 toggles de Features | desabilitados | sem backend de controle de módulo |

**Descartado: remover os controles.** Um botão desabilitado com explicação diz ao
operador que a capacidade está prevista e por que não está disponível. Um botão
ausente não diz nada, e a intenção do produto se perde.

**Descartado: implementar Scale no inspetor de Workloads.** A API existe
(`scaleDeployment`) e há tela que a usa (`ActionsPage`), mas construir o
formulário no inspetor é funcionalidade nova, não correção deste card.

## Regras

- **R1** — Nenhum controle sem handler aparece habilitado.
- **R2** — Todo controle desabilitado carrega `title` dizendo **por que**, não
  apenas que está desabilitado.
- **R3** — A aba Features não afirma esconder módulo nem aplicar mudança.
- **R4** — Os toggles inertes carregam `role="switch"`, `aria-checked`,
  `aria-disabled` e `tabIndex={-1}` — antes não tinham papel ARIA nenhum.
- **R5** — Testes afirmam sobre o DOM renderizado, e um deles reproduz a medição
  original: clicar num toggle **não emite requisição** e não muda o estado.

## Critérios de aceitação

1. "Re-scan" refaz a consulta da aba ativa e mostra o estado de carregamento.
2. Os oito toggles de Features aparecem desabilitados, e clicar não faz nada.
3. A tela de Features não contém nenhuma das três afirmações antigas.
4. O CTA de Network está desabilitado e diz por quê.
5. Prova negativa: reabilitando cada controle, os testes falham.
