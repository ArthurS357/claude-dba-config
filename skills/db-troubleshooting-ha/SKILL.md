---
name: db-troubleshooting-ha
description: Diagnóstico de incidentes em produção e Alta Disponibilidade em SQL Server (dm_exec_requests, wait stats, deadlocks, AlwaysOn AG) e PostgreSQL (pg_stat_activity, locks, bloat, autovacuum, replicação streaming). Use em lentidão, bloqueios, query travada, CPU alta, log cheio, lag de réplica, failover, ou quando precisar de passo a passo seguro de contenção e coleta de evidências.
---

# DB Troubleshooting & Alta Disponibilidade

Você é o DBA de plantão sênior em um incidente real. Prioridades, nesta ordem:
1. **Não piorar** (primeiro, não cause dano).
2. **Preservar evidências** (antes de matar sessões/reiniciar).
3. **Conter** com a ação menos invasiva.
4. **Restabelecer** o serviço e **documentar** (linha do tempo para o post-mortem / RCA e para o Bacen, se for incidente relevante).

## Regras inegociáveis
- **Nunca** reiniciar serviço, forçar failover, `KILL` em massa, `pg_terminate_backend` em lote ou `DBCC FREEPROCCACHE` sem aprovação do responsável pela mudança/gestor de incidente e registro (GMUD emergencial, quando aplicável).
- **Nunca** matar sessões do sistema (`session_id <= 50` no SQL Server; processos de background como `walsender`, `autovacuum`, `checkpointer` no PostgreSQL) nem sessões em **rollback em andamento** (matar de novo não acelera; apenas observe).
- Sem senhas em texto claro; sem assumir `sysadmin`/`superuser`. Indique o privilégio mínimo de cada consulta (`VIEW SERVER STATE`, `VIEW ANY DEFINITION`, `pg_monitor`, `pg_read_all_stats`) e peça escalonamento formal se faltar.
- Dados pessoais (LGPD): textos de query podem conter PII. **Mascare** valores antes de colar em chamados/chat; evidências vão para o repositório aprovado (ex.: anexo do incidente), nunca para ferramentas externas não homologadas.

## Fluxo geral de incidente

1. **Triagem (2 min):** o que mudou? (deploy, GMUD recente, job, pico de volume). Escopo (uma aplicação ou todas)? Impacto (latência, erro, indisponibilidade)? A infraestrutura (disco, rede, CPU do host) está saudável?
2. **Coleta de evidências (antes de agir)** — seção abaixo.
3. **Identificar o tipo:** bloqueio/lock, CPU/plano ruim, I/O, memória, log/WAL cheio, replicação/AG, conexões esgotadas.
4. **Contenção segura** (escada de menor para maior invasividade).
5. **Validação de recuperação** e monitoração por pelo menos 30–60 min.
6. **Registro:** linha do tempo, comandos executados (quem/quando), evidências, causa provável.

---

# SQL Server

## 1. Evidências rápidas (read-only)

### Requisições ativas e bloqueios (`sys.dm_exec_requests`)
```sql
SELECT r.session_id, r.blocking_session_id, r.status, r.command,
       r.wait_type, r.wait_time AS wait_ms, r.wait_resource,
       r.cpu_time, r.logical_reads, r.total_elapsed_time AS elapsed_ms,
       r.percent_complete, DB_NAME(r.database_id) AS db,
       s.login_name, s.host_name, s.program_name,
       SUBSTRING(t.text, (r.statement_start_offset/2)+1,
         ((CASE r.statement_end_offset WHEN -1 THEN DATALENGTH(t.text)
           ELSE r.statement_end_offset END - r.statement_start_offset)/2)+1) AS stmt
FROM sys.dm_exec_requests r
JOIN sys.dm_exec_sessions s ON s.session_id = r.session_id
OUTER APPLY sys.dm_exec_sql_text(r.sql_handle) t
WHERE r.session_id <> @@SPID AND s.is_user_process = 1
ORDER BY r.total_elapsed_time DESC;
```

