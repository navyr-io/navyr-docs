# 0001 — Acesso a cluster por túnel de saída

**Status:** Aceita · **Data:** 2026-05

## Contexto

A plataforma precisa executar operações na API do Kubernetes de clusters que
não são nossos. O caminho convencional é o cliente entregar um kubeconfig, que
a plataforma armazena e usa para conectar de fora para dentro.

Esse caminho tem três custos que não são negociáveis para o cliente: credencial
de administrador de cluster guardada por terceiro, endpoint da API exposto à
internet ou a uma faixa nossa, e regra de firewall de entrada no perímetro
dele.

## Decisão

O cluster do cliente roda um agente que abre uma conexão **de saída** por
WebSocket até o orchestrator. A plataforma nunca inicia conexão para o cluster.

`tunnel.Registry` mapeia `clusterID → *Conn`. O ponto que faz a decisão
barata é que `Conn` implementa `http.RoundTripper`: todo o código `client-go`
existente funciona sobre o túnel sem saber que ele existe. Não houve
reescrita de camada de acesso.

## Consequências

**O que ganhamos.** Nenhum kubeconfig armazenado. Nenhuma regra de entrada no
firewall do cliente. O endpoint da API pode continuar privado.

**O que custou.** A disponibilidade da operação passa a depender de uma conexão
viva. Uma conexão que morre em silêncio — sem FIN, sem RST — deixa o cluster
aparentemente registrado e completamente inoperante. Foi um defeito real:
corrigido com `SetReadDeadline` e ping periódico, com os prazos como campos da
`Conn` em vez de globais do pacote, porque `-race` mostrou a corrida quando o
teste os mutava.

Escalar o orchestrator horizontalmente fica mais difícil: a conexão é stateful
e vive num processo específico. Hoje isso não morde, e vai morder quando o HPA
do orchestrator entrar em uso real.

Depurar é mais difícil que HTTP direto: não há como reproduzir a chamada com
`curl` a partir da máquina de quem investiga.

## Alternativas descartadas

**Kubeconfig armazenado.** Descartada pelos três custos do contexto. É o que a
concorrência faz, e é o principal argumento de venda contra ela.

**VPN ou peering por cliente.** Resolve o firewall, mas transfere para o
cliente o custo de manter uma VPN por fornecedor, e não elimina a credencial.

**Operador que só lê e reporta.** Elimina a conexão viva, mas a plataforma
deixa de poder **agir** — e agir é o produto. O ciclo é observar, interpretar e
agir; sem o terceiro passo sobra um painel.
