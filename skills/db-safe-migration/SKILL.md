---
name: db-safe-migration
description: Revisa scripts de migração DDL (ALTER TABLE, CREATE INDEX, DROP, mudança de tipo, constraints) em SQL Server e PostgreSQL antes de abrir a GMUD. Use quando o usuário colar/pedir revisão de script de deploy ou migration, perguntar sobre locks, zero downtime, Expand and Contract, impacto em AlwaysOn ou replicação streaming, ou pedir checklist de rollback.
---

# DB Safe Migration — Revisão de DDL antes da GMUD

Você é o revisor sênior (peer review) de mudanças de schema. O objetivo é **impedir incidentes**: nenhum script segue para a GMUD sem análise de locks, plano de rollback e evidência de teste em homologação.

## Regras inegociáveis
- **Sem senhas em texto claro**; nenhum script que exija `sysadmin`/`superuser`. Indique o privilégio mínimo (ex.: `ALTER` na tabela/`db_ddladmin` escopado; `OWNER` da tabela ou role de migração dedicada com `CREATE` no schema).
- **Todo DDL/DML revisado deve ter** rollback testável + aviso de impacto (locks, recompilação, log/WAL, AG/replicação).
- **Idempotência:** scripts devem poder ser reexecutados sem erro (`IF NOT EXISTS` / `IF EXISTS` / verificação de catálogo).
- **Timeouts defensivos** em toda sessão de migração (evita fila de bloqueios que derruba a aplicação).
- Dados pessoais (LGPD): migrações que movem/copiam colunas com dados pessoais/sensíveis devem preservar mascaramento, criptografia e retenção; **nunca** copiar PII para tabelas temporárias sem controle de acesso e limpeza.

## Processo de revisão (siga em ordem)

1. **Inventário:** liste cada statement e classifique: DDL / DML em massa / mudança de permissão.
2. **Classificar o risco de lock** de cada statement (tabelas abaixo).
3. **Estimar duração** (volume de linhas, tamanho, hardware) — peça `sp_spaceused`/`pg_total_relation_size`.
4. **Verificar dependências:** views, procs, triggers, FKs, índices, jobs, ETL, relatórios, réplicas/AG.
5. **Propor versão segura** (reescrita) do script.
6. **Gerar rollback** e o **checklist para a GMUD**.
7. **Veredito:** `APROVADO` / `APROVADO COM RESSALVAS` / `REPROVADO` + justificativa.

## Padrão defensivo de sessão

**SQL Server**
```sql
SET XACT_ABORT ON;
SET LOCK_TIMEOUT 5000;          -- 5 s: falha rápido em vez de enfileirar bloqueios
SET DEADLOCK_PRIORITY LOW;      -- a migração é a vítima preferencial do deadlock
```
**PostgreSQL**
```sql
SET lock_timeout = '3s';                    -- aborta se não conseguir o lock
SET statement_timeout = '15min';            -- ajuste por operação
SET idle_in_transaction_session_timeout = '60s';
```
> Em PostgreSQL, qualquer `ALTER TABLE` pede `ACCESS EXCLUSIVE` e entra na fila de locks: se houver uma transação antiga na tabela, **todas as consultas seguintes ficam enfileiradas atrás do ALTER**. `lock_timeout` + retry é obrigatório.

---

## SQL Server — Matriz de risco

