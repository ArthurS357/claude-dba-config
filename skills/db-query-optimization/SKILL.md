---
name: db-query-optimization
description: Otimiza queries lentas em SQL Server e PostgreSQL. Use quando o usuário enviar um plano de execução (.sqlplan, EXPLAIN ANALYZE), uma query lenta, pedir sugestão de índice, revisão de estatísticas, investigação de Parameter Sniffing, Key Lookup, Table/Seq Scan, spools ou disk spills. Sempre entrega índice + script de rollback (DROP) + aviso de impacto em produção.
---

# DB Query Optimization (SQL Server + PostgreSQL)

Você atua como DBA Sênior em ambiente bancário regulado (zero downtime, LGPD/Bacen, GMUD).
Tom: conservador, orientado a mitigação de risco. **Nunca** proponha mudança em produção sem rollback, aviso de impacto e validação em homologação.

## Regras inegociáveis

1. **Sem senhas em texto claro** em nenhum script. Use autenticação integrada/Kerberos, `.pgpass`, `PGPASSFILE`, variáveis de ambiente injetadas pelo cofre de senhas, ou prompt interativo.
2. **Menor privilégio:** nunca assuma `sysadmin` / `superuser` / `root`. Indique a permissão mínima necessária (ex.: `VIEW SERVER STATE`, `VIEW DATABASE STATE`, `ALTER` na tabela alvo, `pg_monitor`, `CREATE` no schema).
3. **Todo DDL/DML vem com:** (a) script de rollback, (b) aviso de impacto (locks, recompilação de planos, I/O, replicação/AG), (c) recomendação de janela.
4. **Primeiro entenda, depois mude.** Peça: versão/edição do engine, DDL da(s) tabela(s), volume de linhas, cardinalidade, plano real (não apenas estimado), parâmetros usados, frequência de execução e SLA.
5. **Não crie índice por reflexo.** Avalie custo de escrita (INSERT/UPDATE/DELETE), espaço, impacto em replicação/AG/streaming e sobreposição com índices existentes.

## Fluxo de trabalho

1. Coletar contexto (veja o item 4 acima).
2. Analisar o plano e classificar os achados (tabela abaixo).
3. Propor a correção de **menor risco primeiro** (reescrita de query / estatísticas → índice → hint/forced plan como último recurso).
4. Entregar: diagnóstico → mudança → rollback → impacto → validação (antes/depois).

---

## SQL Server

### Coleta do plano
- Plano **real** (`.sqlplan`) via SSMS/Azure Data Studio (Include Actual Execution Plan) ou Query Store.
- Habilitar métricas na sessão (não altera dados):
```sql
SET STATISTICS IO ON;
SET STATISTICS TIME ON;
```
- Plano em cache (permissão: `VIEW SERVER STATE`):
```sql
SELECT TOP (20)
       qs.total_elapsed_time / NULLIF(qs.execution_count,0) AS avg_elapsed_us,
       qs.execution_count,
       qs.total_logical_reads / NULLIF(qs.execution_count,0) AS avg_logical_reads,
       SUBSTRING(st.text, (qs.statement_start_offset/2)+1,
         ((CASE qs.statement_end_offset WHEN -1 THEN DATALENGTH(st.text)
           ELSE qs.statement_end_offset END - qs.statement_start_offset)/2)+1) AS statement_text,
       qp.query_plan
FROM sys.dm_exec_query_stats AS qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) AS st
CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle) AS qp
ORDER BY qs.total_elapsed_time DESC;
```

### O que procurar no plano

| Achado | Sintoma | Ação provável |
|---|---|---|
| **Key Lookup / RID Lookup** | Muitos lookups após Index Seek | Índice com `INCLUDE` das colunas faltantes (cuidado com largura) |
| **Table/Clustered Index Scan** | Scan em tabela grande com predicado seletivo | Índice na coluna do predicado; verificar funções em coluna (não-SARGable), conversão implícita |
| **Parameter Sniffing** | Mesma proc rápida/lenta conforme o parâmetro; `Estimated Rows` ≠ `Actual Rows` | `OPTION (RECOMPILE)` pontual, `OPTIMIZE FOR`, Query Store *forced plan*, ou separar em procs/ramos; revisar estatísticas |
| **Conversão implícita** | Warning `CONVERT_IMPLICIT` no plano (ex.: `varchar` vs `nvarchar`) | Corrigir tipo do parâmetro/coluna — evita scan e perda de seek |
| **Spools (Table/Index/Eager)** | Operador Spool com alto custo | Índice adequado, reescrita (CTE recursiva, subquery correlacionada) |
| **Sort/Hash Spill (tempdb)** | Warning de spill no Sort/Hash Match | Estatísticas atualizadas, memory grant, índice que entregue a ordenação |
| **Estimativa ruim** | Estimated vs Actual muito distintos | `UPDATE STATISTICS`, revisar cardinality estimator, estatísticas multi-coluna/filtradas |
| **Nested Loops em volume alto** | Loop com milhões de execuções internas | Índice na tabela interna, avaliar Hash/Merge via reescrita |

