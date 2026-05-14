# lumied-saas

Plugin Claude Code para o ecossistema **Lumied / Construfare** — 26 skills + 1 agente, cobrindo compliance, deploy, UI/UX, scaffold e runbooks operacionais.

## O que está aqui

### Skills (26)

#### Compliance & isolamento de tenant (6)
- `tenant-audit` — detecta queries sem filtro `escola_id`/`tenant_id`
- `rls-check` — verifica RLS em migrations novas
- `edge-fn-authz` — valida JWT + role + escola_id na entrada
- `secrets-scan` — varre secrets hardcoded
- `pii-bucket-audit` — buckets Storage com PII privados + tenant-scoped
- `secret-rotation` — plano de rotação sem downtime

#### Deploy & operação (5)
- `lumied-deploy` — fluxo completo (git push + functions + migrations)
- `deploy-preflight` — orquestra checks antes do deploy
- `vercel-budget` — Vercel Free 100 deploys/dia
- `cron-health` — diagnóstico pg_cron
- `migration-rollback` — down migration antes de aplicar a up

#### UI / UX (4)
- `ui-lumied-style` — design system
- `mobile-audit` — uso em celular (família/aluno acessa do mobile)
- `a11y-quick` — acessibilidade mínima
- `loading-states` — feedback visual em async

#### Scaffold (3)
- `bank-adapter-template` — novo adapter de banco brasileiro
- `erp-importer-template` — novo importer de ERP escolar
- `edge-fn-new` — edge function Supabase com boilerplate Lumied

#### Runbooks & meta (5)
- `escola-onboarding` — provisionar tenant novo
- `bank-homologacao` — checklist por banco
- `erp-import-runbook` — migração assistida D-7 a D+7
- `chrome-extension-pack` — empacotar extensão no Windows
- `cost-audit` — caps Free Tier cruzados

#### Review & qualidade (3)
- `pr-review-lumied` — review combinado consolidado
- `test-coverage` — tiers de prioridade pra Onda 2 refator
- `postmortem` — template guiado de incidente

### Agents (1)

- `lumied-bug-triage` — triagem rápida de bugs reportados em escolas (stack trace → módulo + arquivo candidato + commit relevante + próximo passo).

## Contexto

Lumied é um SaaS multi-tenant para escolas (ERP escolar moderno). Construfare é um produto irmão (reconciliação financeira Sienge). Ambos:
- Operam em Supabase Free Tier + Vercel Free Tier + Cloudflare Workers Free Tier.
- Usam tenant isolation via `escola_id` + RLS — vazamentos cross-tenant são incidente crítico.
- Têm pipeline GTM com pilotos ativos (Maple Bear Caxias, Bento Gonçalves).

As skills aqui foram criadas a partir de incidentes reais, padrões repetidos e regras de operação consolidadas — não são genéricas. Linkam para memórias `project_*` / `feedback_*` no diretório de auto-memory do Claude Code para preservar contexto.

## Instalação

### Como marketplace local (desenvolvimento)

```bash
# No repositório do plugin (este diretório)
claude
> /plugin marketplace add /caminho/local/para/lumied-claude-plugin
> /plugin install lumied-saas
```

### Como marketplace GitHub (depois de publicar)

```bash
claude
> /plugin marketplace add ivyson/lumied-claude-plugin
> /plugin install lumied-saas@lumied-marketplace
```

### Manual (sem marketplace)

As skills/agente são arquivos comuns — pode copiar `skills/*` para `~/.claude/skills/` e `agents/*.md` para `~/.claude/agents/` em qualquer máquina.

## Versionamento

- `0.1.0` — release inicial com 26 skills + 1 agente.
- Roadmap: `reconcile-check`, `incident-runbook`, `escola-deactivation`, comandos slash de orquestração.

## Licença

Uso interno. Decisão de open-source futura pendente.
