---
name: deploy-preflight
description: Rotina pré-deploy que orquestra `secrets-scan` + `tenant-audit` + `rls-check` + `edge-fn-authz` + `cron-health` + `vercel-budget` antes de chamar `lumied-deploy`. Use sempre que houver mudança não-trivial pra subir, ou quando o usuário disser "vamos subir" sem ter rodado checks.
---

# Deploy preflight — Lumied / Construfare

Contexto: o ritmo de deploy do Lumied é alto ([[feedback_always_deploy]]) e o repo é público hoje ([[feedback_repo_public_secrets]]). Cada deploy é uma chance de regredir em isolation, vazar secret ou estourar o cap Vercel. Esta skill bloqueia o `lumied-deploy` até passar pelos checks.

## Ordem de execução

Sequencial, com gate em cada passo. Falhou crítico → parar.

### 1. Status do git (info, sem gate)

```bash
git status --short
git log --oneline -5
git diff --stat HEAD origin/main 2>/dev/null
```

Reportar:
- Qtd de arquivos modificados/criados.
- Quantos commits ahead/behind.
- Branch atual.

### 2. `secrets-scan` (gate 🔴)

Roda [[secrets-scan]]. Se achar VAZA, **parar tudo** e exigir rotação antes de prosseguir.

### 3. `tenant-audit` (gate 🔴)

Roda [[tenant-audit]] no delta. Se achar SUSPEITO em edge function nova → parar.

### 4. `rls-check` (gate 🔴)

Pra cada `supabase/migrations/*.sql` nova/modificada, roda [[rls-check]]. Tabela sem RLS = parar.

### 5. `edge-fn-authz` (gate 🔴)

Pra cada edge function nova/modificada, roda [[edge-fn-authz]]. JWT/role check ausente = parar.

### 6. `pii-bucket-audit` (gate 🟡, só se mig criou bucket)

Se a migração faz `INSERT INTO storage.buckets` ou cria bucket via SQL/API, rodar [[pii-bucket-audit]].

### 7. `cron-health` (gate 🟡)

Roda [[cron-health]]. Se houver cron broken já, **avisar** mas não bloquear (deploy pode estar justamente fixando).

### 8. `vercel-budget` (gate 🟡)

Roda [[vercel-budget]]. Se ≥90/100 deploys hoje, perguntar ao usuário se vale prosseguir ou esperar amanhã. [[feedback_agrupar_commits]] é regra justamente por isso.

### 9. Resumo final

```
🟢 PRONTO PRA DEPLOY
  Mudanças: 12 arquivos, 3 commits, 1 mig nova
  ✅ secrets-scan        nenhum hardcoded
  ✅ tenant-audit        2 edges revisadas
  ✅ rls-check           mig 332 OK
  ✅ edge-fn-authz       JWT+role validados
  ✅ pii-bucket-audit    nenhum bucket novo
  🟡 cron-health         backup-escolas ainda quebrado (não relacionado)
  ✅ vercel-budget       47/100 deploys hoje
  → próximo passo: invocar `lumied-deploy`
```

OU:

```
🔴 DEPLOY BLOQUEADO
  ✅ secrets-scan
  🔴 tenant-audit        edge `boletins_publicar` sem .eq('escola_id')
  ⏸️  passos seguintes não executados
  → fix necessário antes de prosseguir
```

## Procedimento de implementação

Esta skill é uma orquestradora — ela invoca outras skills via Skill tool em sequência, agrega resultados, e gera o resumo. Quando o usuário pedir "deploy" sem rodar preflight, sugerir:

> "Antes de invocar `lumied-deploy`, melhor rodar `deploy-preflight` pra não regredir em isolation/secrets/budget. Quer que eu rode?"

Se o usuário disser "pula", documentar em comentário no commit (ex: "skip preflight — emergência").

## Variantes

### `--quick` (alterações pequenas)

Pular `cron-health` + `vercel-budget`. Manter gates críticos (secrets/tenant/rls/authz).

### `--full` (release semanal/grande)

Adicionar:
- Lint geral (`eslint`/`tsc --noEmit` se houver TS).
- Testes (se [[project_refator_lumied]] Onda 2 já tiver testes).
- `mobile-audit` em arquivos HTML modificados.
- `a11y-quick` em arquivos HTML modificados.

### `--construfare`

Mesma rotina, mas com [[reference_construfare_supabase]] em vez de [[reference_lumied_supabase]]. `cron-health` foca em `sienge_reconcile_run`, `outbound_pulse`, `backup_construfare`.

## Quando NÃO rodar

- Hotfix urgente em prod onde a janela de risco é menor que o tempo da audit (raro — quase sempre vale 30s de check).
- Deploy apenas de docs (README/CHANGELOG sem código).
- Push de mudança em GH Actions YAML que não tem impacto runtime.
