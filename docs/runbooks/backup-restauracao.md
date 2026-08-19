# Backup e restauração do banco

## Antes de qualquer coisa: os scripts só cobrem Compose

`scripts/ops/backup_postgres.sh` e `restore_postgres.sh` executam
`docker compose exec db`. **Não funcionam na instalação Kubernetes**, que é o
caminho de produção. Para Kubernetes, use os comandos `kubectl` desta página.

Isso é dívida conhecida, registrada em
[achados-abertos.md](../achados-abertos.md).

## Backup — Kubernetes

```bash
NS=navyr
POD=$(kubectl -n "$NS" get pod -l app=navyr-postgres -o jsonpath='{.items[0].metadata.name}')
ARQ="navyr-$(date +%Y%m%d-%H%M%S).sql.gz"

kubectl -n "$NS" exec "$POD" -- \
  env PGPASSWORD="$(kubectl -n "$NS" get secret navyr-secrets -o jsonpath='{.data.postgres_password}' | base64 -d)" \
  pg_dump -U postgres -d navyr | gzip > "$ARQ"
```

**Verifique que o backup não está vazio nem truncado** — este passo é o que
separa ter backup de achar que tem:

```bash
gzip -t "$ARQ" && echo "arquivo íntegro"
gzip -dc "$ARQ" | grep -c 'CREATE TABLE'   # deve ser > 0
ls -lh "$ARQ"
```

Um dump de banco vazio também passa no `gzip -t`. A contagem de `CREATE TABLE`
é o que distingue.

## Backup — Compose

```bash
export NAVYR_POSTGRES_PASSWORD='<senha>'
./scripts/ops/backup_postgres.sh
# grava em navyr-deploy/backups/navyr-<timestamp>.sql.gz
```

## Restauração

> **Restauração não apaga o que existe.** O dump contém `CREATE TABLE`, que
> falha se a tabela já existir, e `INSERT`, que duplica linha. Restaurar por
> cima de um banco populado produz um estado pior que o original.
> Restaure em banco vazio.

```bash
NS=navyr
POD=$(kubectl -n "$NS" get pod -l app=navyr-postgres -o jsonpath='{.items[0].metadata.name}')
SENHA=$(kubectl -n "$NS" get secret navyr-secrets -o jsonpath='{.data.postgres_password}' | base64 -d)

# 1. Pare quem escreve, senão a restauração compete com a aplicação.
kubectl -n "$NS" scale deploy navyr-auth navyr-billing navyr-orchestrator navyr-community --replicas=0

# 2. Recrie o banco vazio.
kubectl -n "$NS" exec "$POD" -- env PGPASSWORD="$SENHA" psql -U postgres -d postgres \
  -c 'DROP DATABASE IF EXISTS navyr;' -c 'CREATE DATABASE navyr;'

# 3. Restaure.
gzip -dc navyr-<timestamp>.sql.gz | kubectl -n "$NS" exec -i "$POD" -- \
  env PGPASSWORD="$SENHA" psql -U postgres -d navyr

# 4. Suba de volta.
kubectl -n "$NS" scale deploy navyr-auth navyr-billing navyr-orchestrator navyr-community --replicas=2
```

## Verificação

```bash
# Os serviços voltaram a ficar prontos?
kubectl -n "$NS" get pods

# As migrations reconhecem o schema? Erro aqui significa dump de versão
# diferente da imagem em execução.
kubectl -n "$NS" logs -l app=navyr-auth --tail=20 | grep -i migration
```

Se as migrations falharem, o dump é de uma versão de schema incompatível com a
imagem atual. Restaurar não resolve — é preciso a imagem correspondente ao
dump, ou aplicar as migrations intermediárias.
