# Runbooks de operação

Procedimento que alguém executa **sob pressão**, possivelmente às três da
manhã, possivelmente sem ser quem escreveu o sistema.

Regras destes documentos:

- Comando literal, copiável, sem placeholder que exija adivinhação.
- O que **verificar depois**, para saber se funcionou — um procedimento sem
  verificação é um procedimento que você torce para ter dado certo.
- O que **não** fazer, quando existe um caminho tentador e errado.

| Runbook | Quando usar |
|---|---|
| [Backup e restauração do banco](backup-restauracao.md) | Antes de migration destrutiva, e para recuperar |
| [Reverter uma release](reverter-release.md) | Deploy quebrou produção |
| [Serviço não fica pronto](servico-nao-fica-pronto.md) | Pod em 0/1, readiness reprovando |
| [Agente desconectado](agente-desconectado.md) | Cluster aparece registrado e não responde |
| [Publicar uma release](publicar-release.md) | Cortar uma versão e verificar que ela existe |
