# SPEC-007 — Dado fabricado na tela de Security, com rótulo que explica a causa errada

**Estado:** implementada e verificada — 27/08
**Card:** navyr-io/navyr-frontend#21

## Problema

Duas abas da tela de Security renderizam dados de segurança inventados quando a
chamada à API falha.

`src/screens/SecurityIntelligencePage.tsx:852` e `:972`:

```ts
const data = policyQ.data ?? (policyQ.error ? MOCK_POLICY : null);
const isPreview = !policyQ.data && Boolean(policyQ.error);
```

O dado é rotulado — existe um banner `PREVIEW — showing mock data`. **Não é troca
silenciosa**, e é por isso que esta spec trata de um defeito menor do que o card
originalmente descrevia.

## O defeito

**O rótulo afirma uma causa que deixou de existir.**

```
PREVIEW  Policy violations endpoint arrives in Phase 11 — showing mock data.
```

O comentário no código declara a intenção: `// Mock data — rendered until Phase 11
B1/B2 endpoints are live`. Os endpoints estão vivos — medido com sessão válida e
`X-Cluster-ID`, `security/policy-violations` e `security/compliance/framework`
respondem **200**, e a aba mostra o estado vazio correto.

O gatilho do mock deixou de ser "o endpoint não existe" e passou a ser "a chamada
falhou": 5xx transitório, timeout, sessão expirada, cluster sem agente. Em todos
esses casos o usuário lê que a funcionalidade ainda não foi construída, conclui
"não está pronto", e não investiga o que quebrou.

É diagnóstico errado dirigido ao usuário, em caminho de segurança. O andaime
sobreviveu ao motivo que o justificava.

Secundariamente: o dado fabricado é plausível — `production/api-server`,
`monitoring/node-exporter`, resumo `14 total, 2 critical`, controles CIS com
status `fail`. O rótulo protege quem olha a tela naquele instante; não protege o
recorte de tela nem o relatório que sai dali.

## O tratamento correto já existe na mesma tela

Duas outras abas tratam erro sem inventar dado — linhas 587 e 678:

```tsx
) : complianceQ.error ? (
  <div style={{...}}>
    <p>Compliance scoring unavailable</p>
    <p>Requires cluster agent access.</p>
  </div>
```

Não é padrão novo a inventar; é consistência com o que está ao lado.

## Regras

**R1 — Erro renderiza estado de erro, nunca dado.**
As duas abas passam a seguir a forma das linhas 587 e 678.

**R2 — `MOCK_POLICY` e `MOCK_FRAMEWORKS` são removidos.**
São as únicas ocorrências de `MOCK_` em todo o `src/`. Deixar as constantes sem
uso convidaria a religá-las.

**R3 — O banner `PREVIEW` sai junto.**
Sem mock, não há o que rotular. Manter o banner com dado real seria a mentira
inversa.

**R4 — O texto do erro diz o que fazer, não só o que falhou.**
As duas abas existentes apontam a causa provável ("Requires cluster agent access").
As novas seguem, apontando a causa provável de cada uma.

## Critérios de aceitação

1. `grep -rn 'MOCK_' src/` não retorna nada.
2. Com as chamadas falhando, as abas Policy e Frameworks mostram estado de erro —
   nenhum nome de workload, namespace ou contador diferente de zero.
3. Com as chamadas em 200, as abas seguem renderizando o dado real. Verificável na
   aba Policy, que hoje mostra "No policy engine detected".
4. Nenhuma string "Phase 11" permanece na tela.

## Fora de escopo

- As outras seis abas da tela, que já tratam erro corretamente ou não têm mock.
- Um modo de demonstração com dado de exemplo. Se o produto quiser isso, é
  funcionalidade com decisão própria — não um fallback de erro.