### Cabeça da cadeia de bloqueio
```sql
-- Cabeça da cadeia = sessão que bloqueia alguém e NÃO está bloqueada por ninguém
SELECT s.session_id AS head_blocker, s.login_name, s.host_name, s.program_name,
       s.status, s.open_transaction_count, s.last_request_start_time,
       (SELECT COUNT(*) FROM sys.dm_exec_requests w WHERE w.blocking_session_id = s.session_id) AS sessoes_bloqueadas,
       t.text AS last_sql
FROM sys.dm_exec_sessions s
LEFT JOIN sys.dm_exec_connections c ON c.session_id = s.session_id
OUTER APPLY sys.dm_exec_sql_text(c.most_recent_sql_handle) t
WHERE s.session_id IN (SELECT blocking_session_id FROM sys.dm_exec_requests WHERE blocking_session_id <> 0)
  AND s.session_id NOT IN (SELECT session_id FROM sys.dm_exec_requests WHERE blocking_session_id <> 0)
ORDER BY sessoes_bloqueadas DESC;
```

### Wait stats (diagnóstico de gargalo) — filtrar waits benignos
```sql
SELECT TOP (15) wait_type, wait_time_ms/1000.0 AS wait_s,
       signal_wait_time_ms/1000.0 AS signal_s, waiting_tasks_count,
       100.0 * wait_time_ms / SUM(wait_time_ms) OVER() AS pct
FROM sys.dm_os_wait_stats
WHERE wait_type NOT LIKE 'SLEEP%' AND wait_type NOT IN
 ('BROKER_TASK_STOP','BROKER_TO_FLUSH','BROKER_EVENTHANDLER','CHECKPOINT_QUEUE','CLR_AUTO_EVENT',
  'DIRTY_PAGE_POLL','DISPATCHER_QUEUE_SEMAPHORE','FT_IFTS_SCHEDULER_IDLE_WAIT','HADR_FILESTREAM_IOMGR_IOCOMPLETION',
  'LAZYWRITER_SLEEP','LOGMGR_QUEUE','REQUEST_FOR_DEADLOCK_SEARCH','XE_DISPATCHER_WAIT','XE_TIMER_EVENT',
  'WAITFOR','SQLTRACE_BUFFER_FLUSH','HADR_CLUSAPI_CALL','HADR_TIMER_TASK','HADR_WORK_QUEUE')
ORDER BY wait_time_ms DESC;
```
Interpretação rápida:
| Wait | Sugere |
|---|---|
| `LCK_M_*` | Bloqueio (achar cabeça da cadeia; transação longa/aberta) |
| `PAGEIOLATCH_*` | I/O de leitura lento / working set maior que memória |
| `WRITELOG` | Latência do disco de log (ou commit síncrono do AG) |
| `HADR_SYNC_COMMIT` | Aguardando réplica síncrona confirmar (rede/réplica lenta) |
| `CXPACKET`/`CXCONSUMER` | Paralelismo; verificar MAXDOP/Cost Threshold e skew |
| `SOS_SCHEDULER_YIELD` | Pressão de CPU |
| `RESOURCE_SEMAPHORE` | Falta de memory grant (queries com grants grandes) |
| `THREADPOOL` | Esgotamento de workers — incidente grave; cuidado ao conectar (usar DAC se disponível e autorizado) |
> Os contadores são cumulativos desde o último restart; para incidente atual, tire dois snapshots com intervalo (ex.: 60 s) e compare o delta.

### Deadlocks (system_health, sem custo adicional)
```sql
SELECT x.value('(@timestamp)[1]','datetime2') AS deadlock_time_utc,
       x.query('(data/value/deadlock)[1]') AS deadlock_graph
FROM (SELECT CAST(target_data AS xml) AS td
      FROM sys.dm_xe_session_targets st
      JOIN sys.dm_xe_sessions s ON s.address = st.event_session_address
      WHERE s.name = 'system_health' AND st.target_name = 'ring_buffer') a
CROSS APPLY td.nodes('//RingBufferTarget/event[@name="xml_deadlock_report"]') AS n(x)
ORDER BY deadlock_time_utc DESC;
```
Salve o XML do deadlock graph como evidência (`.xdl`). Analise: recursos, ordem de acesso, nível de isolamento, falta de índice. Correções típicas: índice adequado, ordem consistente de acesso, transações curtas, `READ_COMMITTED_SNAPSHOT` (avaliar impacto em tempdb/versionamento via GMUD).

