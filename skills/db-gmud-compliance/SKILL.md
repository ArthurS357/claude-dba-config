---
name: db-gmud-compliance
description: Governança de mudanças (GMUD) para DBA em ambiente bancário. Use para redigir a descrição da mudança (risco, impacto, plano de execução, plano de rollback), montar o plano de comunicação, gerar queries de smoke test pós-deploy em SQL Server e PostgreSQL, e lembrar de anexar evidências de homologação antes de abrir/aprovar a GMUD.
---

# DB GMUD Compliance — Governança de Mudanças

Você apoia o DBA Sênior na documentação e no controle de mudanças (GMUD/RDM) exigidos em ambiente regulado (Bacen/CMN, auditoria interna e externa, LGPD). Uma mudança sem evidência e sem rollback **não está pronta** para ser aberta.

## Regras inegociáveis
- **Nunca** coloque senhas, strings de conexão com credenciais, tokens ou dados pessoais reais (CPF, nome, conta) na GMUD, em evidências ou em exemplos. Mascare/anonimize resultados antes de anexar.
- **Nunca** assuma `sysadmin`/`superuser`: informe na GMUD a conta/role **nominal** que executará a mudança e seu privilégio mínimo.
- Não existe "mudança sem rollback". Se o rollback for impossível (ex.: `DROP COLUMN` irreversível), a GMUD deve declarar isso explicitamente, exigir **backup restaurado e validado** e aprovação de risco reforçada.
- **Evidências de homologação são obrigatórias** (veja a seção de lembretes). Se o usuário pedir para gerar a GMUD sem elas, gere o rascunho **e** liste de forma destacada o que falta anexar.
- Execução em produção: ao menos 2 pessoas (executor + revisor/observador), conforme política interna.

## Quando o usuário pedir ajuda com GMUD, faça

1. Identifique o **tipo**: Normal / Emergencial / Padrão (pré-aprovada). Emergenciais exigem justificativa do incidente e regularização posterior (evidências completas depois, aprovação a posteriori).
2. Se faltarem dados essenciais, **pergunte antes de preencher**: ambientes, banco/instância, janela, sistemas impactados, volume/tamanho dos objetos, edição do SQL Server / versão do PostgreSQL, existência de AG/réplicas/DR.
3. Preencha o **template** abaixo (sem inventar números: use `<A PREENCHER>` onde faltar dado).
4. Gere os **smoke tests** da plataforma.
5. Emita o **checklist de evidências** e aponte pendências.

---

## Template — Descrição da mudança

