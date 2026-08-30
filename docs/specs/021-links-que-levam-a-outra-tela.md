# SPEC-021 — Cinco links caem em outra tela, e um leva a tela em branco

**Card:** `navyr-io/navyr-frontend#25` · **Severidade:** Médio
**Descoberto:** varredura de interface ponta a ponta, Fase 3 (28/08/2026)

## Medido ao vivo

| Link oferecido | Alvo | O que acontecia |
|---|---|---|
| **ConfigMaps** | `cluster-config/configmaps` | caía no Overview, em silêncio |
| **Secrets** | `cluster-config/secrets` | caía no Overview, em silêncio |
| **ServiceAccounts** | `cluster-config/serviceaccounts` | caía no Overview, em silêncio |
| **PVCs** | `resources/pvcs` | caía em Storage Classes |
| **PVs** | `resources/pvs` | caía em Storage Classes |
| **"View trends"** (cockpit) | `clusters/{id}/observability` | **tela em branco** |

## Causa — e ela é maior que os seis links

Os componentes **ignoram o parâmetro `:section` da rota**:

- `ClusterConfigPage` lia só `useParams().id`; a seção era `useState("overview")`
  e nunca era sincronizada com `:section`.
- `ResourcesPage` nem chamava `useParams` para `section`.

Isso não quebrava só os links errados: quebrava **as oito seções que
funcionam**. Um marcador salvo em `cluster-config/crds` abria o Overview, e
trocar a seção pelo seletor não mexia na URL, então recarregar perdia a seção.

`/clusters/:id/observability` **não existe** no `router.tsx`. O botão que leva a
ela está em `ClusterWorkspacePage.tsx:141`, e o `main` renderizava zero
caracteres.

## Decisões

**D1 — A seção passa a vir da URL, nos dois componentes.**
É a correção de causa, não de sintoma: torna as rotas profundas
compartilháveis e recarregáveis, coisa que nunca funcionou.

**D2 — Os quatro links sem tela saem da navegação.**
ConfigMaps, Secrets, PVCs e PVs apontavam para seções que não existem em
`SECTIONS` nem em `ViewMode`.

O backend e o gateway **servem esses dados** — `/api/v1/configmaps`,
`/api/v1/secrets`, `/api/v1/persistentvolumeclaims` e
`/api/v1/persistentvolumes` estão na allowlist do gateway
(`rotas.go:60,62,122,128`) e o orchestrator os implementa. O que falta é a
tela, e ela vira card próprio.

Oferecer o link antes de a tela existir é pior que não oferecer: o operador
clica, vê outra coisa, e conclui que a plataforma está quebrada. Um link
ausente ele nem procura.

**D3 — ServiceAccounts muda de grupo, não de existência.**
A tela dele é `security/serviceaccounts`, que funciona hoje
(`listSecuritySection` já suporta a seção). O lugar dele é em ACCESS, junto de
Roles e RoleBindings, e não em CONFIGURATION.

**D4 — "View trends" aponta para a observabilidade da organização.**
`/{orgId}/observability` existe, e o cluster já está selecionado no contexto da
barra lateral. Criar uma tela de observabilidade por cluster seria
funcionalidade nova, não correção deste card.

**D5 — Seção inexistente na URL continua caindo no Overview.**
Mas agora isso é o tratamento de URL inválida — digitada ou vinda de um
marcador antigo — e não o destino de um link que a própria navegação oferece.

## Regras

- **R1** — `cluster-config/:section` abre a seção pedida; sem seção, o Overview.
- **R2** — `resources/:section` abre a visão pedida; sem seção, Storage Classes.
- **R3** — Trocar a seção pelo seletor muda a URL, para que recarregar e
  compartilhar funcionem.
- **R4** — Nenhum link da navegação aponta para rota que não renderiza.
- **R5** — Testes afirmam sobre o que a tela **renderiza**, não sobre estado
  interno: o defeito original era exatamente um estado que existia e não
  correspondia à URL.

## Critérios de aceitação

1. `cluster-config/crds` abre CRDs, com o seletor em `crds`.
2. `resources/generic` abre Generic Resources.
3. Nenhum link da barra lateral leva a outra tela.
4. "View trends" abre uma tela com conteúdo.
5. Prova negativa: voltando a ignorar `:section`, os testes falham.

## Cardado à parte

Telas de listagem para ConfigMaps, Secrets, PVCs e PVs — o dado já é servido
pelo backend e pelo gateway; falta a interface.
