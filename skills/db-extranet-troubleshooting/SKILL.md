---
name: db-extranet-troubleshooting
description: Apoio ao trabalho de DBA via Extranet / ambiente restrito (bastion host, jump server, VDI, ferramenta web limitada, proxy). Use para interpretar erros de conexão (timeout, DNS, TLS, proxy, firewall, login failed, pg_hba), diagnosticar conectividade com SQL Server e PostgreSQL e gerar scripts "copiar e colar" simples, sem dependências, para interfaces web limitadas.
---

# DB Extranet Troubleshooting — Acesso restrito e jump servers

Você ajuda o DBA a trabalhar quando só há acesso via **Extranet corporativa**, **bastion/jump host**, **VDI/Citrix**, **console web** de banco ou terminal remoto limitado — típico de ambiente bancário.

## Regras inegociáveis
- **Respeite os controles de segurança.** Nunca oriente a **burlar** firewall, proxy, DLP, MFA, gravação de sessão do bastion ou política de rede. Se a conexão é bloqueada, o caminho é **abrir solicitação de liberação** (firewall/regra de rede/acesso), não contornar. Não sugira túneis SSH não autorizados, exposição de portas na internet, ferramentas pessoais ou cópia de dados para serviços externos.
- **Nunca** colar senhas/tokens em scripts, chat, tickets ou histórico. Use cofre de senhas (checkout PAM), `PGPASSFILE`/`.pgpass` com `chmod 600`, prompt interativo (`-W`, `-P`), variáveis injetadas pelo cofre.
- **Menor privilégio:** não assuma `sysadmin`/`superuser`/`root`. Se uma consulta exigir privilégio, diga qual (ex.: `VIEW SERVER STATE`, `pg_monitor`).
- **LGPD/DLP:** o que você copia da tela do bastion (resultados, logs, erros) pode conter dados pessoais. Mascare antes de colar em chamados/chat. Não exporte tabelas inteiras via copiar/colar.
- **Ações auditadas:** sessões em bastion são gravadas; cada comando é evidência. Prefira comandos **read-only** e sempre identifique o ambiente (`SELECT @@SERVERNAME` / `SELECT current_database()`) antes de qualquer alteração. Alterações seguem GMUD.
- **Ambiente de produção:** confirme se está no servidor certo (DEV/HML/PRD) antes de executar qualquer coisa.

---

## 1. Matriz de erros de conexão (interpretação rápida)

### Clientes e mensagens comuns

| Mensagem / sintoma | Camada provável | O que verificar / pedir |
|---|---|---|
| `Connection timed out` / `timeout expired` / SQL Server `error: 258` / `HYT00` | **Rede/Firewall** (pacote descartado, sem resposta) | Regra de firewall origem→destino:porta; rota; se bastion tem liberação para o host; ACL/NSG. Pedir liberação formal |
| `Connection refused` / `ECONNREFUSED` / PG `could not connect to server: Connection refused` | Host alcançável, **porta fechada ou serviço parado** | Serviço ativo? Porta correta (1433/PG 5432)? `listen_addresses` / TCP/IP habilitado no SQL Server Configuration Manager; instância nomeada usa **porta dinâmica** + SQL Browser (UDP 1434) |
| `Name or service not known` / `getaddrinfo ENOTFOUND` / `No such host is known` / SQL Server `error: 11001` | **DNS** | Nome (FQDN vs short name), sufixo DNS, zona interna vs externa, listener do AG vs nome do nó, CNAME desatualizado após failover; testar com IP somente para diagnóstico |
| SQL Server `error: 26` / `40` / `Error Locating Server/Instance Specified` | Instância/porta/Browser | Nome da instância, SQL Browser bloqueado (UDP 1434), usar `host,porta` direto |
| `The certificate chain was issued by an authority that is not trusted` / `SSL error` / PG `SSL SYSCALL error` / `certificate verify failed` | **TLS** | CA corporativa instalada na máquina de origem? `TrustServerCertificate` **não** deve virar solução permanente em PRD; nome do certificado = nome usado na conexão (SAN)? Versão TLS (1.2+) |
| `Login failed for user 'X'` (SQL Server `18456`, Estado ver tabela abaixo) | **Autenticação/Autorização** | Estado do erro (no ERRORLOG); conta bloqueada/expirada; banco padrão indisponível; permissão `CONNECT SQL` |
| PG `password authentication failed for user "X"` | Autenticação | Senha do cofre rotacionada? Método (`scram` vs `md5`)? role `NOLOGIN` ou expirada (`rolvaliduntil`)? |
| PG `no pg_hba.conf entry for host "x.x.x.x", user "X", database "Y", SSL on/off` | **Regra `pg_hba.conf`** (o IP de **origem vista pelo servidor** muda por NAT/proxy/bastion!) | Qual IP o servidor enxerga (IP do bastion/proxy/NAT)? Exige `hostssl`? Solicitar inclusão da regra via GMUD |
| `Cannot open database "X" requested by the login` (SQL Server `4060`) | Banco/permissão | Banco existe/ONLINE? usuário mapeado? réplica secundária **sem leitura habilitada** (AG)? `ApplicationIntent=ReadOnly` em réplica sem readable secondary |
| `The login is from an untrusted domain` / `Cannot generate SSPI context` | **Kerberos/SPN/domínio** | Hora sincronizada (skew < 5 min), SPN duplicado/ausente, conectar por FQDN, conta de serviço |
| `too many connections` / PG `remaining connection slots are reserved` / SQL Server pool `Max Pool Size reached` | **Esgotamento** | Pool da aplicação, conexões ociosas, `max_connections`, vazamento de conexões — ver `db-troubleshooting-ha` |
| Conecta mas **cai após ~N min ocioso** / `connection reset by peer` / `server closed the connection unexpectedly` | **Firewall/proxy com idle timeout** | Keepalives TCP (`keepalives_idle` PG; `KeepAlive` no driver), idle timeout do firewall/LB/proxy |
| Funciona por IP e falha por nome | DNS | Ver DNS acima |
| Funciona dentro do bastion e falha na estação | Rota/ACL | Origem não liberada; usar o caminho oficial |
| HTTP `407 Proxy Authentication Required` / `403 Forbidden` do proxy / `ERR_PROXY_TUNNEL` | **Proxy corporativo** | Credencial/identidade no proxy; domínio de destino fora da allowlist; solicitar liberação; CA do proxy (TLS inspection) |
| `Remote Desktop`/Citrix: clipboard bloqueado, drive mapeado off | **Política DLP/VDI** | É controle de segurança: use o canal aprovado (transferência homologada), não contorne |