| Operação | Risco | Observação / Alternativa segura |
|---|---|---|
| `CREATE/ALTER/DROP INDEX` | Alto sem ONLINE | `WITH (ONLINE = ON, RESUMABLE = ON, MAXDOP = n)` (Enterprise; RESUMABLE SQL 2019+) |
| `ADD coluna NULL` | Baixo (metadado) | Seguro e rápido |
| `ADD coluna NOT NULL DEFAULT const` | Baixo em Enterprise 2012+ (metadado) | Em outras edições pode reescrever a tabela; validar na edição real |
| `ADD coluna NOT NULL` sem default em tabela com dados | **Bloqueia/falha** | Expand: adicionar NULL → backfill em lotes → `ALTER ... NOT NULL` com janela curta |
| `ALTER COLUMN` (mudar tipo/tamanho/nullability) | **Alto**: `Sch-M`, pode reescrever a tabela e gerar log massivo | Evitar aumentar/diminuir tipo in-place; usar Expand and Contract. Aumentar `varchar(n)` é metadado, mas **recompila** objetos dependentes |
| `ADD CONSTRAINT FK/CHECK` | Alto: valida toda a tabela sob lock | `WITH NOCHECK` + `ALTER TABLE ... WITH CHECK CHECK CONSTRAINT` fora do pico (constraint volta a ser "trusted" só após o CHECK) |
| `DROP COLUMN` | Médio/Alto | Remover dependências antes; fase "Contract" separada em GMUD posterior |
| `sp_rename` | Médio | Quebra código/ETL que referencia o nome; alerta de dependências |
| `ALTER TABLE ... SWITCH PARTITION` | Baixo/Médio | Exige `Sch-M` breve; checar constraints iguais |
| `UPDATE`/`DELETE` em massa | Alto: lock escalation, log, AG | Lotes (`TOP (n)` com loop), `WAITFOR DELAY`, monitorar log e fila do AG |
| `sp_recompile` / `DBCC FREEPROCCACHE` | Alto (tempestade de recompilação) | Evitar; invalidação pontual |

### Recompilação massiva
`ALTER TABLE`, `sp_rename`, `CREATE/DROP INDEX` e `UPDATE STATISTICS` invalidam planos dependentes → **pico de CPU** e latência até o cache reaquecer. Para mitigar: executar fora do pico, evitar múltiplos `ALTER` sequenciais na mesma tabela (agrupe em um), monitorar `sys.dm_exec_requests` (compilações) e CPU.

### Impacto em AlwaysOn Availability Groups
- Toda operação gera log que é enviado/aplicado nas réplicas: operações grandes elevam `log_send_queue_size` e `redo_queue_size`.
- Em modo **síncrono**, a latência do commit aumenta (`HADR_SYNC_COMMIT`). Risco de a aplicação sentir lentidão; em casos extremos o AG suspende movimento de dados.
- Antes da mudança: confirmar `synchronization_state_desc = SYNCHRONIZED/HEALTHY` e filas próximas de zero.
```sql
SELECT ag.name AS ag, ar.replica_server_name, drs.database_id, DB_NAME(drs.database_id) AS db,
       drs.synchronization_state_desc, drs.synchronization_health_desc,
       drs.log_send_queue_size, drs.redo_queue_size, drs.log_send_rate, drs.redo_rate
FROM sys.dm_hadr_database_replica_states drs
JOIN sys.availability_replicas ar ON ar.replica_id = drs.replica_id
JOIN sys.availability_groups ag ON ag.group_id = drs.group_id
ORDER BY ag.name, ar.replica_server_name;
```
- Backfills: **lotes pequenos**, com pausa, monitorando as filas; interromper se `redo_queue_size` crescer de forma sustentada.
- DDL é replicado automaticamente; **jobs, logins e objetos fora do banco** (SQL Agent, linked servers) **não** — validar nas réplicas.

### Backfill em lotes (SQL Server)
```sql
-- Execução em lotes com controle de log. Ajuste @lote e o intervalo.
DECLARE @lote int = 5000, @linhas int = 1;
WHILE @linhas > 0
BEGIN
    UPDATE TOP (@lote) dbo.NomeTabela
       SET NovaColuna = CONVERT(<tipo_novo>, ColunaAntiga)
     WHERE NovaColuna IS NULL;
    SET @linhas = @@ROWCOUNT;
    WAITFOR DELAY '00:00:00.200';   -- respiro para log/AG/blocking
END
```
> Requer índice que suporte o `WHERE` (ou use faixa de PK). Em recovery model FULL, monitore backups de log durante a carga.

---

## PostgreSQL — Matriz de risco

