# SPEC-030 — Gate de SAST que reprova sem achado verdadeiro

**Estado:** proposta
**Data:** 16/09/2026
**Card:** navyr-io/navyr-deploy#18
**Relacionada:** SPEC-029 (o portão de ação crítica, cujo código criou o sink novo)

## 1. Problema

O CI voltou a executar em **01/09**, quando o teto de minutos do plano Free
resetou no ciclo de cobrança. Consumo de setembro: 146 de 2000 minutos.

A prova é a duração dos jobs, não a ausência de erro: durante o bloqueio eles
morriam em 2–4s sem executar passo nenhum (`navyr-auth` run 33432615584 —
`started 19:48:54 → completed 19:48:56`); depois do reset há runs de 244s, 454s
e 597s com passos reais.

Desde então o gate do `gosec` **reprova 5 dos 7 serviços Go**. Os achados foram
triados um a um, com o código lido em cada ponto:

| Repo | Achado | Leitura |
|---|---|---|
| `navyr-auth` | `G115` overflow `uint64→int64`, Severity HIGH — `cmd/server/migration_lock.go:63` | Falso positivo. É a chave do `pg_advisory_lock`, `int64(binary.BigEndian.Uint64(sha256[:8]))`. O código traz 12 linhas explicando que o sinal é irrelevante para a identidade da trava e que o Postgres aceita o intervalo inteiro de `bigint`. |
| `navyr-orchestrator` | `G705` XSS via taint — `internal/handler/agent_manifest_handler.go:136` | Falso positivo. A linha é `w.Write([]byte(manifest))`, com `Content-Type: application/yaml` e `Content-Disposition: attachment`. O `cluster.Name` interpolado acima é validado por `clusterNamePattern = ^[a-zA-Z0-9][a-zA-Z0-9._-]{0,62}$` — sem aspas, sem espaço, sem quebra de linha. |
| `navyr-gateway` | `G704` SSRF via taint — `cmd/server/main.go:2941` | Falso positivo. Destino é `orchestrator.ResolveReference()`, host fixado por env e não pelo request; o `approvalID` passa por `url.PathEscape`; a requisição é assinada com HMAC de contexto interno. |
| `navyr-gateway` | `G124`×2 cookie inseguro — `cmd/server/sessao.go:194,206` | Falso positivo. `HttpOnly: true` e `SameSite: Lax` presentes. `Secure: cookieSeguro()` devolve `true` só com `APP_ENV=production`, com comentário registrando que fixar `true` quebraria dev em http sem dar pista. O gosec reprova por não ser literal `true`. |
| `navyr-gateway` | `G117`×2 `RefreshToken` — `cmd/server/sessao.go:115,168` | Falso positivo. É o `json.Marshal` do registro de sessão para gravar no Redis. Não vai para cliente nenhum. |
| `navyr-community` | `G124`×2 cookie inseguro — `cmd/server/oauth_state.go:85,97` | Mesmo padrão do gateway. |

**7 achados, 7 falsos positivos.**

### A causa

O commit `c45f0c1` (20/08, repo `navyr-io/.github`) religou `G704`, `G705` e
`G710` depois de triar 72 achados. A triagem foi legítima e valiosa: foi ela que
descobriu o takeover de conta por `sso_token` na query string do callback de
SSO, sinalizado por uma regra que estava desligada. Registrou "zero achados nos
7 serviços" **naquele dia**.

A mesma mensagem de commit registra o que parecia ser a armadilha:

> as regras de taint do gosec 2.28 **não honram `#nosec`**, nem no destino nem
> na origem

**Isto foi refutado no ciclo 2 desta spec (16/09).** `#nosec` funciona em regras
de taint; a anotação precisa estar na linha que o gosec **reporta** — o sink —
e não na que parece a origem.

O `navyr-gateway` tem **54 anotações `#nosec G704`**, todas na linha do
`http.NewRequestWithContext`. O gosec reporta a linha do `client.Do(req)`. Por
isso nenhuma suprime nada. Medido: mover a anotação para a linha do sink faz o
achado sumir (`Nosec` 54→55, `Issues` 5→4). Enquanto a regra esteve excluída,
esse desalinhamento não aparecia.