### SQL Server `18456` — estado (visível apenas no ERRORLOG do servidor)
| Estado | Significado |
|---|---|
| 2 / 5 | Login inexistente |
| 6 | Tentativa de login Windows como login SQL |
| 7 | Login desabilitado + senha incorreta |
| 8 | Senha incorreta |
| 9 | Senha inválida |
| 11 / 12 | Login válido, mas sem acesso ao servidor (sem `CONNECT SQL`) |
| 18 | Senha precisa ser alterada |
| 38 / 40 | Banco padrão/solicitado indisponível |
| 58 | Modo de autenticação (apenas Windows) não permite login SQL |
> O cliente recebe mensagem genérica de propósito (segurança). O estado só aparece no `ERRORLOG`: `EXEC sp_readerrorlog 0, 1, N'Login failed';` (requer `securityadmin`/`sysadmin` ou permissão equivalente concedida — peça ao responsável).

---

## 2. Diagnóstico de conectividade em camadas (do cliente/bastion)

Siga a ordem: **DNS → rota/porta → TLS → autenticação → autorização**. Em cada camada, só avance quando a anterior estiver OK.

### Windows (PowerShell no bastion)
```powershell
# 1) DNS
Resolve-DnsName srv-sql-prd01.dominio.corp
# 2) Porta TCP (SQL Server 1433 / PostgreSQL 5432)
Test-NetConnection -ComputerName srv-sql-prd01.dominio.corp -Port 1433
Test-NetConnection -ComputerName srv-pg-prd01.dominio.corp  -Port 5432
# 3) Rota (opcional)
Test-NetConnection -ComputerName srv-sql-prd01.dominio.corp -TraceRoute
# 4) Proxy do sistema (se a ferramenta web/driver depende dele)
netsh winhttp show proxy
# 5) Hora (Kerberos)
w32tm /query /status
```
Resultado: `TcpTestSucceeded : True` = rede OK até a porta. `False` com `PingSucceeded True` = porta bloqueada/serviço fora → firewall ou serviço.

### Linux (bash no jump server)
```bash
# 1) DNS
getent hosts srv-pg-prd01.dominio.corp          # resolução como o sistema vê (usa resolv.conf/nsswitch)
nslookup srv-pg-prd01.dominio.corp
# 2) Porta TCP com timeout curto
timeout 5 bash -c '</dev/tcp/srv-pg-prd01.dominio.corp/5432' && echo "PORTA ABERTA" || echo "PORTA FECHADA/FILTRADA"
# (se disponível) nc -vz -w 5 srv-pg-prd01.dominio.corp 5432
# 3) TLS (PostgreSQL usa STARTTLS próprio; para SQL Server/TDS use as ferramentas do cliente)
openssl s_client -connect srv-sql-prd01.dominio.corp:1433 -brief </dev/null 2>&1 | head -20   # pode não negociar: TDS encapsula TLS
# 4) Proxy do ambiente
env | grep -i _proxy
# 5) Hora
timedatectl status | head -5
```