### Estatísticas (revisão)
```sql
-- Estatísticas desatualizadas da tabela alvo (permissão: VIEW DATABASE STATE / SELECT na tabela)
SELECT s.name AS stat_name,
       sp.last_updated, sp.rows, sp.rows_sampled, sp.modification_counter
FROM sys.stats AS s
CROSS APPLY sys.dm_db_stats_properties(s.object_id, s.stats_id) AS sp
WHERE s.object_id = OBJECT_ID(N'dbo.NomeTabela')
ORDER BY sp.modification_counter DESC;
```
```sql
-- Atualização controlada (evite FULLSCAN em tabelas gigantes no horário de pico)
UPDATE STATISTICS dbo.NomeTabela WITH SAMPLE 30 PERCENT;
-- ou, para a janela de manutenção:
-- UPDATE STATISTICS dbo.NomeTabela WITH FULLSCAN;
```
> **Impacto:** `UPDATE STATISTICS` invalida/recompila planos que dependem da estatística (pico de CPU por recompilação). Execute fora do pico e monitore.

### Índices ausentes (sugestão do engine — validar, não aplicar cegamente)
```sql
SELECT TOP (20)
       mid.statement AS tabela, mid.equality_columns, mid.inequality_columns, mid.included_columns,
       migs.user_seeks, migs.avg_total_user_cost, migs.avg_user_impact
FROM sys.dm_db_missing_index_details mid
JOIN sys.dm_db_missing_index_groups mig ON mig.index_handle = mid.index_handle
JOIN sys.dm_db_missing_index_group_stats migs ON migs.group_handle = mig.index_group_handle
ORDER BY migs.avg_total_user_cost * migs.avg_user_impact * (migs.user_seeks + migs.user_scans) DESC;
```
> Os DMVs de missing index não consideram sobreposição, custo de escrita nem largura. Sempre consolide com índices existentes.

### Criação do índice + rollback
```sql
-- ===== MUDANÇA =====
CREATE NONCLUSTERED INDEX IX_NomeTabela_ColA_ColB
    ON dbo.NomeTabela (ColA ASC, ColB ASC)
    INCLUDE (ColC, ColD)
    WITH (ONLINE = ON,            -- Enterprise/Azure SQL; em Standard, ONLINE=ON não é suportado
          FILLFACTOR = 90,        -- ajuste conforme perfil de escrita (tabelas append-only podem usar 100)
          SORT_IN_TEMPDB = ON,
          MAXDOP = 4,             -- limite de paralelismo para proteger a carga de produção
          DATA_COMPRESSION = PAGE -- avaliar CPU x I/O; remova se CPU já estiver saturada
         );
GO
-- ===== ROLLBACK =====
DROP INDEX IF EXISTS IX_NomeTabela_ColA_ColB ON dbo.NomeTabela;
GO
```
**Avisos de impacto (produção):**
- `ONLINE = ON` reduz bloqueios, mas ainda adquire `Sch-M` brevemente no final e `Sch-S`/`S` no início; uma transação longa aberta na tabela pode **bloquear o início/fim** da operação e gerar fila de bloqueios.
- Em **AlwaysOn AG**: a criação gera log volumoso → pode aumentar *log send queue* / *redo queue* na réplica secundária. Monitorar latência antes e durante.
- Crescimento de `tempdb` (`SORT_IN_TEMPDB`) e do log de transações — confirmar espaço livre.
- Planos que referenciam a tabela podem ser recompilados após a criação.
- Em edições sem `ONLINE`, a criação **bloqueia escrita** na tabela — exige janela de mudança.
- Se a opção `RESUMABLE = ON` estiver disponível (SQL 2017+ com ONLINE), considere-a para permitir pausa controlada.

### Parameter Sniffing — ordem de tratamento (menos invasivo → mais invasivo)
1. Atualizar estatísticas / corrigir estimativa.
2. Query Store: identificar regressão e usar *forced plan* (`sp_query_store_force_plan`) — reversível com `sp_query_store_unforce_plan`.
3. `OPTION (OPTIMIZE FOR (@p = valor))` ou `OPTION (RECOMPILE)` no statement problemático (custo: CPU de compilação por execução).
4. Refatorar (procs separadas por faixa de seletividade).
Evite `DBCC FREEPROCCACHE` em produção (limpa todo o cache → tempestade de recompilações). Prefira remover um único plano: `DBCC FREEPROCCACHE (plan_handle)`.

---

## PostgreSQL

### Coleta do plano
```sql
-- ATENÇÃO: EXPLAIN ANALYZE EXECUTA a query. Para DML, envolva em transação e faça ROLLBACK.
BEGIN;
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, SETTINGS, FORMAT TEXT)
SELECT ... ;
ROLLBACK;
```
> Em SELECT pesado em produção, prefira executar em **réplica de leitura** ou em homologação com volume equivalente. Para PG ≥ 13, `BUFFERS` também mostra uso de WAL com a opção `WAL`.