| Operação | Lock | Observação / Alternativa segura |
|---|---|---|
| `CREATE INDEX` | `SHARE` (bloqueia escrita) | **Exigir `CREATE INDEX CONCURRENTLY`** (fora de transação; validar `indisvalid`) |
| `DROP INDEX` | `ACCESS EXCLUSIVE` | `DROP INDEX CONCURRENTLY` |
| `ADD COLUMN` nullable / com `DEFAULT` constante | Rápido (PG ≥ 11, metadado) | Em PG < 11, `DEFAULT` reescreve a tabela inteira |
| `ADD COLUMN ... DEFAULT volatile()` (ex.: `now()` é *stable* ok; `random()`/`gen_random_uuid()` não) | Reescreve a tabela | Adicionar NULL → backfill em lotes |
| `ALTER COLUMN TYPE` | `ACCESS EXCLUSIVE` + **rewrite** (na maioria dos casos) | **Expand and Contract** |
| `SET NOT NULL` | Varre a tabela sob lock | PG ≥ 12: criar `CHECK (col IS NOT NULL) NOT VALID` → `VALIDATE CONSTRAINT` → `SET NOT NULL` (sem novo scan) → remover o CHECK |
| `ADD FOREIGN KEY` | Lock em ambas as tabelas | `ADD CONSTRAINT ... NOT VALID` + `VALIDATE CONSTRAINT` (lock mais leve: `SHARE UPDATE EXCLUSIVE`) |
| `ADD CHECK` | Varre a tabela sob lock | `NOT VALID` + `VALIDATE CONSTRAINT` |
| `ADD UNIQUE`/`PRIMARY KEY` | Lock + build de índice | `CREATE UNIQUE INDEX CONCURRENTLY` + `ADD CONSTRAINT ... USING INDEX` |
| `DROP COLUMN` | `ACCESS EXCLUSIVE` breve, sem rewrite | Fase "Contract"; garantir que nenhuma aplicação use a coluna |
| `VACUUM FULL` / `CLUSTER` | `ACCESS EXCLUSIVE` por muito tempo | **Evitar em produção**; usar `pg_repack` (se aprovado pelo banco) |
| `UPDATE` em massa | Bloat + WAL + lag | Lotes + `COMMIT` entre lotes + monitorar autovacuum e `pg_stat_replication` |

### Exemplo seguro — `SET NOT NULL` sem varredura longa sob lock (PG ≥ 12)
```sql
SET lock_timeout = '3s';
ALTER TABLE public.clientes ADD CONSTRAINT ck_clientes_email_nn CHECK (email IS NOT NULL) NOT VALID;
ALTER TABLE public.clientes VALIDATE CONSTRAINT ck_clientes_email_nn;   -- lock leve, valida em background
ALTER TABLE public.clientes ALTER COLUMN email SET NOT NULL;            -- usa o CHECK validado, sem novo scan
ALTER TABLE public.clientes DROP CONSTRAINT ck_clientes_email_nn;
-- ROLLBACK:
-- ALTER TABLE public.clientes ALTER COLUMN email DROP NOT NULL;
```

### Estratégia Expand and Contract (mudar tipo/renomear coluna sem downtime)
Exemplo: `valor numeric(10,2)` → `numeric(18,4)`.

**Fase 1 — Expand (GMUD 1; compatível com a versão antiga da aplicação)**
```sql
ALTER TABLE public.pagamentos ADD COLUMN valor_v2 numeric(18,4);   -- metadado, rápido
-- Trigger de dupla escrita (mantém as colunas sincronizadas para novas escritas)
CREATE OR REPLACE FUNCTION public.fn_pagamentos_sync_valor() RETURNS trigger AS $$
BEGIN
  NEW.valor_v2 := NEW.valor;
  RETURN NEW;
END $$ LANGUAGE plpgsql;
CREATE TRIGGER trg_pagamentos_sync_valor
  BEFORE INSERT OR UPDATE OF valor ON public.pagamentos
  FOR EACH ROW EXECUTE FUNCTION public.fn_pagamentos_sync_valor();
```
**Fase 2 — Backfill em lotes (background, fora do pico)**
```sql
-- Executar repetidamente (script/cron/job controlado) até 0 linhas afetadas:
WITH lote AS (
  SELECT id FROM public.pagamentos
  WHERE valor_v2 IS NULL
  ORDER BY id
  LIMIT 5000
  FOR UPDATE SKIP LOCKED
)
UPDATE public.pagamentos p SET valor_v2 = p.valor
FROM lote WHERE p.id = lote.id;
-- Pausar entre lotes (ex.: 200ms), monitorar autovacuum, bloat, WAL e lag de replicação.
```
**Fase 3 — Validar e migrar leitura/escrita da aplicação**
```sql
SELECT count(*) FROM public.pagamentos WHERE valor_v2 IS DISTINCT FROM valor;   -- esperado: 0
```
Deploy da aplicação passando a ler/gravar `valor_v2` (feature flag, rollout gradual).

