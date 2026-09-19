# SPEC-031 — Tags v0.1.0 pelo caminho de release, provado antes de escalar

**Estado:** proposta
**Data:** 16/09/2026
**Card:** navyr-io/navyr-deploy#3
**ADR:** 0005 (separação de `ci.yml`, `publish.yml` e `release.yml` por resolução de permissões na partida)
**Relacionada:** SPEC-030 (gate do gosec — não bloqueia esta), navyr-deploy#4 (re-escopado em 16/09)

## 1. Problema

**0 de 11 repositórios têm tag `v0.1.0`.** Os outros dois artefatos da entrega
existem: `CHANGELOG.md` em 11 de 11, `release.yml` em 11 de 11. O card `#3`
media 22 de 33 artefatos — 67%.

O card declarava **"Destrava com: GitHub Team"**. Isso era falso, e o próprio
corpo dele explica o bloqueio verdadeiro:

> a tag dispara o workflow que publica a imagem versionada. Com o CI recusando
> iniciar jobs, a tag produz release sem imagem — pior que não ter tag, porque
> parece entrega.

Esse bloqueio acabou. O teto de minutos do plano Free resetou no ciclo de
cobrança e **o CI voltou a executar em 01/09**; consumo de setembro: 146 de 2000
minutos, novo reset em 01/10. A prova é a duração dos jobs, não a ausência de
erro: durante o bloqueio morriam em 2–4s sem rodar passo nenhum (`navyr-auth`
run 33432615584 — `started 19:48:54 → completed 19:48:56`); depois do reset há
runs de 244s, 454s e 597s com passos reais.

### O gate quebrado do gosec não alcança este caminho

Medido em `navyr-gateway/.github/workflows/release.yml`: **nenhuma dependência
de `ci.yml`**.

```yaml
on:
  push:
    tags:
      - 'v[0-9]+.[0-9]+.[0-9]+'
      - 'v[0-9]+.[0-9]+.[0-9]+-*'
jobs:
  imagem:
    uses: navyr-io/.github/.github/workflows/publish-image.yml@main
  release:
    needs: imagem
    uses: navyr-io/.github/.github/workflows/release.yml@main
```

O `gosec` reprova o `ci.yml` (SPEC-030) e não toca aqui. E o
`publish-image.yml` roda **trivy** no próprio caminho, então a etapa 4 do
pipeline obrigatório fica coberta sem depender da SPEC-030.

### O risco que esta spec existe para não correr

O medo registrado no card continua legítimo, só mudou de causa: **uma tag que
produz release sem imagem é pior que não ter tag, porque parece entrega.** O CI
voltar não prova que o caminho de release funciona — ele nunca foi executado
ponta a ponta. Há **0 runs** do workflow `Release` na história dos repositórios.

Criar 11 tags para descobrir isso é apostar onze vezes o mesmo palpite.

## 2. Regras e critérios de aceitação

| # | Regra | Como se prova |
|---|---|---|
| 1 | O caminho de release é provado em **um** repositório antes de qualquer outro | `gh run list --repo navyr-io/<repo> --workflow Release` mostra 1 run `success` |
| 2 | A imagem versionada existe no GHCR antes de a Release ser anunciada | `docker manifest inspect ghcr.io/navyr-io/<img>:<tag>` retorna 0 — rodado **antes** de criar tag definitiva |
| 3 | A prova não anuncia entrega | A tag do ciclo 1 é pré-release (`-rc1`); a GitHub Release correspondente fica marcada como pre-release |
| 4 | O trivy do caminho de release realmente roda e reprova o que deve | Saída do passo trivy colada; se houver CVE crítica, ela aparece e o job falha |
| 5 | A Release é gerada a partir do `CHANGELOG.md`, não de texto solto | Corpo da Release publicada bate com a seção correspondente do CHANGELOG |
| 6 | Só depois de 1–5 as 11 tags definitivas são criadas | `gh api /repos/navyr-io/<repo>/tags` mostra `v0.1.0` em 11 de 11 |
| 7 | Nenhum artefato de prova sobrevive à entrega | Tag e Release `-rc1` removidas ao fim, ou declaradas mantidas com motivo |

## 3. Fora de escopo

- **SPEC-030 / `#18`** — o gate do gosec. Medido: não bloqueia este caminho.
- **`#19`** — lint e testes de integração.
- **`#1` branch protection** — bloqueio real de plano (`403 — Upgrade`,
  `plan.name = free`, repos privados). Nada aqui o destrava.
