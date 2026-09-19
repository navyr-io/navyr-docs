# SPEC-033 — Ponto de entrada público para terceiros

**Estado:** proposta
**Data:** 16/09/2026
**Card:** navyr-io/navyr-deploy#12
**ADR:** 0006 (empacotamentos que divergem)
**Relacionada:** SPEC-031 (tags v0.1.0), navyr-deploy#1 (branch protection), navyr-deploy#4 (GitHub Team)

## 1. Problema

As **8 imagens são públicas** e baixam sem credencial. Os **repositórios não**:
só o `navyr-docs` é público. Quem recebe as instruções consegue baixar as
imagens e não consegue obter o compose que as orquestra.

Percorrendo o Quick start do README como um terceiro:

| Passo | Resultado |
|---|---|
| 1. `docker login ghcr.io` | desnecessário — as imagens são públicas (README desatualizado) |
| 2. `cp .env.example .env` | **bloqueado** — arquivo em repo privado |
| 3. `docker compose up -d` | **bloqueado** — `docker-compose.yml` em repo privado |
| 4. conectar cluster | **bloqueado** — chart não publicado |

Enquanto isso durar, os 134 itens concluídos no quadro não são instaláveis por
ninguém de fora. É o item que decide se o produto existe para terceiros.

## 2. A recomendação técnica, e por que ela

Das três opções que o card registra — abrir o `navyr-deploy`, criar repositório
separado de distribuição, ou entregar tarball por cliente — a recomendação é
**tornar o `navyr-deploy` público**, pelos três eixos:

### Segurança — medido, não suposto

| Verificação | Resultado |
|---|---|
| Arquivos rastreados | 47; único `.env` é `.env.example` |
| `.env` real já commitado em qualquer ponto | **nunca** (`git log --all --diff-filter=A`) |
| Padrões de credencial (`AKIA`, `ghp_`, `github_pat_`, chave privada PEM) no conteúdo atual | **nenhum** |
| Os mesmos padrões em **todo o histórico** (22 commits) | **nenhum** |
| Valores literais em variáveis de segredo | só defaults de teste (`navyr-functional`, `0123456789abcdef…`), em `${VAR:-default}` |

E o argumento de sigilo já está vencido por outro caminho: **as imagens são
públicas**, então o binário e a árvore de arquivos já são inspecionáveis por
quem quiser. Manter o compose fechado não protege o código — só impede a
instalação.

### Custo — zero, e evita um custo recorrente

Abrir o repositório não custa nada. As alternativas custam:

- **Repositório separado de distribuição** cria uma segunda fonte da verdade
  para o mesmo compose. É exatamente o modo de falha que a ADR 0006 descreve e
  que o `navyr-helm#6` mediu: 15 variáveis que o código lê, presentes no compose
  e ausentes do chart. Duas cópias divergem; a terceira divergiria igual.
- **Tarball por cliente** não tem caminho de atualização: cada correção exige
  reenviar a todos, e não há como saber quem está em qual versão.

### Escalabilidade — e um efeito colateral que destrava outro card

Um ponto de entrada público atende N clientes sem trabalho por cliente.

E há um ganho que o card `#1` não considerou. A API responde:

> `403 — Upgrade to GitHub Pro or make this repository public`

**Repositório público tem branch protection no plano Free.** Abrir o
`navyr-deploy` protege o `navyr-deploy` sem assinar nada — não resolve os 11,
mas resolve o repositório que os terceiros de fato consomem, que é onde a
proteção mais importa.

## 3. Regras e critérios de aceitação

| # | Regra | Como se prova |
|---|---|---|
| 1 | Nenhum segredo entra no domínio público | Varredura de conteúdo **e histórico** repetida imediatamente antes da troca de visibilidade; saída vazia colada |
| 2 | Um terceiro sem credencial chega à tela de login só com o README | Execução em máquina/contexto sem credencial e sem acesso à org, do passo 1 ao fim, com saída colada |
| 3 | O README deixa de mandar fazer o que não é necessário | `docker login ghcr.io` sai do Quick start — as imagens são públicas |
| 4 | O `.env.example` cobre tudo que o compose lê | `tests/contract/test_env_example_cobre_o_compose.py` verde — a catraca já existe |
| 5 | O chart do agente fica alcançável, ou o README para de prometê-lo | Passo 4 do Quick start funciona, **ou** é substituído por instrução que funciona |
| 6 | Branch protection ligado no `navyr-deploy` | `gh api /repos/navyr-io/navyr-deploy/branches/main/protection` responde 200 |
| 7 | A abertura não expõe repositório que deva seguir privado | Só `navyr-deploy` muda de visibilidade; os outros 10 conferidos `private=true` ao fim |

