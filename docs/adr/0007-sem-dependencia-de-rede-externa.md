# 0007 — A aplicação não busca nada de fora

**Status:** Aceita · **Data:** 2026-08

## Contexto

O frontend dependia de dois terceiros em runtime:

- o editor de manifesto era baixado da **jsdelivr**, porque o
  `@monaco-editor/react` busca o Monaco na CDN quando ninguém configura o
  loader;
- as três famílias tipográficas vinham do **Google Fonts**.

Três consequências, e a terceira é a que decide:

1. **A versão servida ficava fora do nosso controle.** O `package.json` pedia
   `monaco-editor@0.56.0` e a CDN entregava `0.55.1`. Ninguém tinha como notar.
2. **A navegação do usuário era exposta a terceiros.** Toda sessão da
   plataforma gerava requisição para dois domínios que não são nossos, com
   `Referer` do nosso host.
3. **Deploy air-gapped não funcionava** — e é objetivo declarado do roadmap.
   Um cluster sem saída para a internet renderizava sem fonte e sem editor.

## Decisão

Nenhuma dependência de rede externa em runtime.

**Monaco** vem do pacote local, com import seletivo — `editor.api` mais o
registro de YAML, única linguagem usada — e carregado sob demanda via
`React.lazy`.

**Fontes** são embarcadas no repositório, em `src/assets/fonts/`, com
`@font-face` explícito. Variáveis, subsets `latin` e `latin-ext`.

## Consequências

**O que ganhamos.** Air-gapped funciona. A versão do editor é a declarada. A
CSP pode negar todo host externo, o que transforma uma regra de confiança numa
regra de rede. E nenhum dado de navegação sai para terceiros.

**O que custou:**

1. **A divisão do bundle deixou de ser opcional.** Trazer o Monaco para dentro
   sem dividir levaria o bundle principal de 1,95 MB para 4,89 MB — toda a
   aplicação baixando o editor, inclusive quem nunca abre a aba de YAML. Com
   `React.lazy`: principal em 1,85 MB e editor em chunk de 2,6 MB, buscado ao
   abrir a aba.

2. **As fontes foram vendorizadas, não instaladas por npm.** O `rolldown`, que
   é o bundler do Vite 8, não resolve o mapa de `exports` com wildcard dos
   pacotes `@fontsource-variable`. Vendorizar resolveu e permitiu embarcar só
   os subsets necessários — 320 KB contra 436 KB dos pacotes completos — ao
   custo de o Dependabot não conseguir atualizá-las. Fonte muda pouco; a
   origem e a versão estão no cabeçalho do CSS.

3. **Licença de fonte agora é nossa responsabilidade.** Inter e JetBrains Mono
   são SIL OFL 1.1, que permite redistribuição embarcada e exige o aviso de
   licença junto. Os dois arquivos estão em `src/assets/fonts/`.

## Alternativas descartadas

**Manter a CDN e liberar na CSP.** É o estado anterior. Não resolve o
air-gapped, que é o requisito que motivou a mudança.

**Servir a CDN a partir de um proxy nosso.** Resolveria versão e privacidade,
mas exige um serviço a mais no caminho de renderização e continua sem funcionar
sem rede.

**Fontes do sistema, sem embarcar nada.** Elimina o problema por completo e
custa a identidade visual — a interface passaria a ter aparência diferente em
cada sistema operacional.

## Como isso não regride

`navyr-frontend/tests/e2e/monaco-sem-cdn.spec.ts` observa os pedidos do
navegador e reprova em **qualquer** requisição para fora. Verificar por busca no
bundle não serviria: a URL da jsdelivr continua lá como constante padrão da
biblioteca, então o `grep` daria falso positivo.