- **`#4` assinar GitHub Team** — decisão de negócio, re-escopada em 16/09.
- **Merge dos 28 PRs do Dependabot** — depende da SPEC-030, não desta.
- **Conteúdo do `CHANGELOG.md`** — assume-se correto; revisá-lo é outro trabalho.

## 4. Plano de execução

| Ciclo | Entrega | Critério de pronto |
|---|---|---|
| 1 | Escolher o repositório de prova e conferir o estado de partida | `gh api /repos/navyr-io/<repo>/tags --jq length` devolve `0`; `CHANGELOG.md` tem seção `0.1.0`; saída colada |
| 2 | Tag `v0.1.0-rc1` no repositório de prova, e acompanhar o run | Run do workflow `Release` termina `success`; log dos dois jobs (`imagem`, `release`) colado |
| 3 | Verificar o artefato, não o job | `docker manifest inspect ghcr.io/navyr-io/<img>:v0.1.0-rc1` retorna 0 e a Release existe com corpo vindo do CHANGELOG — critérios 2, 4 e 5 |
| 4 | Decidir e registrar: escalar para os 11, ou corrigir o que o ciclo 2–3 revelou | Nota de ciclo com a decisão e a alternativa rejeitada |
| 5 | Tags `v0.1.0` definitivas nos 11 repositórios, **uma a uma, verificando a imagem antes da seguinte** | Critério 6 — `tags --jq length` ≥ 1 em 11 de 11, com `manifest inspect` colado para cada |
| 6 | Limpeza dos artefatos de prova e convergência | Critério 7; percorrer §2 item a item com saída colada |

Cada ciclo termina em commit semântico com a suíte verde. O ciclo 5 é o único
irreversível na prática — tag pode ser apagada, mas release anunciada já foi
vista.

## 5. Pipeline de entrega — o que se aplica

| Etapa | Status | Observação |
|---|---|---|
| 1. Versionamento | ✅ | a tag **é** o versionamento; commit semântico nas notas de ciclo |
| 2. Testes unitários | ✅ | suíte existente dos serviços; esta spec não altera código de aplicação |
| 3. Build de imagem | ✅ | executado pelo `publish-image.yml` dentro do caminho de release |
| 4. Scan de imagem | ✅ | **trivy roda no próprio `publish-image.yml`** — não depende da SPEC-030 |
| 5. Push para registry | ✅ | é o objeto do critério 2 |
| 6. Teste funcional | ✅ | `docker manifest inspect` da imagem versionada; puxar a imagem e subir o compose |
| 7. Teste E2E | ⛔ N/A | nenhuma mudança de comportamento de produto; a imagem é a mesma, com outra tag |

Este é o primeiro item em muitas semanas em que **as 7 etapas são executáveis de
verdade** — é o que o retorno do CI destravou.

**Correção ao texto da skill:** ela afirma que o CI está bloqueado pelo plano
Free desde 19/08/2026 e que a verificação é substituída pelo `pre-push`. Deixou
de ser verdade em 01/09.

## 6. Decisões em aberto (nenhuma bloqueia o início)

1. **Qual repositório serve de prova.** Assumo `navyr-billing`: é o de menor
   superfície entre os que têm CI passando hoje (o `CI` dele fechou `success` em
   01/09, 331s), então uma falha aponta para o caminho de release e não para o
   serviço. — Custa pouco se errado: troca-se o alvo e repete o ciclo 2.

2. **Se o ciclo 2 falhar por permissão de `packages: write`.** Assumo que é
   configuração do workflow reutilizável, não do plano — há precedente em
   `41b57e8` ("concede `packages: write` ao chamar o workflow reutilizável"). —
   Custa um ciclo extra; não muda o escopo.

3. **A tag `-rc1` fica ou sai.** Assumo removê-la no ciclo 6, junto com a
   pre-release, para não deixar duas versões concorrentes visíveis a terceiros.
   — Custa perder o rastro da prova; mitigo colando os logs na nota de ciclo.

4. **Ordem dos 11 no ciclo 5.** Assumo começar pelos que não têm dependentes
   (`navyr-collector`, `navyr-community`) e deixar `navyr-gateway` e
   `navyr-orchestrator` por último, por serem o caminho que o agente e a SPA
   usam. — Custa nada se errado; é ordem, não conteúdo.

5. **`navyr-docs` é público e esta spec descreve o caminho de publicação.**
   Assumo publicar, como as 29 anteriores e a SPEC-030. — Custa expor a
   mecânica de release; nenhum segredo está aqui.
