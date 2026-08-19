# 0004 — Edições detectadas em runtime, com enforcement central

**Status:** Aceita · **Data:** 2026-05

## Contexto

O produto tem edições — OSS, Enterprise e SaaS, mais o add-on de IA na nuvem —
com recursos distintos. Era preciso decidir como o binário sabe o que pode
fazer.

## Decisão

**Um único artefato por serviço**, igual para todas as edições. A edição é
detectada em runtime a partir de `NAVYR_LICENSE_KEY` e da configuração de
ambiente.

A verificação de direito é **centralizada**: os serviços consultam
`billing /enforcement/check` em vez de cada um interpretar a licença. Um lugar
para mudar regra de plano.

## Consequências

**O que ganhamos.** Um build, um scan, uma imagem publicada por serviço.
Mudança de plano não exige redeploy. A regra de enforcement muda em um serviço,
não em cinco.

**O que custou:**

1. **O código de todas as edições está em todo lugar.** Não há separação
   física entre núcleo e módulo enterprise. Isso torna a separação open core
   um trabalho de extração, não de configuração — e é justamente a decisão que
   segue em aberto no [editions.md](../editions.md).

   O fatiamento do `auth_service.go` em agosto reduziu esse custo sem querer:
   LDAP, SSO, TOTP, grupos e convites saíram para arquivos próprios, o que
   aproxima os candidatos naturais a `ee/` de estarem isolados.

2. **O billing vira dependência de disponibilidade para todo mundo.** Se ele
   cai, o enforcement precisa decidir entre negar tudo — e derrubar o produto —
   ou liberar tudo, e virar bypass. O comportamento de degradação é a parte
   mais delicada dessa decisão e merece revisão própria.

3. **A verificação é observável e adulterável por quem controla o ambiente.**
   Numa instalação self-hosted, o operador tem o binário e a rede. Detecção em
   runtime não é proteção contra o cliente: é conveniência operacional
   apoiada em contrato, não em criptografia.

## Alternativas descartadas

**Build por edição.** Elimina o item 3 e ajuda no item 1, mas multiplica
artefato, scan e superfície de publicação por edição — e um bug corrigido
precisaria de N builds.

**Licença verificada offline por assinatura.** Mais robusta contra adulteração,
porém exige distribuir chave pública e resolver revogação sem rede. Continua
sendo o caminho se o item 3 virar problema comercial real.

**Diretório `ee/` desde o início**, no estilo GitLab. Teria evitado o item 1.
Não foi feito, e o custo de fazer agora é a extração descrita acima.