```
GMUD: <número>                         TIPO: [ ] Normal  [ ] Emergencial  [ ] Padrão
TÍTULO: <verbo no infinitivo + objeto + sistema> (ex.: "Criar índice IX_xxx na tabela dbo.Pagamento — Sistema Y")
SOLICITANTE / EXECUTOR (nominal) / REVISOR: <nomes> | CONTATO PLANTÃO: <telefone/canal>

1. JUSTIFICATIVA DE NEGÓCIO / TÉCNICA
   <problema ou necessidade; link para incidente/problema/projeto; benefício esperado (ex.: reduzir p95 de 8s para <1s)>

2. ESCOPO
   Ambiente: [ ] DEV [ ] HML [ ] PRD   Plataforma: [ ] SQL Server <versão/edição> [ ] PostgreSQL <versão>
   Instância/Cluster/AG: <nome>   Banco(s): <nome>   Objetos afetados: <lista>
   Sistemas/aplicações dependentes: <lista>   Réplicas/DR afetados: <lista>

3. DESCRIÇÃO DA MUDANÇA (PLANO DE EXECUÇÃO)
   Passo | Responsável | Comando/Script (anexo) | Duração estimada | Ponto de verificação
   1. Pré-checks (espaço, backup, saúde AG/replicação, ausência de bloqueios) ...
   2. ...
   N. Smoke tests pós-mudança ...
   Janela: <data/hora início – fim>  Fuso: <BRT/UTC>   Duração total estimada: <>  Ponto sem retorno (go/no-go): <hora>

4. ANÁLISE DE RISCO
   Probabilidade: [ ] Baixa [ ] Média [ ] Alta    Severidade: [ ] Baixa [ ] Média [ ] Alta [ ] Crítica
   Riscos identificados e mitigação:
   - Locks / bloqueios: <objeto, tipo de lock, duração esperada, mitigação (ONLINE/CONCURRENTLY, lock_timeout)>
   - Recompilação de planos / aquecimento de cache: <>
   - Log de transações / WAL / lag AG ou streaming: <>
   - Espaço em disco / tempdb: <>
   - Dependência de aplicação (versão compatível antes/depois): <>
   - Dados pessoais (LGPD): <há PII envolvida? mascaramento? retenção? acesso restrito?>

5. IMPACTO
   Indisponibilidade esperada: [ ] Nenhuma (zero downtime) [ ] Parcial <detalhar> [ ] Total <tempo>
   Impacto em performance durante a janela: <>   Usuários/serviços impactados: <>
   Impacto em integrações / ETL / relatórios / DR / backups: <>

6. PLANO DE ROLLBACK
   Gatilhos de acionamento: <ex.: erro X; latência p95 > Y por Z min; bloqueio > N min; lag AG > M MB>
   Passos de rollback (script anexo, testado em HML): 1... 2... 3...
   Tempo estimado de rollback: <>   Responsável pela decisão: <nome/cargo>
   Backup/ponto de restauração: <tipo, hora, validação de restore (data)>
   Reversibilidade: [ ] Total  [ ] Parcial <explicar>  [ ] Irreversível <justificar + aprovação reforçada>
   Smoke tests pós-rollback: <ref. seção abaixo>

7. TESTES E EVIDÊNCIAS (ANEXOS)
   [ ] Execução em HML com volume representativo (tempos, planos, locks) — data: ____
   [ ] Rollback testado em HML — data: ____
   [ ] Peer review do script por 2º DBA — nome: ____
   [ ] Evidências "antes/depois" (métricas, planos de execução)
   [ ] Aprovação do dono do dado/sistema (LGPD, quando aplicável)

8. COMUNICAÇÃO
   Antes (D-x): <quem avisar, canal>   Início da janela: <>   Fim/sucesso: <>   Rollback/incidente: <>
   Stakeholders: <aplicação, negócio, NOC/monitoração, segurança, infraestrutura, DR>

9. CRITÉRIOS DE SUCESSO
   <smoke tests OK; sem erro nos logs; latência dentro do SLA; AG/replicação saudáveis por 30–60 min>

10. PÓS-MUDANÇA
   Monitoração reforçada até: <data/hora>   Fechamento da GMUD: <responsável>   Lições aprendidas: <>
```

### Dicas de redação (auditoria gosta de objetividade)
- Verbos no infinitivo, passos numerados, cada um com **responsável** e **critério de verificação**.
- Declare o **estado final esperado** e como será provado (a query e o resultado esperado).
- Fuso horário explícito; duração com margem (planejado + 30–50 %).
- Nunca "executar script anexo" sem versão/hash/nome do arquivo e caminho no repositório aprovado.

---

## Lembretes obrigatórios ao DBA (emita sempre ao finalizar a GMUD)

> **Antes de abrir/submeter a GMUD, confirme que anexou:**
> 1. 📎 **Evidências de testes em homologação**: saída do script, tempos de execução, planos de execução antes/depois, métricas de bloqueio.
> 2. 📎 **Teste do rollback em homologação** (não apenas o script escrito).
> 3. 📎 **Prova de backup/restore válido** (data do último restore test) e status de AG/replicação saudável.
> 4. 📎 **Peer review** do script (nome do revisor + data).
> 5. 📎 Evidências **sem dados pessoais** (mascarar CPF, nomes, contas) — LGPD.
> 6. 📎 Aprovações exigidas (gestor, dono do sistema, segurança/DPO quando houver dado pessoal).
> 7. 📎 Hash/versão do script no repositório de código (rastreabilidade).
> 8. Homologação com **volume e configuração equivalentes** à produção? Se não, declare a diferença como risco residual.

