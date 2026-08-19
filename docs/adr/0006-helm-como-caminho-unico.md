# 0006 — Helm como único caminho de deploy

**Status:** Aceita · **Data:** 2026-08

## Contexto

A mesma plataforma era descrita por **três** conjuntos de manifestos
independentes:

- o chart Helm `navyr-platform`;
- manifestos Kustomize em `navyr-helm/k8s/`, com overlays dev, staging e prod;
- o `docker-compose.yml` em `navyr-deploy`.

Três descrições da mesma coisa divergem — a questão é quando, não se. E já
haviam divergido em direções opostas, o que é o pior caso, porque cada uma
estava certa sobre algo:

- O **Kustomize** rodava o Postgres como uid 70 e tinha probes nos deployments.
  O Helm rodava como root e não tinha probes.
- O **Helm** tinha NetworkPolicy, PodDisruptionBudget, community, collector,
  redis e frontend. O Kustomize não tinha nenhum deles.

O Kustomize também estava mais desatualizado do que aparentava: os overlays
ainda geravam `kubeops-config`, nome trocado no chart, e a documentação
descrevia o overlay de produção como "production-hardened, PDB, resource
limits, NetworkPolicy" — nenhum dos quatro existia ali.

## Decisão

O Helm é o único caminho de deploy para Kubernetes. O Kustomize em `k8s/` foi
removido.

O Compose permanece, com escopo declarado: **desenvolvimento local e
demonstração**, não produção. Ele não compete com o Helm porque não descreve o
mesmo alvo.

Antes de remover, o que só existia no Kustomize foi portado ou preservado:

| Item | Destino |
|---|---|
| Probes nos deployments | Portado para o chart (4.2) |
| Postgres como uid 70 | Portado para o chart (4.2) |
| Manifestos do `navyr-site` | Preservados em `navyr-helm/site/` |
| `k8s/agent/` | Já era subconjunto do chart `navyr-agent`, que tem tudo isso mais NetworkPolicy, PDB e secrets |
| Overlays dev/staging/prod | Cobertos por `values.yaml`: `appEnv`, `clusterValidationMode`, `dbFallback` |

O `navyr-site` foi preservado **fora** do chart de propósito: é o site
institucional, com imagem, release e dependências próprias. Colocá-lo no chart
faria um `helm upgrade` da plataforma inteira ser necessário para trocar uma
página.

## Consequências

**O que ganhamos.** Uma descrição da plataforma. Correção de segurança feita
uma vez, não duas. Fim do risco de alguém instalar pelo caminho menos
endurecido sem saber que existia um melhor.

**O que custou:**

1. **Perdemos a instalação sem Helm.** Quem não quer Helm no cluster agora
   precisa de `helm template | kubectl apply -f -`, que funciona mas perde o
   histórico de release e o `helm rollback` — que é o procedimento do runbook
   de reverter release.

2. **Os overlays eram mais legíveis para diferença de ambiente.** Três linhas
   num `kustomization.yaml` são mais claras que a árvore de `values.yaml` de
   um chart. A troca vale porque a diferença real entre ambientes eram
   exatamente três variáveis — se voltar a crescer, o argumento enfraquece.

3. **O `navyr-site` ficou sem lar definitivo.** Está em `navyr-helm/site/`,
   mas pertence ao repositório `site`, junto do código que publica.

## Alternativas descartadas

**Manter os dois e sincronizar.** É o que se estava fazendo sem admitir, e
produziu a divergência do contexto. Sincronizar manualmente duas descrições
sem teste que compare as duas não funciona.

**Kustomize como único, gerando a partir do `helm template`.** Tecnicamente
viável, mas descarta o versionamento de release e o rollback do Helm, que o
runbook de incidente usa.

**Manter o Kustomize só para o `navyr-site`.** Manter a ferramenta inteira e
os overlays por causa de dois manifestos autocontidos não se paga.
