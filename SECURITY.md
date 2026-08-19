# Política de segurança

## Reportar uma vulnerabilidade

Não abra issue pública para vulnerabilidade. Envie para **security@navyr.io**
com descrição, passos de reprodução e impacto observado.

Retorno inicial em até 5 dias úteis. Pedimos 90 dias antes de divulgação
pública, ou menos se a correção sair antes.

## Versões suportadas

Correções de segurança são aplicadas na branch `main` de cada repositório e
publicadas nas imagens `ghcr.io/navyr-io/*`. Não há suporte retroativo a tags
antigas.

## Escopo

Em escopo: os serviços da plataforma, o agente in-cluster, o chart Helm e o
manifesto de onboarding.

Fora de escopo: os charts em `navyr-helm/labs/`, que **contêm
vulnerabilidades por design** — são cenários de falha usados pelos Operational
Labs para ensinar diagnóstico. Container privilegiado, secret em variável de
ambiente e RBAC excessivo ali são intencionais e documentados.
