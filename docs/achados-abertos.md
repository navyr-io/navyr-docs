# Achados em aberto

Defeitos e riscos encontrados durante a profissionalização da plataforma
(agosto/2026) que **ainda não foram corrigidos**. Cada item registra o que é, por
que importa e o que a correção envolve.

Os achados **já corrigidos** estão nas mensagens de commit dos repositórios
correspondentes; o backlog SEC-04..14 do `NAVYR.md` foi fechado na Fase 1.

**Última revisão: 2026-08-19**

---

## Segurança

### Token em query string
**Onde:** `navyr-orchestrator` (manifesto do agente), `navyr-auth` (callback SSO)

O endpoint de manifesto autentica por `?token=` e o redirect de SSO devolve
`sso_token`, `refresh_token` e e-mail na query string. Query string vaza para
access log de proxy, histórico do navegador e cabeçalho `Referer` de qualquer
recurso externo que a página carregue.

**Correção:** mover para cabeçalho no manifesto — o cliente usa `curl`, então é
viável. No SSO exige mudar o handoff para o frontend, provavelmente via cookie
de uso único ou POST.

### JWT em `localStorage`
**Onde:** `navyr-frontend` — era o SEC-13

Vulnerável a XSS: qualquer script injetado lê o token. Migrar para cookie
`HttpOnly` é mudança de arquitetura de sessão, com impacto em CORS, CSRF e no
fluxo de refresh. Ficou fora da Fase 1 por isso.

### 63 achados de análise de taint sem triagem
**Onde:** todos os serviços Go

O `gosec` reporta 64 achados nas regras G702/G704/G705/G710. Um foi confirmado
real e corrigido (injeção de parâmetro no OAuth do community). Os outros 63 não
foram analisados um a um — foram excluídos do gate porque numa frota cujo
trabalho é proxiar requisições essas regras disparam constantemente e afogariam
o sinal.

**Correção:** uma passada dedicada de triagem, marcando cada um como falso
positivo com justificativa ou corrigindo. Só então voltar a ativá-las no gate.

### Monaco carregado de CDN
**Onde:** `navyr-frontend`

Não há `loader.config()` e `monaco-editor` nunca é importado direto — só
`@monaco-editor/react`. O editor é baixado da CDN jsdelivr em runtime. Três
consequências: a versão servida está fora do nosso controle, a CSP precisa
liberar jsdelivr, e **deploy air-gapped não funciona** — que é objetivo
declarado da Fase 6 do roadmap.

**Correção:** apontar o loader para o pacote local e configurar os workers no
Vite. Exige verificação visual do editor.

---

## Correção e robustez

### Nenhum serviço tem retry de conexão ao banco
**Onde:** todos os serviços Go

O boot faz `log.Fatalf` se o Postgres não estiver pronto. Em Docker Compose o
`restart` recupera; em Kubernetes vira `CrashLoopBackOff` até o banco subir —
recuperável, mas ruidoso e confuso para quem está diagnosticando outra coisa.

### `navyr-collector` reintroduz kubeconfig
**Onde:** `navyr-collector/internal/publish/kubeconfig.go`

Obtém kubeconfig por cluster através da API interna do orchestrator, para o SDK
do Helm operar. O agent tunnel foi construído justamente para a plataforma nunca
precisar de kubeconfig do cluster do cliente — a migration `000009` removeu o
modo direto.

Pode ser decisão consciente (o SDK do Helm não fala pelo túnel), mas contradiz o
diferencial que a documentação vende. **Merece um ADR justificando ou uma
revisão do desenho.**

### Migrations sem versionamento
**Onde:** todos os serviços com banco

Não existe tabela `schema_migrations`. A lista é descoberta em disco e
reexecutada inteira a cada boot, dependendo de todo `CREATE` ter `IF NOT
EXISTS`. Funciona, e é idempotente hoje — verificado. Mas uma migration futura
que não seja idempotente quebra o serviço em todo restart, e o defeito só
aparece no segundo boot.

Quatro migrations não têm `.down.sql`.

---

## Infraestrutura

### NetworkPolicy impedia a plataforma de subir — corrigido em 19/08

Eram quatro defeitos independentes no mesmo arquivo, e o chart entregava a
policy **ligada por padrão**:

1. **Sem egress de DNS** em `gateway-allow` e `backend-allow`. Liberavam as
   portas dos serviços e do Postgres, mas `navyr-postgres` é um nome — então
   nem a regra do banco funcionava.