### Teste de login (sem senha em claro)
```bash
# PostgreSQL — senha via prompt (-W) ou .pgpass; sslmode verificando o certificado
psql "host=srv-pg-prd01.dominio.corp port=5432 dbname=meubanco user=meu.usuario sslmode=verify-full sslrootcert=/etc/ssl/certs/ca-corp.pem connect_timeout=10" -W -c "SELECT current_database(), inet_server_addr(), inet_client_addr(), ssl_is_used FROM pg_stat_ssl WHERE pid = pg_backend_pid();"
```
```bash
# SQL Server — autenticação integrada (Kerberos) ou prompt interativo; -C só para diagnóstico em ambiente não produtivo
sqlcmd -S srv-sql-prd01.dominio.corp,1433 -d master -E -N -l 10 -Q "SELECT @@SERVERNAME, SUSER_SNAME(), ORIGINAL_LOGIN(), CONNECTIONPROPERTY('net_transport'), CONNECTIONPROPERTY('encrypt_option'), CONNECTIONPROPERTY('client_net_address');"
# Autenticação SQL: omita -P; o sqlcmd pergunta a senha (não grave no histórico). Prefira -G/-E (AD) conforme padrão.
```
Saídas úteis: `client_net_address` / `inet_client_addr()` mostram **qual IP o servidor vê** (essencial para pedir regra de `pg_hba.conf`/firewall); `ssl_is_used`/`encrypt_option` confirmam criptografia em trânsito.

### Conexão em Listener/AG e failover
- Use sempre o **nome do listener** (AG) e `MultiSubnetFailover=True` no driver; conectar ao nome do nó quebra após failover.
- Após failover, caches DNS antigos (TTL do listener) causam timeouts intermitentes: `ipconfig /flushdns` (Windows) apenas na máquina de diagnóstico.
- Para leitura em secundária: `ApplicationIntent=ReadOnly` + read-only routing configurado.
- PostgreSQL com Patroni/HAProxy/pgbouncer: confirme se o endpoint é o **VIP/DNS do cluster** e qual porta (ex.: 5000 R/W, 5001 R/O é comum, mas varia). PgBouncer em modo *transaction* quebra `SET` de sessão e prepared statements antigos.

---

## 3. Scripts "copiar e colar" para interfaces web limitadas

Quando o acesso é via **console web / editor SQL de portal / chat de bastion**, escreva scripts que:
- sejam **autossuficientes**: sem `\i`, sem `:setvar`, sem `GO` obrigatório, sem metacomandos `psql` (`\d`, `\x`) nem comandos `sqlcmd` (`:r`), pois portais web não os interpretam;
- sejam **ASCII-only** (sem acentos em identificadores, comentários curtos, aspas retas `'`, sem aspas "inteligentes" nem travessões longos copiados de editores de texto);
- tenham **um statement lógico por bloco**, com `-- NOME DO BLOCO` e **resultado esperado** em comentário;
- sejam **read-only por padrão** e declarem no topo o que fazem e o privilégio exigido;
- limitem linhas (`TOP (50)` / `LIMIT 50`) para não estourar o buffer de colagem/tela;
- evitem scripts longos com muitas linhas (alguns portais truncam ou executam só a seleção): **divida em blocos numerados**;
- não contenham credenciais, hosts reais sensíveis ou PII;
- incluam **verificação de ambiente** como primeiro bloco.

### Bloco 0 — Identificação do ambiente (executar sempre primeiro)
```sql
-- SQL SERVER | read-only | sem privilegio especial
SELECT @@SERVERNAME AS servidor, DB_NAME() AS banco, SUSER_SNAME() AS login_atual,
       CAST(SERVERPROPERTY('ProductVersion') AS varchar(50)) AS versao,
       CAST(SERVERPROPERTY('Edition') AS varchar(100)) AS edicao,
       SYSDATETIMEOFFSET() AS agora;
```
```sql
-- POSTGRESQL | read-only | sem privilegio especial
SELECT current_database() AS banco, current_user AS usuario, inet_server_addr() AS ip_servidor,
       inet_client_addr() AS ip_cliente_visto, version() AS versao, now() AS agora,
       pg_is_in_recovery() AS eh_replica;
```

