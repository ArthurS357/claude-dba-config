# claude-dba-config — Skills de DBA Sênior (Santander)

Repositório central de **skills do Claude Code** para a rotina de um DBA Sênior em ambiente bancário
regulado: **SQL Server + PostgreSQL**, zero downtime, conformidade **LGPD/Bacen**, governança de mudanças
(**GMUD**) e operação via **Extranet** corporativa.

## Diretrizes embutidas em todas as skills

- **Tom sênior e conservador:** mitigação de risco e prevenção de incidentes em primeiro lugar.
- **Segurança:** nenhum script com senha em texto claro; nenhum pressuposto de `sysadmin`/`root`; menor privilégio.
- **Dualidade:** comandos equivalentes para SQL Server e PostgreSQL sempre que fizer sentido.
- **Código seguro:** todo DDL/DML vem com **script de rollback** e **aviso de impacto em produção** (locks, recompilação, log/WAL, AG/replicação).

## Skills disponíveis

| Skill | Objetivo |
|---|---|
| [`db-query-optimization`](skills/db-query-optimization/SKILL.md) | Otimizar queries lentas: planos de execução, índices (+ rollback), estatísticas, Parameter Sniffing, `work_mem`. |
| [`db-safe-migration`](skills/db-safe-migration/SKILL.md) | Revisar DDL antes da GMUD: locks, `ONLINE`/`CONCURRENTLY`, Expand and Contract, impacto em AG/streaming, checklist de rollback. |
| [`db-troubleshooting-ha`](skills/db-troubleshooting-ha/SKILL.md) | Diagnóstico de incidentes e HA: bloqueios, wait stats, deadlocks, AlwaysOn, bloat/autovacuum, replicação; contenção segura e coleta de evidências. |
| [`db-security-audit`](skills/db-security-audit/SKILL.md) | Auditoria de acessos: `GRANT`/`REVOKE`, logins/roles, TDE, `pg_hba.conf`, RLS, senhas expirando, excesso de privilégios. |
| [`db-gmud-compliance`](skills/db-gmud-compliance/SKILL.md) | Template de GMUD (risco, impacto, rollback), smoke tests pós-deploy e lembrete de evidências de homologação. |
| [`db-extranet-troubleshooting`](skills/db-extranet-troubleshooting/SKILL.md) | Trabalho via bastion/jump server/portal web: erros de conexão (timeout, DNS, TLS, proxy) e scripts "copiar e colar". |
| [`db-dba-general-help`](skills/db-dba-general-help/SKILL.md) | Dúvidas conceituais, scripts de manutenção (backup, reindex, statistics, vacuum) e estudo para certificações (DP-300 etc.). |

## Estrutura

```
claude-dba-config/
├── README.md
└── skills/
    ├── db-query-optimization/SKILL.md
    ├── db-safe-migration/SKILL.md
    ├── db-troubleshooting-ha/SKILL.md
    ├── db-security-audit/SKILL.md
    ├── db-gmud-compliance/SKILL.md
    ├── db-extranet-troubleshooting/SKILL.md
    └── db-dba-general-help/SKILL.md
```

Cada `SKILL.md` tem *frontmatter* YAML (`name` e `description`) seguido do corpo em Markdown. O Claude Code usa
o `description` para decidir quando carregar a skill automaticamente.

## Como copiar para a máquina local

O Claude Code carrega skills de `~/.claude/skills/` (pessoais, valem para todos os projetos) ou de
`<projeto>/.claude/skills/` (apenas naquele projeto). Cada skill é uma **pasta** contendo o `SKILL.md`.

### 1. Clonar o repositório

```bash
git clone https://github.com/arthurs357/claude-dba-config.git
cd claude-dba-config
```

### 2. Instalar as skills (escolha uma opção)

**Linux / macOS — cópia (recomendado em ambiente corporativo):**

```bash
mkdir -p ~/.claude/skills
cp -r skills/* ~/.claude/skills/
```

**Linux / macOS — symlink (atualiza com `git pull`):**

```bash
mkdir -p ~/.claude/skills
for d in "$PWD"/skills/*/; do ln -sfn "$d" ~/.claude/skills/"$(basename "$d")"; done
```

**Windows (PowerShell):**

```powershell
New-Item -ItemType Directory -Force "$env:USERPROFILE\.claude\skills" | Out-Null
Copy-Item -Recurse -Force .\skills\* "$env:USERPROFILE\.claude\skills\"
```

**Somente para um projeto específico:**

```bash
mkdir -p <caminho-do-projeto>/.claude/skills
cp -r skills/* <caminho-do-projeto>/.claude/skills/
```

### 3. Verificar

Abra uma nova sessão do Claude Code e confirme que as skills aparecem (por exemplo, pergunte
"quais skills `db-*` você tem disponíveis?" ou invoque `/db-query-optimization`).

### Atualizar

```bash
cd claude-dba-config && git pull
cp -r skills/* ~/.claude/skills/   # não necessário se usou symlink
```

## Uso responsável (importante)

- As skills são **apoio à decisão**: todo script gerado deve passar por **peer review** e **teste em homologação**
  antes de ir para produção, com **evidências anexadas à GMUD**.
- **Nunca cole** no Claude senhas, tokens, strings de conexão com credenciais, nem dados pessoais reais
  (CPF, nomes, contas). Mascare saídas de consultas e logs antes de compartilhar (LGPD).
- Respeite as políticas de segurança da instituição (DLP, proxy, bastion): as skills **não** orientam a
  contornar controles; quando houver bloqueio, o caminho é a solicitação formal de liberação.
- Confirme se o uso de ferramentas de IA com informações de ambiente está autorizado pela política interna
  antes de compartilhar nomes de servidores, esquemas ou planos de execução reais.