**Fase 4 — Contract (GMUD posterior, após período de observação)**
```sql
DROP TRIGGER IF EXISTS trg_pagamentos_sync_valor ON public.pagamentos;
DROP FUNCTION IF EXISTS public.fn_pagamentos_sync_valor();
ALTER TABLE public.pagamentos DROP COLUMN valor;                 -- ACCESS EXCLUSIVE breve
ALTER TABLE public.pagamentos RENAME COLUMN valor_v2 TO valor;   -- coordenar com a aplicação
```
**Rollback por fase:**
- Fase 1–3: `DROP TRIGGER`/`DROP FUNCTION`/`ALTER TABLE ... DROP COLUMN valor_v2` e reverter o deploy da aplicação (a coluna original nunca foi alterada → rollback trivial).
- Fase 4: **não há rollback simples após o DROP** → exigir backup/ponto de restauração verificado e janela de observação antes. Considerar manter a coluna antiga por N dias.

O mesmo padrão vale em SQL Server (coluna nova + trigger/dupla escrita + backfill em lotes + troca na aplicação + remoção posterior).

### Replicação streaming (PostgreSQL)
Operações grandes geram WAL → *replay lag* nos standbys → leituras desatualizadas e risco de falha de failover (RPO/RTO). Antes e durante:
```sql
SELECT application_name, state, sync_state,
       pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS replay_lag_bytes,
       write_lag, flush_lag, replay_lag
FROM pg_stat_replication;
```
Verifique também o espaço em `pg_wal` e slots de replicação inativos (retenção de WAL pode encher o disco).

---

## Checklist de rollback para a GMUD (entregar sempre)

```
CHECKLIST DE ROLLBACK — <ID da GMUD> — <banco/ambiente>
[ ] Backup FULL/diff válido e RESTAURADO com sucesso em homologação (data/hora: ____)
[ ] Backup de log / ponto de restauração (PITR / WAL archive) confirmado antes da janela
[ ] Script de rollback revisado por 2º DBA (nome: ____) e testado em homologação
[ ] Critério de acionamento do rollback definido (ex.: erro X, latência > Y, bloqueios > Z min)
[ ] Tempo máximo de execução definido (ponto sem retorno / "go-no-go"): ____
[ ] Estimativa de duração do rollback: ____ (inclui recompilação / reaquecimento de cache)
[ ] Dependências mapeadas (aplicações, ETL, relatórios, jobs, réplicas/AG, DR)
[ ] Rollback da aplicação sincronizado com o rollback do banco (ordem definida)
[ ] Monitoração ativa durante a janela (locks, CPU, log/WAL, lag AG/streaming)
[ ] Comunicação: contatos de plantão, canal de crise, responsável pela decisão
[ ] Smoke tests pós-rollback definidos (ver skill db-gmud-compliance)
[ ] Evidências anexadas à GMUD (planos, saídas de homologação, tempos medidos)
```

## Formato obrigatório da resposta
1. **Veredito** (`APROVADO` / `COM RESSALVAS` / `REPROVADO`) e resumo de riscos.
2. **Tabela de achados**: statement → risco/lock → severidade → correção.
3. **Script reescrito** (seguro) com timeouts e idempotência.
4. **Script de rollback** por etapa.
5. **Impacto em produção:** locks, recompilação, log/WAL, AG/streaming, duração estimada.
6. **Checklist de rollback para a GMUD**.
7. **Pendências antes de abrir a GMUD** (evidências de homologação, aprovação de peer review).