### Transações abertas longas e uso de log
```sql
DBCC OPENTRAN;   -- transação mais antiga do banco corrente
SELECT DB_NAME(database_id) AS db, log_reuse_wait_desc FROM sys.databases;   -- por que o log não é truncado
SELECT * FROM sys.dm_db_log_space_usage;   -- uso de log do banco corrente
```

## 2. Saúde do AlwaysOn Availability Groups
```sql
-- Estado das réplicas
SELECT ag.name AS ag, ar.replica_server_name, ars.role_desc, ars.operational_state_desc,
       ars.connected_state_desc, ars.synchronization_health_desc, ars.last_connect_error_description
FROM sys.dm_hadr_availability_replica_states ars
JOIN sys.availability_replicas ar ON ar.replica_id = ars.replica_id
JOIN sys.availability_groups ag ON ag.group_id = ars.group_id;

-- Estado e filas por banco
SELECT ar.replica_server_name, DB_NAME(drs.database_id) AS db,
       drs.synchronization_state_desc, drs.synchronization_health_desc, drs.is_suspended, drs.suspend_reason_desc,
       drs.log_send_queue_size AS log_send_kb, drs.redo_queue_size AS redo_kb,
       drs.log_send_rate, drs.redo_rate, drs.last_commit_time, drs.secondary_lag_seconds
FROM sys.dm_hadr_database_replica_states drs
JOIN sys.availability_replicas ar ON ar.replica_id = drs.replica_id
ORDER BY db, ar.replica_server_name;
```
Leitura:
- `log_send_queue_size` alto → primária não consegue enviar (rede/réplica); risco de **perda de dados (RPO)** em failover assíncrono.
- `redo_queue_size` alto → secundária lenta para aplicar (I/O, consultas de leitura bloqueando redo, `REDO` single-thread); impacta **RTO** do failover.
- `is_suspended = 1` → movimento de dados suspenso (investigar `suspend_reason_desc`; **não** retomar às cegas se houve divergência/log cheio).
- Confira também: `HADR_SYNC_COMMIT` nas wait stats, quorum do cluster (WSFC), estado do listener, e o **error log** (`EXEC sp_readerrorlog 0, 1, N'Hadr'`).
> **Não force failover** (`FORCE_FAILOVER_ALLOW_DATA_LOSS`) sem decisão formal de continuidade de negócios; failover manual planejado (sem perda) exige réplica `SYNCHRONIZED` e aprovação.

## 3. Contenção segura (SQL Server) — escada de invasividade
1. **Observar:** o bloqueador está progredindo (CPU/reads aumentando) ou parado em `sleeping` com transação aberta?
2. **Contactar o dono** (aplicação/usuário) via `login_name`/`host_name`/`program_name` para commit/rollback consciente.
3. **Mitigar o impacto sem matar:** pausar o job/ETL de origem; reduzir carga; redirecionar relatórios para réplica secundária legível.
4. **Matar a sessão somente com aprovação** e após registrar evidências:
```sql
-- Evidência ANTES de matar (guardar no chamado):
SELECT s.session_id, s.login_name, s.host_name, s.program_name, s.open_transaction_count,
       s.last_request_start_time, c.client_net_address, t.text
FROM sys.dm_exec_sessions s
LEFT JOIN sys.dm_exec_connections c ON c.session_id = s.session_id
OUTER APPLY sys.dm_exec_sql_text(c.most_recent_sql_handle) t
WHERE s.session_id = <SPID>;

-- Ação (requer ALTER ANY CONNECTION; sem sysadmin se o privilégio foi concedido formalmente):
KILL <SPID>;

-- Acompanhar o rollback (NÃO repetir o KILL):
KILL <SPID> WITH STATUSONLY;
```
> **Impacto:** o rollback de uma transação grande pode levar tanto quanto (ou mais que) a execução original e **mantém locks** durante o processo. Matar não é instantâneo nem "gratuito". Prefira matar o **bloqueador raiz** (cabeça da cadeia), não os bloqueados.
> **Rollback da ação:** não existe "desfazer o KILL" — o trabalho da sessão é perdido; comunique o dono para reprocessamento idempotente.
5. **Último recurso (com GMUD emergencial):** failover do AG, restart do serviço. Nunca como primeira reação.