---

## Smoke tests pós-deploy (validar que o banco está saudável)

Execute **antes** (baseline) e **depois** da mudança; anexe as saídas à GMUD (mascaradas). Todas são read-only.

### SQL Server

**1. Conectividade, versão e estado dos bancos**
```sql
SELECT @@SERVERNAME AS servidor, SERVERPROPERTY('ProductVersion') AS versao, SYSDATETIMEOFFSET() AS agora;
SELECT name, state_desc, user_access_desc, recovery_model_desc, is_read_only, log_reuse_wait_desc
FROM sys.databases WHERE name = DB_NAME() OR database_id > 4;   -- esperado: ONLINE, MULTI_USER, log_reuse_wait coerente
```
**2. Objetos alterados existem e estão válidos**
```sql
SELECT o.name, o.type_desc, o.modify_date FROM sys.objects o WHERE o.modify_date >= DATEADD(HOUR,-4,GETDATE()) ORDER BY o.modify_date DESC;
SELECT i.name AS indice, i.type_desc, i.is_disabled, i.is_hypothetical
FROM sys.indexes i WHERE i.object_id = OBJECT_ID(N'dbo.NomeTabela');           -- esperado: is_disabled = 0
-- Objetos com dependências quebradas
SELECT referencing_entity_name = OBJECT_NAME(referencing_id), referenced_entity_name
FROM sys.sql_expression_dependencies WHERE referenced_id IS NULL AND is_ambiguous = 0 AND referenced_database_name IS NULL;
```
**3. Constraints "trusted" e integridade lógica**
```sql
SELECT name, is_disabled, is_not_trusted FROM sys.foreign_keys WHERE is_not_trusted = 1 OR is_disabled = 1;   -- esperado: vazio
SELECT name, is_disabled, is_not_trusted FROM sys.check_constraints WHERE is_not_trusted = 1 OR is_disabled = 1;
```
**4. Bloqueios, esperas e sessões problemáticas**
```sql
SELECT COUNT(*) AS sessoes_bloqueadas FROM sys.dm_exec_requests WHERE blocking_session_id <> 0;   -- esperado: 0 (ou estável e baixo)
SELECT TOP (10) session_id, wait_type, wait_time, blocking_session_id, command FROM sys.dm_exec_requests WHERE session_id > 50 ORDER BY wait_time DESC;
```
**5. Alta disponibilidade (se AG)**
```sql
SELECT ar.replica_server_name, DB_NAME(drs.database_id) AS db, drs.synchronization_state_desc, drs.synchronization_health_desc,
       drs.is_suspended, drs.log_send_queue_size, drs.redo_queue_size
FROM sys.dm_hadr_database_replica_states drs
JOIN sys.availability_replicas ar ON ar.replica_id = drs.replica_id;   -- esperado: SYNCHRONIZED/HEALTHY, is_suspended=0, filas ~0
```
**6. Jobs e erros recentes**
```sql
-- Jobs falhos nas últimas 4h (permissão em msdb: SQLAgentReaderRole)
SELECT j.name, h.run_date, h.run_time, h.message
FROM msdb.dbo.sysjobhistory h JOIN msdb.dbo.sysjobs j ON j.job_id = h.job_id
WHERE h.run_status = 0 AND h.step_id = 0
  AND msdb.dbo.agent_datetime(h.run_date, h.run_time) >= DATEADD(HOUR,-4,GETDATE());
EXEC sp_readerrorlog 0, 1, N'Error';   -- erros no ERRORLOG
```
**7. Teste funcional mínimo (ajuste ao sistema)**
```sql
-- Exemplo: leitura de verificação nos objetos impactados, SEM expor PII
SELECT COUNT(*) AS linhas FROM dbo.NomeTabela;                      -- comparar com baseline
SELECT TOP (1) 1 AS ok FROM dbo.NomeTabela WITH (NOLOCK);          -- NOLOCK somente em smoke test; nunca para dados críticos de negócio
```
Compare também: tempo de queries-chave (antes/depois), contagem de linhas em tabelas afetadas, e CPU/duração dos 5 principais statements (`sys.dm_exec_query_stats`).

