# SPEC-034 — Contrato de empacotamento antes do terceiro alvo

**Estado:** parcialmente recusada — decidido em 18/09/2026

> **A recomendação central desta spec foi recusada.** Ela propunha partir o card
> em dois e deixar o módulo ECS fora da v0.1.0, pelo princípio de 21/08 ("tudo
> que depende de custo segura"). Erick decidiu em 18/09 que **o ECS entra na
> v0.1.0**, sobrepondo esse princípio para este item.
>
> O que a spec entregou e segue valendo: o contrato de empacotamento
> (`contrato/plataforma.yaml`), que existe, é verificado em CI e virou
> pré-requisito cumprido — e que no caminho provou o seu valor três vezes,
> achando defeitos que nenhum dos dois empacotamentos anteriores expunha:
>
> | Achado | Onde |
> |---|---|
> | `frontend.saude` era `null`, o nginx serve `/healthz` | navyr-deploy e6c6f27 |
> | `AUTH_UPSTREAM`/`GATEWAY_UPSTREAM` invisíveis ao contrato | navyr-deploy e6c6f27 |
> | 14 variáveis lidas por função auxiliar, invisíveis à varredura de `os.Getenv` | navyr-deploy 98b209f |
>
> A ressalva operacional que esta spec registrou também segue valendo, e está
> agora escrita na documentação de instalação: com `desiredCount: 1` no Fargate,
> **todo deploy derruba o plano de controle**.
**Data:** 16/09/2026
**Card:** navyr-io/navyr-deploy#17
**ADR:** 0006 (empacotamentos que divergem)
**Relacionada:** navyr-helm#6 (a catraca, concluída), SPEC-033 (ponto de entrada público)

## 1. Problema

Hoje há dois caminhos de instalação — Helm e Docker Compose — e falta o terceiro:
**ECS**, para quem tem só o cluster de produção e não quer o plano de controle
dentro dele.

A vantagem que só o ECS dá é real e vale registrar: **o plano de controle não
roda na coisa que ele gerencia**. Rodar o Navyr no Kubernetes cria a dependência
circular clássica — o dia em que a ferramenta é mais necessária é o dia em que
ela pode estar fora.

O risco, também real, é o da ADR 0006: um terceiro artefato **escrito à mão**
somado a um par que já diverge não dá três opções, dá três descrições que
discordam, e o cliente descobre qual está errada em produção. Não é hipótese —
o `navyr-helm#6` mediu **15 variáveis** que o código lê, presentes no compose e
ausentes do chart, incluindo uma que impedia o auth de subir com
`APP_ENV=production`.

### O que já existe, medido em 16/09

| Item | Estado |
|---|---|
| Serviços no compose | **9** (`db`, `redis`, `auth`, `billing`, `orchestrator`, `community`, `collector`, `gateway`, `frontend`) |
| Volumes | **1** (`postgres_data`) |
| Redes | **1** (`navyr`) |
| Serviços com `depends_on` | **7** |
| Catraca de contrato | `tests/contract/` — `test_empacotamentos_cobrem_o_codigo.py`, `test_env_example_cobre_o_compose.py`, `test_openapi_contract.py` |

A semente do contrato **já está construída**: é o que o `navyr-helm#6` entregou.

## 2. A recomendação técnica: partir o card em dois

O card pede duas coisas de naturezas opostas, e juntá-las é o que o torna caro
sem necessidade:

1. **O contrato legível por máquina** — quais serviços existem, quais variáveis
   cada um lê, o que depende de quê — com um teste por empacotamento que falha
   quando ele não cobre o contrato.
2. **O módulo Terraform de ECS** — 600 a 900 linhas, 7 task definitions, ALB,
   Cloud Map, RDS, ElastiCache, Secrets Manager, IAM, log groups.

A recomendação, pelos três eixos:

### Custo — o item 1 é grátis; o item 2 é recorrente e não tem demanda

Erick decidiu em 21/08: **tudo que depende de custo segura.** O item 2 é
exatamente isso. Fargate + RDS + ElastiCache é infraestrutura sempre ligada, e
validar a spec exige uma conta AWS real rodando a stack inteira — o critério de
aceitação do card é "instalação do zero numa conta AWS limpa".

O item 1 não custa nada e **paga hoje**: ele protege os dois empacotamentos que
já existem e são usados.

### Escalabilidade — e uma ressalva ao argumento do card

O card afirma que o teto de instância única "não atrapalha aqui", porque
`tunnel.Registry` é `map[uuid.UUID]*Conn` em memória do processo e o limite vale
em qualquer plataforma. Isso está correto.

A ressalva é operacional, não de escala: com `desiredCount: 1` no Fargate,
**todo deploy derruba o plano de controle** — a task é substituída, o WebSocket
cai, e todos os agentes reconectam. No laptop e no on-premise isso é aceitável.
Num caminho que o cliente paga para ter disponível, é uma característica que
precisa estar escrita antes de ser vendida, não descoberta no primeiro rollout.

O card já nomeia o que resolveria: **tirar o `Registry` da memória é o item de
maior alavanca que existe**. Enquanto ele não sair, o ECS entrega a vantagem de
isolamento com um custo de disponibilidade que o Helm e o Compose não cobram
porque ninguém espera HA deles.

### Segurança — o item 1 melhora, o item 2 amplia superfície

O contrato torna explícito quais variáveis são segredo e quais não, o que hoje
está implícito em três lugares. O módulo ECS adiciona IAM, Secrets Manager e um
ALB exposto — superfície nova que precisa de revisão própria.

**Conclusão:** esta spec entrega o **contrato**. O módulo ECS vira card próprio,
dependente deste e da decisão de custo, e não entra na v0.1.0.

## 3. Regras e critérios de aceitação

| # | Regra | Como se prova |
|---|---|---|
| 1 | Existe um contrato legível por máquina com serviços, variáveis lidas por cada um, e dependências | O arquivo existe e é parseável; `python -c "import json,sys; json.load(...)"` retorna 0 |
| 2 | O contrato é derivado do **código**, não escrito à mão | O gerador lê os `os.Getenv`/`viper` dos 7 serviços Go e do frontend; rodar de novo sobre o mesmo código produz arquivo idêntico |
| 3 | Cada empacotamento tem teste que falha quando não cobre o contrato | Remover uma variável do compose faz `tests/contract/` reprovar; saída colada antes e depois |
| 4 | O compose cobre o contrato hoje | `run_contract_tests.sh` verde |
| 5 | O chart cobre o contrato hoje | idem, incluindo as 15 variáveis do `navyr-helm#6` |
| 6 | Divergência nova é impedida, não detectada depois | O teste roda no CI dos repositórios de empacotamento |
| 7 | O que o contrato **não** cobre está escrito | Lista explícita das lacunas conhecidas na nota do último ciclo |

## 4. Fora de escopo

- **O módulo Terraform de ECS.** Vira card próprio; ver §2.
- **Tirar o `tunnel.Registry` da memória.** Maior alavanca do produto, e card
  próprio — citado aqui porque condiciona o valor do ECS.
- **Cache do Trivy em Fargate** (33s frio, 2,6s quente) e **timeout do ALB**
  (padrão 60s corta SSE e WebSocket; o nginx já usa `proxy_read_timeout 1h`).
  São decisões do card de ECS. Registradas aqui para não se perderem.
- **`#12` / SPEC-033** — visibilidade do ponto de entrada.

## 5. Plano de execução

| Ciclo | Entrega | Critério de pronto |
|---|---|---|
| 1 | Levantar o que cada serviço realmente lê, a partir do código dos 7 Go + frontend | Lista com serviço → variáveis, e a linha de origem de cada uma; saída colada |
| 2 | Gerador do contrato e o arquivo gerado | Critérios 1 e 2 — rodar duas vezes produz `diff` vazio |
| 3 | Teste de cobertura por empacotamento, ligado ao contrato | Critério 3 — prova negativa: remover variável faz reprovar |
| 4 | Fechar as lacunas que o ciclo 3 revelar no compose e no chart | Critérios 4 e 5 |
| 5 | Teste no CI dos repositórios de empacotamento | Critério 6 |
| 6 | Convergência e registro das lacunas conhecidas | Critério 7 |

Cada ciclo termina em commit semântico com a suíte verde.

## 6. Pipeline de entrega — o que se aplica

| Etapa | Status | Observação |
|---|---|---|
| 1. Versionamento | ✅ | commit semântico por ciclo |
| 2. Testes unitários | ✅ | `tests/contract/` é a suíte desta spec |
| 3. Build de imagem | ⛔ N/A | contrato e testes; nenhuma imagem nova |
| 4. Scan de imagem | ⛔ N/A | idem |
| 5. Push para registry | ⛔ N/A | idem |
| 6. Teste funcional | ✅ | subir a stack pelo compose e pelo chart e conferir que ambos satisfazem o contrato |
| 7. Teste E2E | ⛔ N/A | nenhuma mudança de comportamento de produto |

## 7. Decisões em aberto (nenhuma bloqueia o início)

1. **Partir o card em dois.** Assumo que sim, pelas razões da §2: o contrato
   paga hoje e não custa; o ECS custa e não tem demanda declarada. — Custa adiar
   a terceira opção de deploy. Se houver cliente esperando ECS, isso inverte, e
   quero saber antes do ciclo 1.

2. **Formato do contrato.** Assumo **JSON gerado**, não YAML escrito à mão: o
   critério 2 exige derivação do código, e JSON tem parser em toda ferramenta que
   vai consumi-lo. — Custa legibilidade para humano; mitigo com um `make
   contrato-diff` que imprime em tabela.

3. **De onde extrair as variáveis.** Assumo varredura estática dos
   `os.Getenv`/`viper.Get` nos repositórios de serviço. — Custa: variável lida
   por caminho dinâmico escapa. Registro isso como lacuna conhecida (critério 7)
   em vez de fingir cobertura total.

4. **Onde o contrato vive.** Assumo `navyr-deploy`, junto de `tests/contract/`,
   que é quem já o consome. — Custa: os repositórios de serviço passam a ter o
   `navyr-deploy` como dependência de CI. A alternativa é o `navyr-io/.github`,
   que já é dependência de todos.