2. **`navyr-community` e `navyr-frontend` sem policy nenhuma.** Só o
   `default-deny` valia para eles, deixando-os sem egress algum.
3. **Ingress do Postgres não incluía o community**, que usa banco desde a
   Fase 0. NetworkPolicy exige as duas pontas: não basta a origem ter egress,
   o destino precisa aceitar ingress. Foi o mais sutil dos quatro.
4. **`{{- end }}` na linha 111 de 184**, deixando `redis-allow` e
   `collector-allow` fora do condicional — aplicadas mesmo com a feature
   desligada.

Junto entrou egress externo para provedores de IA, KMS, OIDC, Stripe, SMTP e
LDAP, com `except` negando faixas privadas e `169.254.0.0/16`. É defesa em
profundidade para o mesmo SSRF já tratado no código: se a validação de URL
falhar, a rede ainda barra o endpoint de metadados.

**Por que passou despercebido até agora, e a correção de um relato meu:** a
validação da Fase 4.2, que reportei como "13/13 pods 1/1", rodou no CNI padrão
do kind — que **não aplica NetworkPolicy**. O teste passou sem testar. A
verificação correta exige um CNI que aplique, e foi refeita com Calico: 13/13
pods, gateway pronto (o `/ready` dele consulta os quatro serviços), e os
bloqueios confirmados por teste negativo — pod sem rótulo da plataforma não
alcança Postgres nem auth, e o endpoint de metadados está barrado.

**Vale como padrão:** teste de controle de segurança precisa provar que o
controle **bloqueia**, não só que o sistema sobe. Um teste que só verifica
funcionamento passa igual quando o controle está inerte.

### Três caminhos de deploy paralelos — parcialmente resolvido em 19/08

Helm, Kustomize em `k8s/` e o Compose em `navyr-deploy` descrevem a mesma
plataforma de formas independentes. O que o Kustomize tinha a mais — probes e
`securityContext` do Postgres — foi portado para o Helm na Fase 4.2, então a
divergência que justificava manter os dois acabou.

**Continua aberto:** aposentar o caminho Kustomize. Enquanto os dois existirem,
qualquer correção precisa ser feita duas vezes, e foi assim que a divergência
apareceu.

### Fase 4.2 — o que fechou e o que ela revelou

Fechado no chart em 19/08: probes nos 8 Deployments, Postgres non-root (uid 70),
ServiceAccount por serviço sem token da API montado, TLS opcional no Ingress,
HPA para gateway e orchestrator, renomeação completa de `kubeops-` para
`navyr-`, e recusa de instalar com os segredos de exemplo.

Três coisas que só apareceram porque o chart foi instalado de verdade, e que
`helm lint` e `helm template` não pegam:

- **`PGDATA` na raiz do volume.** Com Postgres non-root o `initdb` precisa dar
  `chmod` no diretório do mount, que não pertence ao usuário, e o container
  entra em crashloop. Corrigido movendo para um subdiretório.
- **`values-prod.yaml` apontava para `docker.io/erickdavi/kubeops-*`** — o
  Docker Hub pessoal da era do monorepo, não o registry da organização.
- **README e `deployment.md` documentavam `global.jwtSecret` e
  `global.databaseUrl`**, values que nunca existiram no chart. Quem seguisse a
  instrução instalaria com os segredos de exemplo sem perceber.

**Confirmado depois:** as tags em `values-prod.yaml` (`v0.1.0-20260501`) **não
existem**. O registry publica apenas `main`, `latest` e `sha-<commit>` — não há
nenhuma tag semver, porque releases versionadas ainda não foram feitas (ver
"Sem releases versionadas"). Instalar com esse arquivo falha em `ImagePullBackOff`
nos cinco serviços. Fixado em tags `sha-` reais até existir semver.

**Continua aberto:** o HPA nunca foi exercido com carga real: foi validado apenas
que os objetos são criados e apontam para os Deployments certos, com as métricas
em `<unknown>` por falta de metrics-server no cluster de teste.

---

### CI nunca rodou em PR do Dependabot — corrigido em 19/08

**28 dos 36 PRs abertos do Dependabot não tinham CI nenhum.** Todos falhavam
com `startup_failure`, antes de qualquer job iniciar. A fila não estava parada
por falta de atenção: estava invisível. Não havia sinal algum de que as
atualizações passavam ou quebravam, e o Dependabot foi instalado na Fase 2
justamente para dar esse sinal — depois de 123 CVEs acumuladas por deriva
silenciosa. O alarme estava mudo desde que foi instalado.

