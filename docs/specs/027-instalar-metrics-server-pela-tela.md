# SPEC-027 — O botão de instalar metrics-server não podia funcionar

**Card:** `navyr-io/navyr-frontend#33` · **Severidade:** Médio
**Motivada por:** SPEC-016, ao fazer `manifests/apply` aplicar de verdade

## O que o card supunha, e o que a medição mostrou

O card dizia: depois da SPEC-016 o botão passa a falhar com `403`, porque o
manifesto do metrics-server traz `ClusterRole`, `ClusterRoleBinding` e
`APIService`, que o agente não pode criar. Verdadeiro — **e é o menor dos
três problemas.**

### 1. O `fetch` é bloqueado por CORS, em qualquer instalação

O primeiro passo do botão era buscar o manifesto **do navegador**:

```ts
const resp = await fetch(METRICS_SERVER_URL);   // https://github.com/…/components.yaml
```

Medido:

```
$ curl -D- -L -H "Origin: http://localhost:5173" <url do manifesto>
HTTP/2 302  →  HTTP/2 302  →  HTTP/2 200
(nenhum access-control-allow-origin em ponto nenhum da cadeia)
```

O navegador recusa antes de qualquer coisa. Reproduzido na stack:
`TypeError: Failed to fetch`. O host e o contêiner alcançam o GitHub sem
problema (`HTTP 200` em 0,9 s) — o bloqueio é do navegador, e vale para
qualquer instalação, não só esta.

**O botão nunca funcionou.** Não é regressão da SPEC-016.

### 2. O agente não poderia aplicar, ainda que buscasse

O ClusterRole do agente concede `get,list,watch` — e só isso — em `roles`,
`rolebindings`, `clusterroles` e `clusterrolebindings`, e nada em
`apiregistration.k8s.io`.

Ampliar isso daria ao agente poder de criar binding de cluster, ou seja, de
**se auto-promover a cluster-admin**. É a fronteira que sustenta todo o modelo
de confiança do agente, inclusive a garantia registrada na SPEC-016 de que um
manifesto arbitrário não escala privilégio. Nenhuma conveniência de instalação
a justifica.

### 3. Contradiz um princípio que o produto testa

`tests/e2e/monaco-sem-cdn.spec.ts` garante que *"a aplicação não busca nada de
fora — editor nem fontes"*, e o comentário explica por quê: a versão servida
fica fora do controle do produto, a CSP precisa liberar um terceiro, e
**instalação air-gapped não funciona**.

Buscar `github.com` em runtime é exatamente o que esse spec existe para
impedir. Ele não pegou porque carrega telas e não aciona este botão.

### 4. E o botão nunca chegou a renderizar

`canApply` tem padrão `false`, e nenhum dos três chamadores o passa
(`ClusterWorkspacePage`, `NodesPage`, `NodeCard`). Era código morto, além de
quebrado.

## Decisão

**Remover o botão e o `fetch` externo.**

A opção "tratar o `403` com mensagem específica", que o card recomendava, parte
de uma premissa falsa: o `403` nunca chega, porque a requisição morre no CORS.
Tratar um erro que não acontece seria escrever código para um caminho
inexistente.

O caminho que **funciona** já estava na tela: os comandos `helm` e `kubectl`,
prontos para copiar, rodados pelo operador com a própria credencial. É o mesmo
que o produto faz para instalar o próprio agente, e é o único que respeita a
fronteira de privilégio.

## Regras

- **R1** — O componente não dispara requisição alguma ao renderizar.
- **R2** — Não há botão que prometa instalar pela plataforma.
- **R3** — Os comandos de instalação seguem em primeiro plano, com copiar.
- **R4** — Teste que falha se o botão ou o `fetch` externo voltarem.

## Critérios de aceitação

1. Renderizar o componente não emite requisição.
2. Não há botão de "apply" ou "install".
3. Os comandos `helm` e `kubectl` continuam disponíveis, com copiar.
4. Prova negativa: reintroduzindo o botão e o `fetch`, o teste falha.