O que mudou entre 20/08 e 01/09: código novo entrou **durante o apagão do CI**
(19/08 a 01/09) e criou sinks novos — no gateway, o `novoConsultorDeAprovacao`
da SPEC-029 C. Sem CI, ninguém viu.

### O que isso esconde

Três coisas, e a terceira é a que importa:

1. Os **28 PRs do Dependabot** não têm sinal: todos herdam a reprovação da `main`.
2. O pipeline obrigatório de entrega não pode ser cumprido — a etapa 4 (scan)
   reprova sempre.
3. **Um gate 100% ruído treina a equipe a ignorar SAST.** Quando ele achar o
   próximo takeover de SSO — como achou em 20/08 — ninguém vai olhar. Pelo
   critério de severidade do projeto ("defeito silencioso sobe de nível"), é
   isto que faz o item ser **Alto** e não Médio.

### Uma armadilha de ambiente encontrada junto

O clone local de `navyr-io/.github` (`/home/erick/navyr-repos/.github-org`)
estava **10 commits atrás** do remoto, ainda com
`-exclude=G702,G704,G705,G710,G118` enquanto o CI usava `-exclude=G702,G118`.
Rodar `gosec` local passava verde e o CI reprovava. O repo começa com ponto, o
que o esconde de `ls` sem `-a`, e o gate não está em nenhum dos 11 repos de
serviço — cada `ci.yml` apenas chama
`navyr-io/.github/.github/workflows/go-service.yml@main`.

## 2. Regras e critérios de aceitação

| # | Regra | Como se prova |
|---|---|---|
| 1 | O gate só reprova diante de achado verdadeiro | `gosec` com os flags do CI nos 7 serviços Go: exit 0 em todos, **ou** achado remanescente com análise escrita nesta spec |
| 2 | Toda regra ligada tem supressão que funciona | Para cada regra ligada, um caso de teste anotado com o mecanismo escolhido deixa de ser reportado. Colar a saída antes e depois |
| 3 | Regra que já achou defeito real não é desligada | `G710` permanece ligada (foi ela que sinalizou o takeover de SSO em 20/08). Provar com `grep exclude` no workflow |
| 4 | Toda regra desligada tem motivo técnico escrito no ponto da exclusão | `grep -B12 'exclude=' go-service.yml` mostra comentário nomeando cada regra e o porquê |
| 5 | As 54 anotações `#nosec G704` do gateway ou passam a valer, ou são removidas | Não fica anotação inerte no código: `grep -c '#nosec' cmd/ internal/` bate com o número de supressões efetivas |
| 6 | O clone local não pode divergir em silêncio do gate do CI | Comando de checagem de drift documentado e executado; saída vazia = alinhado |
| 7 | Os 7 serviços fecham um run de CI verde em `main` | `gh run list --branch main` mostra `success` no workflow `CI` em cada repo |

## 3. Fora de escopo

- **`navyr-deploy#19`** — `golangci-lint` e testes de integração. Card irmão,
  arquivos e causas diferentes.
- **`G702`** — já excluída em 20/08 por razão técnica registrada (o orchestrator
  legitimamente executa `trivy`, `syft` e `cosign`). Não se reabre aqui.
- **Os 3 achados `G118`** nunca examinados, registrados em
  `achados-abertos.md` como não verificados. Continuam fora.
- **HA / `tunnel.Registry` em memória.** Citado em `navyr-deploy#17` como o item
  de maior alavanca; não é deste card.
- **Merge dos 28 PRs do Dependabot.** Consequência desta spec, não entrega dela.

## 4. Plano de execução