**Causa.** Permissões de workflow são resolvidas na partida, para **todos** os
jobs do grafo, antes de qualquer condição `if:` ser avaliada. Em execução
disparada pelo Dependabot o `GITHUB_TOKEN` é limitado a leitura. O job de
imagem dos workflows reutilizáveis pedia `packages: write` — e um único pedido
de escrita acima do teto derruba a execução inteira.

Isso não se resolve com `if:` no job, nem movendo a permissão do topo do
arquivo para o job: em ambos os casos a permissão continua no grafo. A única
saída é o arquivo avaliado num evento de pull request não conter escrita
nenhuma.

**Correção.** A publicação saiu para `publish-image.yml`, chamado apenas de um
`publish.yml` com trigger de push. O `go-service.yml` e o `frontend.yml` ficaram
só com validação, com `contents: read`. O scan de imagem em pull request foi
preservado — é o que pega problema de imagem antes do merge.

**Três hipóteses erradas antes da certa**, todas testadas contra o GitHub:
permissão no chamador; job guardado por `if:` não reservaria permissão; e
repositório privado do workflow reutilizável bloquearia o Dependabot. A
terceira foi descartada com um workflow de diagnóstico isolado, rodando em
paralelo — em vez de uma hipótese por vez, que foi o que custou oito rodadas
na Fase 2.

### Ferramentas de análise não acompanham o toolchain — corrigido em 19/08

Subir os 7 serviços para Go 1.26.6 — necessário porque helm 3.21.4 e
k8s.io 0.36.x exigem 1.26 — quebrou **as duas** ferramentas de análise
estática do pipeline. Nem `golangci-lint` (v2.12.2, de maio) nem `gosec`
(v2.28.0, de julho) têm release compilada com 1.26.

O `golangci-lint` falha de forma honesta, dizendo que a versão usada para
compilá-lo é menor que a versão alvo do módulo.

**O `gosec` falha de forma perigosa.** Ele não reprovava por achado de
segurança: emitia `Golang errors in file` em todos os pacotes — ou seja, não
analisava nada — e ainda assim imprimia `Issues : 0`. Só deu vermelho porque
sai com código 1 nesse caso. Se saísse com 0, teríamos um SAST verde e
completamente vazio, indistinguível de cobertura real. Depois da correção, o
mesmo scan analisa 7 arquivos e 2.600 linhas só no billing.

**Correção:** ambos passaram a ser compilados com o toolchain do próprio
projeto, via `go install` da mesma versão fixada. A versão continua pinada e o
`go install` verifica contra a checksum database. Voltar às actions quando
saírem releases compiladas com 1.26.

**Vale como padrão, não como caso isolado:** ao subir o toolchain, verificar se
as ferramentas do pipeline acompanham — e desconfiar de ferramenta que passa a
reportar zero achados logo após um bump.

### `--with-deps` do Playwright travava o CI — corrigido em 19/08

O job de E2E chegou a queimar 25 minutos sem rodar um único teste. A causa não
era o download do browser, como supus duas vezes: era o `apt-get` embutido no
`--with-deps`, que ficava em silêncio no `archive.ubuntu.com` até o timeout.

O apt era dispensável desde o início — a imagem do runner já traz o Google
Chrome instalado, então as bibliotecas de sistema que o Chromium precisa já
estão presentes. `playwright install chromium` sem `--with-deps` elimina o
passo instável.

**Registro do erro de método:** as duas primeiras tentativas atacaram o
download (cache, depois bifurcação por cache-hit). A segunda ainda introduziu
uma regressão, derrubando PRs que já estavam verdes. Só ler o log resolveu — o
que deveria ter sido o primeiro passo, não o terceiro.

### Agrupamento do Dependabot juntava majors — corrigido em 19/08

O grupo `build` do `navyr-frontend` propôs cinco saltos de major num PR só
(TypeScript 5→7, Vite→8, ESLint→10, vitest→4, plugin-react→6). Quebrou no
`npm ci` com `ERESOLVE`: o grupo subia `typescript` mas não
`typescript-eslint`, que limita a versão de TS suportada — os padrões não
casavam esse pacote.

Os grupos agora só agregam `minor` e `patch`, que é o caso em que mover junto
ajuda. Major vem em PR próprio, para ser avaliado com o custo de migração à
vista.