### PostgreSQL

**1. Conectividade e versão**
```sql
SELECT version(), current_database(), now() AS agora, pg_is_in_recovery() AS em_recovery;   -- primária: false
```
**2. Objetos válidos (índices e constraints)**
```sql
-- Índices INVÁLIDOS (resíduo típico de CREATE INDEX CONCURRENTLY que falhou) — esperado: vazio
SELECT n.nspname, c.relname AS indice, i.indisvalid, i.indisready
FROM pg_index i JOIN pg_class c ON c.oid = i.indexrelid JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE NOT i.indisvalid OR NOT i.indisready;

-- Constraints não validadas (NOT VALID) pendentes — esperado: apenas as planejadas
SELECT conrelid::regclass AS tabela, conname, contype FROM pg_constraint WHERE NOT convalidated;
```
**3. Bloqueios e sessões**
```sql
SELECT count(*) FILTER (WHERE wait_event_type = 'Lock') AS aguardando_lock,
       count(*) FILTER (WHERE state = 'idle in transaction' AND now() - xact_start > interval '1 minute') AS idle_in_tx_longas,
       count(*) AS total
FROM pg_stat_activity WHERE backend_type = 'client backend';       -- esperado: 0 / 0 / abaixo do max_connections
```
**4. Replicação e WAL**
```sql
SELECT application_name, state, sync_state, replay_lag,
       pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS replay_lag_bytes
FROM pg_stat_replication;            -- esperado: streaming, lag baixo/estável
SELECT slot_name, active FROM pg_replication_slots WHERE NOT active;   -- esperado: vazio (ou conhecido)
```
**5. Saúde de dados e vacuum**
```sql
SELECT relname, n_dead_tup, last_autovacuum FROM pg_stat_user_tables ORDER BY n_dead_tup DESC LIMIT 5;
SELECT datname, age(datfrozenxid) FROM pg_database ORDER BY 2 DESC LIMIT 3;
SELECT datname, xact_commit, xact_rollback, deadlocks, conflicts FROM pg_stat_database WHERE datname = current_database();
```
**6. Teste funcional mínimo**
```sql
SELECT count(*) FROM public.nometabela;                              -- comparar com baseline
EXPLAIN (COSTS OFF) SELECT ... ;                                      -- confirmar uso do novo índice (sem executar a query pesada)
```
Verifique também o **log do PostgreSQL** (`ERROR`, `FATAL`, `deadlock detected`) na janela da mudança.

---

## Critério de "go/no-go" pós-mudança (sugestão)
| Verificação | Resultado esperado | Ação se falhar |
|---|---|---|
| Bancos online / réplicas saudáveis | 100 % | Acionar rollback / escalar |
| Bloqueios / esperas por lock | 0 (ou baixo e estável) | Investigar (`db-troubleshooting-ha`) |
| Latência p95 de queries-chave | ≤ baseline + tolerância acordada | Rollback se persistir |
| Erros no log (ERRORLOG/PG log) | Sem erros novos | Analisar; rollback se crítico |
| Lag AG/streaming | Voltando ao baseline | Pausar backfills; escalar |

## Formato obrigatório da resposta
1. GMUD preenchida (template) com `<A PREENCHER>` onde faltar dado.
2. Smoke tests da plataforma-alvo (e rollback) com o **resultado esperado** de cada um.
3. **Lista de pendências/evidências** a anexar (bloco de lembretes acima).
4. Alertas de risco que possam levar à reprovação do CAB.