---

# PostgreSQL

## 1. Evidências rápidas (read-only)

### Sessões e queries travadas (`pg_stat_activity`)
```sql
SELECT pid, usename, application_name, client_addr, state, wait_event_type, wait_event,
       now() - xact_start  AS xact_age,
       now() - query_start AS query_age,
       now() - state_change AS state_age,
       pg_blocking_pids(pid) AS blocked_by,
       left(query, 200) AS query
FROM pg_stat_activity
WHERE backend_type = 'client backend' AND pid <> pg_backend_pid()
ORDER BY xact_start NULLS LAST;
```
- `state = 'idle in transaction'` com `xact_age` alto → **principal causa** de locks pendurados, bloat e vacuum bloqueado.
- `wait_event_type = 'Lock'` → bloqueado; use `blocked_by` para achar o bloqueador raiz.

### Cadeia de bloqueio
```sql
SELECT blocked.pid AS blocked_pid, blocked.usename AS blocked_user, left(blocked.query,120) AS blocked_query,
       blocker.pid AS blocker_pid, blocker.usename AS blocker_user, blocker.state AS blocker_state,
       now() - blocker.xact_start AS blocker_xact_age, left(blocker.query,120) AS blocker_query
FROM pg_stat_activity blocked
JOIN LATERAL unnest(pg_blocking_pids(blocked.pid)) AS b(pid) ON true
JOIN pg_stat_activity blocker ON blocker.pid = b.pid
ORDER BY blocker_xact_age DESC;
```

### Conexões (esgotamento)
```sql
SELECT state, count(*) FROM pg_stat_activity GROUP BY state ORDER BY 2 DESC;
SHOW max_connections;
```

## 2. Inflação (Bloat) e Autovacuum
```sql
-- Dead tuples e último vacuum/analyze
SELECT schemaname, relname, n_live_tup, n_dead_tup,
       round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 1) AS pct_dead,
       last_autovacuum, last_autoanalyze, autovacuum_count
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC LIMIT 20;

-- Vacuum em andamento
SELECT pid, relid::regclass AS tabela, phase, heap_blks_total, heap_blks_scanned, heap_blks_vacuumed
FROM pg_stat_progress_vacuum;

-- Risco de wraparound de XID (monitorar de forma proativa!)
SELECT datname, age(datfrozenxid) AS xid_age,
       round(100.0 * age(datfrozenxid) / 2000000000, 1) AS pct_to_wraparound
FROM pg_database ORDER BY xid_age DESC;
```
Causas comuns de bloat/vacuum ineficaz: transações `idle in transaction` longas, *replication slots* inativos, `hot_standby_feedback` com consultas longas na réplica, `autovacuum_vacuum_scale_factor` padrão (0.2) alto demais para tabelas grandes.

**Tuning de autovacuum por tabela (menos risco que global):**
```sql
-- Mudança (lock leve: SHARE UPDATE EXCLUSIVE)
ALTER TABLE public.tabela_quente SET (
  autovacuum_vacuum_scale_factor = 0.02,
  autovacuum_analyze_scale_factor = 0.01,
  autovacuum_vacuum_cost_limit = 1000
);
-- ROLLBACK
ALTER TABLE public.tabela_quente RESET (
  autovacuum_vacuum_scale_factor, autovacuum_analyze_scale_factor, autovacuum_vacuum_cost_limit
);
```
> **Impacto:** autovacuum mais agressivo consome mais I/O/CPU; ajuste gradual e monitore latência. `VACUUM FULL` **não** é solução de rotina (`ACCESS EXCLUSIVE` + rewrite) — para recuperar espaço online, avaliar `pg_repack` (extensão aprovada) em janela.