### O que procurar no plano

| Achado | Sintoma | Ação provável |
|---|---|---|
| **Seq Scan** em tabela grande | `Seq Scan` com `Rows Removed by Filter` alto | Índice na coluna filtrada; checar cast/função (usar índice funcional) |
| **Nested Loop ineficiente** | `loops=` muito alto no nó interno, estimativa ≪ real | `ANALYZE`, estatísticas estendidas (`CREATE STATISTICS`), índice na tabela interna, revisar join order |
| **Disk spill** | `Sort Method: external merge  Disk: …kB`, `Hash Batches: >1`, `Temp read/written` em BUFFERS | Aumentar `work_mem` **por sessão/role** (não global), índice que evite sort |
| **Estimativa ruim** | `rows=` estimado vs `actual rows` muito diferentes | `ANALYZE tabela`, aumentar `default_statistics_target` da coluna, estatísticas estendidas |
| **Bitmap Heap Scan com lossy** | `Heap Blocks: lossy=` | `work_mem` insuficiente; avaliar índice mais seletivo |
| **Muitos `shared read`** | Cache frio / working set grande | Revisar `shared_buffers`, índices cobrindo (`INCLUDE`), particionamento |
| **Bloat** | Tabela/índice inchado, scans lentos | Ver skill `db-troubleshooting-ha` (autovacuum/bloat) |

### Escolha do tipo de índice
- **B-tree:** igualdade/range/ordenação (padrão). Considere índices parciais (`WHERE ativo`) e de cobertura (`INCLUDE`, PG ≥ 11).
- **GIN:** `jsonb`, arrays, full-text (`tsvector`), `pg_trgm` para `LIKE '%x%'`. Escrita mais cara.
- **BRIN:** tabelas muito grandes com correlação física (ex.: `created_at` append-only). Índice minúsculo, custo de escrita baixo.
- **GiST/SP-GiST:** geoespacial, ranges, exclusão.

### Tuning de `work_mem` (sem alterar global)
```sql
-- Escopo mínimo: sessão ou transação
SET LOCAL work_mem = '64MB';   -- dentro de BEGIN ... COMMIT
-- Escopo de role (persistente, afeta só a aplicação específica):
-- ALTER ROLE app_relatorios SET work_mem = '64MB';
-- Rollback: ALTER ROLE app_relatorios RESET work_mem;
```
> **Impacto:** `work_mem` é por *nó de plano* e por *conexão*. `64MB × nós × conexões` pode causar OOM. Calcule o pior caso antes de aumentar. Nunca aumente global sem análise de `max_connections`.

### Criação do índice + rollback
```sql
-- ===== MUDANÇA ===== (CONCURRENTLY não pode rodar dentro de bloco de transação)
CREATE INDEX CONCURRENTLY IF NOT EXISTS ix_nometabela_cola_colb
    ON public.nometabela (cola, colb)
    INCLUDE (colc, cold)
    WITH (fillfactor = 90);

-- Validar que o índice ficou VÁLIDO (CONCURRENTLY pode deixar índice INVALID em caso de falha):
SELECT c.relname, i.indisvalid, i.indisready
FROM pg_index i JOIN pg_class c ON c.oid = i.indexrelid
WHERE c.relname = 'ix_nometabela_cola_colb';

-- ===== ROLLBACK =====
DROP INDEX CONCURRENTLY IF EXISTS public.ix_nometabela_cola_colb;
-- Se ficou INVALID: DROP INDEX CONCURRENTLY e recriar.
```
**Avisos de impacto (produção):**
- `CREATE INDEX CONCURRENTLY` não bloqueia escrita, mas faz **duas varreduras** da tabela, leva mais tempo e consome I/O/WAL; aguarda transações antigas terminarem (uma transação longa pode atrasá-lo indefinidamente).
- WAL gerado → impacto em **replicação streaming** (lag). Monitore `pg_stat_replication`.
- Defina `SET lock_timeout = '3s';` na sessão para não enfileirar bloqueios atrás de uma transação longa.
- `maintenance_work_mem` maior acelera a criação (ajuste por sessão).
- Rodar `ANALYZE public.nometabela;` após a criação, se necessário.

---

## Formato obrigatório da resposta

1. **Diagnóstico:** o que o plano mostra (com os números relevantes: reads, rows estimadas vs reais, tempo).
2. **Causa raiz provável** e nível de confiança.
3. **Proposta** (menor risco primeiro) — SQL Server e/ou PostgreSQL conforme a plataforma.
4. **Script de mudança** + **Script de rollback**.
5. **Impacto em produção:** locks, recompilação, I/O, log/WAL, AG/replicação, espaço.
6. **Validação:** métricas antes/depois (logical reads, duração, plano) e critério de sucesso.
7. **Lembrete:** testar em homologação com volume representativo e anexar evidências à GMUD (skill `db-gmud-compliance`).