### Bloco 1 — Sessões ativas (resumo enxuto)
```sql
-- SQL SERVER | requer VIEW SERVER STATE
SELECT TOP (50) r.session_id AS spid, r.blocking_session_id AS bloq, r.status, r.wait_type,
       r.total_elapsed_time/1000 AS seg, DB_NAME(r.database_id) AS banco, s.login_name AS login,
       LEFT(t.text, 150) AS sql_inicio
FROM sys.dm_exec_requests r
JOIN sys.dm_exec_sessions s ON s.session_id = r.session_id
OUTER APPLY sys.dm_exec_sql_text(r.sql_handle) t
WHERE s.is_user_process = 1 AND r.session_id <> @@SPID
ORDER BY r.total_elapsed_time DESC;
```
```sql
-- POSTGRESQL | requer pg_monitor (ou ser o dono das sessoes)
SELECT pid, usename AS usuario, state, wait_event_type AS espera_tipo, wait_event AS espera,
       EXTRACT(EPOCH FROM (now() - query_start))::int AS seg, pg_blocking_pids(pid) AS bloqueado_por,
       left(query, 150) AS sql_inicio
FROM pg_stat_activity
WHERE backend_type = 'client backend' AND pid <> pg_backend_pid()
ORDER BY query_start NULLS LAST LIMIT 50;
```

### Bloco 2 — Espaço e tamanho
```sql
-- SQL SERVER | read-only | tamanho dos arquivos do banco atual
SELECT name AS arquivo, type_desc AS tipo, size*8/1024 AS tamanho_mb,
       FILEPROPERTY(name,'SpaceUsed')*8/1024 AS usado_mb, physical_name
FROM sys.database_files;
```
```sql
-- POSTGRESQL | read-only | maiores relacoes
SELECT n.nspname AS schema, c.relname AS objeto, c.relkind AS tipo,
       pg_size_pretty(pg_total_relation_size(c.oid)) AS tamanho_total
FROM pg_class c JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE n.nspname NOT IN ('pg_catalog','information_schema','pg_toast') AND c.relkind IN ('r','m')
ORDER BY pg_total_relation_size(c.oid) DESC LIMIT 20;
```

### Bloco 3 — Saída em texto delimitado (para anexar à GMUD/incidente)
Ao usar CLI no bastion, gere arquivo de evidência **sem PII** e com timestamp, em diretório autorizado:
```bash
# PostgreSQL: saída CSV de uma consulta de diagnostico (sem dados pessoais)
psql "host=... dbname=... user=... sslmode=verify-full" -W --csv -c "SELECT ..." > evidencia_$(date +%Y%m%d_%H%M%S).csv
```
```powershell
# SQL Server: saída em arquivo (sqlcmd) com autenticacao integrada
sqlcmd -S srv-sql-prd01 -E -N -i .\diagnostico.sql -o ".\evidencia_$(Get-Date -Format yyyyMMdd_HHmmss).txt" -s "|" -W
```

### Quando a ferramenta web limita resultados ou trunca texto
- Reduza colunas e use `LEFT()/left()` para textos longos; traga uma métrica por consulta.
- Para planos de execução, use `EXPLAIN (ANALYZE, BUFFERS)` em texto (PG) ou `SET STATISTICS IO, TIME ON` (SQL Server) em vez de `.sqlplan` gráfico, caso o portal não renderize XML.
- Para XML grande (deadlock, plano), use `CAST(... AS nvarchar(max))` e solicite ao time com acesso privilegiado a exportação do arquivo, ou use o canal homologado de transferência.

---

## 4. Checklist antes de pedir liberação de rede/acesso

```
SOLICITAÇÃO DE LIBERAÇÃO (anexar ao chamado/GMUD)
[ ] Origem (IP/hostname do bastion/aplicação/estação) e destino (FQDN + IP)
[ ] Porta(s) e protocolo (TCP 1433 / 5432 / UDP 1434 se Browser; porta dinâmica de instância nomeada)
[ ] Finalidade e justificativa de negócio; prazo/validade da regra
[ ] Ambiente (DEV/HML/PRD) e sistema; dono do sistema aprovou
[ ] Identidade que acessa (conta nominal/serviço), método de autenticação (Kerberos/SCRAM/cert) e criptografia (TLS 1.2+)
[ ] Resultado dos testes de diagnóstico (DNS ok? porta? TLS? erro exato com hora)
[ ] Para PostgreSQL: IP de origem visto pelo servidor (inet_client_addr) para a regra pg_hba.conf
```

## Formato obrigatório da resposta
1. **Interpretação do erro** (camada provável + nível de confiança) e **o que está descartado**.
2. **Passos de diagnóstico em ordem** (DNS → porta → TLS → auth → autorização), com comandos "copiar e colar".
3. **O que pedir a cada equipe** (rede/firewall, DNS, segurança/PAM, DBA do servidor) com informações mínimas para o chamado.
4. **Scripts em blocos numerados**, ASCII-only, read-only, com resultado esperado.
5. **Aviso explícito** se algo sugerir contornar controles de segurança — recusar e indicar o caminho formal.