**Continua aberto:** os majors pendentes precisam de decisão individual —
TypeScript 7, Vite 8, ESLint 10, Tailwind 3→4, Node 22→26 na imagem base, e
`actions/checkout` 4→7. Tailwind 4 em particular é reescrita de configuração,
não bump.

## Qualidade

### Arquivos-deus — parcialmente resolvido em 19/08

| Arquivo | Antes | Depois |
|---|---|---|
| `orchestrator/internal/handler/kubernetes_handler.go` | 4.316 | **967** (dividido em 7) |
| `gateway/cmd/server/main.go` | 4.349 | **2.904** (6 extraídos) |
| `auth/internal/service/auth_service.go` | 3.906 | **1.743** (6 extraídos) |
| `orchestrator/cmd/server/main.go` | 3.033 | 3.033 — não tocado |
| `frontend/src/screens/SettingsPage.tsx` | 1.864 | 1.864 — não tocado |

A divisão feita foi **movimento puro**: declarações movidas entre arquivos do
mesmo pacote, sem alterar assinatura nem corpo. A forma foi escolhida por ser
verificável — Go recusa declaração duplicada, então compilar já prova que nada
foi copiado em dobro, e a cobertura permanecer idêntica prova que nenhum caminho
de execução mudou.

**Efeito colateral em `navyr-auth`.** A divisão por domínio deixou visível a
fronteira entre a edição livre e a enterprise: LDAP, SSO, SCIM, grupos e grants
somam cerca de 1.760 linhas agora isoladas em arquivos próprios. Se a separação
open core for adotada, o trabalho neste repositório — que era o mais pesado —
deixa de ser desembaraçar código e passa a ser mover arquivos.

**O que falta no gateway, e por quê.** A função `main()` sozinha tem 794 linhas,
quase todas na tabela de 132 rotas, e o despacho vive em `apiHandler` — uma
closure que captura `httpClient`, as URLs dos serviços e os proxies do escopo de
`main`. Extrair isso exigiria passar essas dependências por parâmetro ou struct,
o que deixa de ser movimento puro e vira **mudança de assinatura no caminho de
autenticação de toda a plataforma**.

Com 15,7% de cobertura nesse arquivo, o passo correto é cobrir o despacho de
rotas com teste primeiro. Fazer o contrário é refatorar sem rede o único ponto
público do produto.

### E2E — fechado em 19/08, com cobertura perdida

Os 4 specs passam ponta a ponta. O que estava quebrado não era o teste: os
specs foram escritos contra uma versão anterior da aplicação e ficaram parados
enquanto a UI evoluiu. A depuração expôs quatro defeitos reais, corrigidos na
fonte e não no teste:

| Defeito | Onde | Correção |
|---|---|---|
| Rotas de workspace passaram a exigir escopo de organização, mas o login não propagava `org_id` — a barra lateral de cluster nunca montava em link direto | `AppShell.tsx` + contrato de login | mocks alinhados ao contrato real; comportamento da aplicação estava correto |
| Estado do pod comunicado **apenas por cor** — sem texto equivalente (WCAG 1.4.1) | `WorkloadDetailPage.tsx` | indicador ganhou `role="img"` e `aria-label` com o status |
| Linha de workload era `div` clicável sem papel, foco ou acionamento por teclado | `WorkloadsPage.tsx` | `role="button"`, `tabIndex`, `aria-pressed`, handler de `Enter`/`Espaço` |
| Namespace da tela de rede vivia só em estado local: deep link e link compartilhado perdiam o filtro | `NetworkPage.tsx` | sincronizado com a query string, como já fazia a tela de workloads |

**Cobertura que se perdeu junto com as telas.** Duas assertivas testavam
funcionalidade que não existe mais e foram reescritas contra o comportamento
atual, não remendadas:

- **Paginação e seleção em massa de workloads** (`Página 1 de 3`,
  `Selecionados: 1`, `Próxima`) — a tabela paginada virou lista completa com
  seleção de linha única e painel de inspeção. Se a paginação voltar, precisa de
  teste novo.
- **Tela de "Recursos Genéricos"** — removida da aplicação. O spec ainda mocka
  `/api/v1/resources**`, mock hoje sem consumidor.

**O que continua aberto:** o job de E2E roda e publica relatório, mas não
bloqueia (`continue-on-error`) — mesma limitação de plano Free descrita em
"Gate de CI". Os specs cobrem um caminho feliz por fluxo; não há teste de erro,
de permissão negada nem de sessão expirada.