## 4. Fora de escopo

- **Abrir os outros 10 repositórios.** Esta spec abre um, o de instalação.
- **`#1` branch protection nos 11** — continua bloqueado por plano para os
  privados.
- **`#4` assinar GitHub Team** — decisão de negócio, re-escopada em 16/09.
- **`#17` ECS** — terceira opção de deploy.
- **Publicar o chart do `navyr-helm`** além do mínimo que o critério 5 exige.
- **Licenciamento por edição** (`#8`, despriorizado).

## 5. Plano de execução

| Ciclo | Entrega | Critério de pronto |
|---|---|---|
| 1 | Varredura de segredo em conteúdo e histórico, imediatamente antes de qualquer troca | Critério 1 — saída colada na nota de ciclo |
| 2 | Corrigir o README: tirar o `docker login`, conferir o passo 4 | Critérios 3 e 5; `test_env_example_cobre_o_compose.py` verde (critério 4) |
| 3 | **Ponto de não retorno:** tornar `navyr-deploy` público | `gh api /repos/navyr-io/navyr-deploy --jq .private` devolve `false` |
| 4 | Ligar branch protection no `navyr-deploy` | Critério 6 |
| 5 | Prova de terceiro: seguir o README do zero, sem credencial | Critério 2 — a prova real desta spec |
| 6 | Convergência | §3 item a item, com saída colada; critério 7 |

O ciclo 3 é **irreversível na prática**: republicar como privado não apaga o que
foi clonado ou indexado. Por isso o ciclo 1 roda imediatamente antes, e não no
começo da sessão.

## 6. Pipeline de entrega — o que se aplica

| Etapa | Status | Observação |
|---|---|---|
| 1. Versionamento | ✅ | commit semântico por ciclo |
| 2. Testes unitários | 🔁 | `navyr-deploy` não tem suíte Go; o equivalente é `tests/contract/` |
| 3. Build de imagem | ⛔ N/A | o repositório não produz imagem; ele orquestra as dos outros |
| 4. Scan de imagem | ⛔ N/A | nenhuma imagem nova |
| 5. Push para registry | ⛔ N/A | idem |
| 6. Teste funcional | ✅ | é o critério 2 — subir a stack seguindo só o README |
| 7. Teste E2E | ✅ | chegar à tela de login e autenticar |

## 7. Decisões em aberto

> **Esta é a única spec desta rodada cuja decisão principal não é técnica.**
> Tornar um repositório público é irreversível e é chamada do Erick. O ciclo 3
> não executa sem o "sim" explícito, mesmo com a spec aprovada.

1. **Abrir o `navyr-deploy` em vez de criar repo de distribuição.** Assumo
   abrir, pelas três razões da §2. — Custa tornar visível a topologia do
   compose, os scripts de operação e o `SECURITY.md`. Nenhum deles contém
   segredo (§2), mas todos descrevem como o sistema é operado.

2. **Os scripts `scripts/ops/` vão junto.** Assumo que sim: `backup_postgres.sh`,
   `production_cutover.sh` e `rollback_release.sh` são parte de operar o produto,
   e quem instala precisa deles. — Custa expor o procedimento de cutover; se
   preferir, movem-se para repo privado e o README passa a citá-los como
   material de suporte.

3. **Defaults de teste que funcionam em produção.** `CLUSTER_CREDENTIAL_ENCRYPTION_KEY`
   cai para `0123456789abcdef0123456789abcdef` quando não definido, em três
   scripts de `scripts/beta/`. Hoje isso é ruído; **em repositório público vira
   chave de criptografia conhecida e documentada**. Assumo que o ciclo 2 faz
   esses scripts recusarem partir sem a variável, em vez de cair no default —
   mesmo padrão que o `navyr-helm#6` aplicou a `APP_ENV=production`. — Custa um
   ciclo maior; não fazer é publicar uma chave conhecida.

4. **O chart do agente (passo 4 do Quick start).** Assumo, se não for publicável
   neste escopo, trocar o passo por instrução que funcione hoje, em vez de
   deixá-lo prometendo o que falha. — Custa um passo manual a mais para o
   terceiro; é melhor que um passo quebrado.
