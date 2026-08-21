# Registros de decisão de arquitetura

Cada arquivo registra **uma decisão** e, principalmente, o que ela custou —
não o que ela promete. Um ADR sem consequência negativa registrada é um ADR
incompleto: a consequência é o que permite reavaliar quando o contexto mudar.

Formato: contexto, decisão, consequências, alternativas descartadas.

Decisão registrada não é decisão imutável. Quando uma for revertida, o ADR
antigo ganha status `Substituída por NNNN` em vez de ser apagado — o histórico
de por que se tentou o caminho anterior é o que evita repeti-lo.

| ADR | Decisão | Status |
|---|---|---|
| [0001](0001-agent-tunnel.md) | Acesso a cluster por túnel de saída, não por kubeconfig | Aceita |
| [0002](0002-criptografia-de-credenciais.md) | Envelope encryption com chave derivada por organização | Aceita |
| [0003](0003-multi-repo.md) | Um repositório por serviço, em vez de monorepo | Aceita |
| [0004](0004-modelo-de-edicoes.md) | Edições detectadas em runtime, com enforcement central | Aceita |
| [0005](0005-ci-validacao-separada-de-publicacao.md) | Validação e publicação em workflows distintos | Aceita |
| [0006](0006-helm-como-caminho-unico.md) | Helm como único caminho de deploy para Kubernetes | Aceita |
| [0007](0007-sem-dependencia-de-rede-externa.md) | A aplicação não busca nada de fora em runtime | Aceita |
| [0008](0008-autorizacao-fail-closed-e-cache-de-sessao.md) | Autorização fail-closed e cache de sessão | Proposta |