### Logging não estruturado
Todos os serviços usam o `log` da stdlib. Sem `org_id`, `cluster_id` ou
`request_id` nos registros, correlacionar um incidente em produção depende de
`grep` e sorte.

### Cobertura — meta revisada em 19/08

A meta original era **60% de statements** em `internal/service` e
`internal/repository`. Foi **substituída por cobrir todo caminho onde a falha é
cara**, com o percentual como consequência e não como alvo.

**Por que mudou.** Os números pararam de subir por composição dos pacotes, não
por dificuldade: `auth/internal/repository` tem 1.980 linhas com dezenas de
métodos de SSO, LDAP, SCIM, grupos e webhooks; `orchestrator/internal/service`
tem cerca de 5.000 linhas de wrappers de client-go. Chegar a 60% exigiria 40 a
60 funções de teste quase idênticas — testar `ListDaemonSets` depois de
`ListDeployments` move o percentual sem encontrar defeito novo. Escrever teste
para satisfazer métrica produz suíte grande que não pega bug, e que ninguém
mantém.

**Critério que substitui o número.** Um caminho está coberto quando há teste
para: perda de isolamento entre organizações, invalidação de sessão e revogação
de permissão, criptografia de credencial, travessia do túnel, e a cadeia de
migrations. São os caminhos em que o defeito é silencioso e a consequência é
vazamento, cobrança errada ou acesso indevido.

**Estado atual:**

| Pacote | Cobertura |
|---|---|
| `orchestrator/internal/tunnel` | 83,2% |
| `billing/internal/repository` | 27,5% |
| `orchestrator/internal/service` | 14,8% |
| `auth/internal/repository` | 12,1% |
| `orchestrator/internal/repository` | 4,9% |

Os cinco caminhos do critério estão cobertos. O percentual segue baixo nos
pacotes grandes, e isso é aceito conscientemente.

**Continua valendo:** todo defeito encontrado ganha teste de regressão, e o
harness de integração com Postgres real é o mecanismo padrão para a camada de
repositório — foi ele que encontrou as nove colunas sem migration em
`navyr-auth` e a coluna obrigatória em `user_invites`.

---

## Vulnerabilidades sem correção upstream

Quatro CVEs alcançáveis em `navyr-orchestrator` e `navyr-collector`, todas
transitivas do SDK do Helm, sem correção publicada. Declaradas em
`.govulncheck-ignore` de cada repositório com justificativa. Mais nove no
binário do `helm` embarcado na imagem do orchestrator, e duas no `dompurify`
que o Monaco traz.

Todas já estão na versão mais recente disponível. Revisar quando o Dependabot
abrir PR para `containerd`, `golang.org/x/crypto` ou `monaco-editor`.

---

## Governança

### Arquivos de governança — resolvido em 19/08

`LICENSE`, `SECURITY.md`, `CONTRIBUTING.md` e `CODEOWNERS` existem agora nos 11
repositórios; templates de issue e PR ficam no repo `.github` da organização,
valendo para todos sem duplicação.

A ausência de `LICENSE` era o caso mais desconfortável: sem licença explícita o
padrão legal é "todos os direitos reservados" **por omissão**, não por decisão —
o que não é a mesma coisa que declarar a licença que de fato vale. Aplicada a
Navyr Software License v1.0, já vigente, sem tomar decisão nova.

**Continua em aberto:** a separação open core, registrada como "em consideração"
no `editions.md`. É decisão de negócio. Vira urgente se algum repositório for
publicado — inclusive se essa for a saída para o teto de minutos de CI.

Também em aberto: `CHANGELOG.md`, que só faz sentido junto das releases semver
(item 4.5), e o CLA, sem o qual contribuição externa não pode ser aceita.

### Documentação

Sem ADRs e sem runbooks de incidente — os scripts em `navyr-deploy/scripts/ops`
existem e não estão documentados.

`openapi.yaml` não existe em `navyr-gateway` nem em `navyr-billing`, e a spec
unificada em `navyr-deploy/spec` convive com as por serviço sem reconciliação.
O `docs/deployment.md` foi corrigido em 19/08 — descrevia values que nunca
existiram no chart.

**Branch protection** não pode ser configurada: o plano Free da organização
retorna 403 para repositório privado. O CI roda e reprova de forma visível, mas
nada impede o merge.