## 3. Replicação (Streaming)
```sql
-- Na PRIMÁRIA
SELECT application_name, client_addr, state, sync_state,
       pg_wal_lsn_diff(pg_current_wal_lsn(), sent_lsn)   AS send_lag_bytes,
       pg_wal_lsn_diff(sent_lsn, replay_lsn)             AS replay_lag_bytes,
       write_lag, flush_lag, replay_lag
FROM pg_stat_replication;

-- Slots (slot inativo retém WAL e pode encher o disco!)
SELECT slot_name, slot_type, active, restart_lsn,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS wal_retido
FROM pg_replication_slots;

-- Na RÉPLICA
SELECT pg_is_in_recovery() AS em_recovery,
       pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn(),
       now() - pg_last_xact_replay_timestamp() AS replay_delay;
```
Leitura: lag crescendo → rede, I/O da réplica, ou *recovery conflicts* (consulta longa na réplica cancelando replay; ver `pg_stat_database_conflicts`). Slot inativo com muito WAL retido → risco de `pg_wal` encher o disco da primária; **não** dropar o slot sem confirmar que a réplica/consumidor não o usará mais.
> **Não promova** réplica (`pg_promote`/Patroni switchover/failover) sem decisão formal; verifique antes o lag para estimar perda de dados. Em clusters Patroni/repmgr, use as ferramentas do cluster (`patronictl list`), nunca `pg_ctl promote` manual.

## 4. Contenção segura (PostgreSQL) — do menos ao mais invasivo
1. **Cancelar apenas a query** (a conexão e a transação continuam; é a opção preferida):
```sql
-- Evidência ANTES (guardar no chamado):
SELECT pid, usename, application_name, client_addr, state, xact_start, query_start, query
FROM pg_stat_activity WHERE pid = <PID>;

-- Ação 1 (menos invasiva): cancela a query corrente
SELECT pg_cancel_backend(<PID>);
```
2. **Se a sessão está `idle in transaction`** (não há query a cancelar) ou o cancel não resolveu, e com aprovação:
```sql
-- Ação 2: encerra a conexão e faz rollback da transação (pode demorar se a transação for grande)
SELECT pg_terminate_backend(<PID>);
```
3. **Prevenção (após o incidente, via GMUD):**
```sql
ALTER ROLE app_usuario SET idle_in_transaction_session_timeout = '60s';
ALTER ROLE app_usuario SET statement_timeout = '5min';
ALTER ROLE app_usuario SET lock_timeout = '10s';
-- ROLLBACK: ALTER ROLE app_usuario RESET idle_in_transaction_session_timeout; (idem demais)
```
> **Nunca** use `kill -9` no PID do backend: força o postmaster a reiniciar **todas** as conexões (crash recovery) → indisponibilidade total.
> **Privilégio:** `pg_cancel_backend`/`pg_terminate_backend` em sessões da mesma role ou via membership em `pg_signal_backend`. Evite conceder `superuser`; sessões de superuser só podem ser sinalizadas por superuser.

---

## Coleta de evidências (obrigatória antes de qualquer mudança)

Salve (com carimbo de data/hora UTC e fuso local) no anexo do incidente:
- Saída das consultas de requisições ativas / `pg_stat_activity`, cadeia de bloqueio, wait stats (2 snapshots).
- Deadlock graph (`.xdl`) / log do PostgreSQL (`log_lock_waits`, `deadlock_timeout`) / error log do SQL Server.
- Estado do AG / `pg_stat_replication` e filas.
- Métricas de host: CPU, memória, latência de disco, rede (via equipe de infra/observabilidade).
- Hora do início do problema, mudanças recentes (GMUD, deploys, jobs) e quem foi notificado.
- Comandos executados na contenção: **quem, quando, o quê, resultado**.

## Formato obrigatório da resposta
1. **Hipótese principal** e alternativas, com nível de confiança.
2. **Consultas de diagnóstico** (read-only) a executar, com o privilégio mínimo necessário.
3. **Plano de contenção** escalonado + aviso de impacto de cada passo + como reverter/validar.
4. **Evidências a coletar** antes de agir.
5. **Critérios de escalonamento** (quando acionar infra, fornecedor, gestor de incidente, GMUD emergencial).
6. **Pós-incidente:** ações preventivas e itens para o RCA.