| Ciclo | Entrega | Critério de pronto |
|---|---|---|
| 1 | Reproduzir o gate real localmente, com o clone `.github-org` sincronizado, e registrar os 7 achados | `gosec -severity medium -confidence medium -exclude-generated -exclude=G702,G118 ./...` nos 5 repos afetados reproduz exatamente os 7 achados da tabela §1 — saída colada na nota de ciclo |
| 2 | **Medir** se `gosec` posterior à 2.28 honra `#nosec` em regras de taint | Um arquivo mínimo com sink de taint anotado, rodado nas duas versões. A saída decide o ciclo 3 e fica colada. Esta é a reavaliação que o `c45f0c1` pediu e nunca foi feita |
| 3 | ~~Aplicar o caminho que o ciclo 2 determinar~~ → **Nenhuma exclusão nova.** O ciclo 2 mostrou que o gate pode ficar inteiro; o `go-service.yml` não muda | `grep 'exclude=' go-service.yml` continua em `G702,G118` |
| 4 | Mover as 54 anotações do gateway para a linha do sink, e as equivalentes nos demais serviços | Critério 5 da §2: `gosec` com os flags do CI devolve `exit 0` nos 5 repos afetados |
| 5 | Checagem de drift do `.github-org` como passo documentado da validação local | Comando no `CLAUDE.md` e saída vazia demonstrada |
| 6 | Convergência: CI verde em `main` nos 7 serviços | Critério 7 da §2 — `gh run list` colado para cada repo |

Cada ciclo termina em commit semântico com a suíte verde.

## 5. Pipeline de entrega — o que se aplica

| Etapa | Status | Observação |
|---|---|---|
| 1. Versionamento | ✅ | commit semântico por ciclo, em `navyr-io/.github` e nos repos de serviço tocados |
| 2. Testes unitários | ✅ | suíte existente dos serviços; esta spec não deve alterar comportamento |
| 3. Build de imagem | ⛔ N/A | `navyr-io/.github` não produz imagem. Os serviços não têm código de runtime alterado nesta spec |
| 4. Scan de imagem | 🔁 | é o próprio objeto da spec: o gate volta a ser o controle, em vez de estar quebrado |
| 5. Push para registry | ⛔ N/A | nenhuma imagem nova |
| 6. Teste funcional | 🔁 | o teste funcional **é** o CI verde nos 7 repos (critério 7 da §2) |
| 7. Teste E2E | ⛔ N/A | nenhuma mudança de comportamento de produto |

**Correção ao texto da skill:** ela afirma que "o CI está bloqueado pelo plano
Free desde 19/08/2026" e que a verificação é substituída pelo `pre-push`. Isso
deixou de ser verdade em **01/09**, quando o teto resetou. O CI executa; o que
não passa é o gate. A verificação aqui é o próprio CI, não o `pre-push`.

## 6. Decisões em aberto (nenhuma bloqueia o início)

1. **`navyr-docs` é o único repositório público da org, e esta spec descreve
   quais regras de SAST estão desligadas e onde ficam os sinks.** Assumo
   publicar, como as 29 specs anteriores — várias já descrevem vulnerabilidade
   (`018-auth-no-nginx-contorna-o-gate`). — Custa expor, a quem ler, que a
   análise de taint está parcialmente desligada e em que arquivos. Se preferir,
   a spec vive fora do `navyr-docs` e só o card referencia.

2. ~~Subir a versão do gosec se ela honrar `#nosec` em taint.~~
   **RESPONDIDA no ciclo 2:** 2.29.0 é idêntica à 2.28.0 — mesmos 5 achados,
   mesmo `Nosec: 54`. Subir versão não resolve nem atrapalha. Fica em 2.28.0.

3. ~~Excluir `G704`/`G705` com a razão do `G702`.~~
   **NÃO É MAIS NECESSÁRIO:** `#nosec` funciona. O gate fica inteiro, com as
   três regras de taint ligadas.

4. ~~Remover as 54 anotações inertes.~~
   **INVERTIDA:** elas não saem, **mudam de linha**. Passam a valer quando
   movidas para o sink que o gosec reporta. É movimento puro — nenhuma
   assinatura muda, e o compilador prova que nada se perdeu.

6. **Novo, vindo do ciclo 2: o `G702` foi excluído pela mesma razão agora
   refutada.** Assumo **não** reabri-lo nesta spec — está na §3 (fora de
   escopo) e mexer nele aqui aumenta o alcance de um card que já está em
   execução. — Custa manter uma regra desligada sem motivo válido; registro
   como card próprio, já que devolveria ao gate a detecção de execução de
   comando no orchestrator.

5. **`G115`, `G117` e `G124` não são regras de taint**, então `#nosec` funciona
   nelas. Assumo anotar caso a caso, com o motivo já escrito na tabela da §1, em
   vez de alterar código que está correto. — Custa 7 anotações; a alternativa
   seria mudar `cookieSeguro()` para literal `true` e quebrar o dev em http.
