---
name: pr-review-lumied
description: Review combinado de PR Lumied/Construfare — orquestra tenant-audit + rls-check + edge-fn-authz + mobile-audit + secrets-scan + ui-lumied-style + loading-states numa só passada, e produz um relatório consolidado com bloqueios vs nits. Use quando o usuário pedir "review esse PR", "vê se tá pronto pra subir", "auditoria completa antes do merge".
---

# PR review consolidado — Lumied / Construfare

Contexto: temos várias skills de check focadas (tenant-audit, rls-check, edge-fn-authz, secrets-scan, mobile-audit, etc.). Chamar uma por uma é repetitivo. Esta skill orquestra a passagem completa e separa **bloqueios** (não pode subir) de **nits** (subir e abrir issue depois).

## Quando rodar

- PR aberto pronto pra merge.
- Antes de chamar [[deploy-preflight]] direto (que é o passo final, deploy iminente).
- Branch grande/longa que acumulou mudanças em várias camadas.

NÃO usar pra:
- Hotfix de 3 linhas óbvio — review manual rápido + push direto.
- WIP / draft PR — esperar o autor sinalizar pronto.

## Procedimento

### 1. Inventário do diff

```bash
git diff origin/main...HEAD --stat
git diff origin/main...HEAD --name-only
```

Classifica os arquivos modificados:
- `supabase/migrations/*.sql` → rodar [[rls-check]] + [[migration-rollback]]
- `supabase/functions/**` → rodar [[tenant-audit]] + [[edge-fn-authz]]
- `*.html` / `*.jsx` / `*.tsx` (UI) → rodar [[ui-lumied-style]] + [[mobile-audit]] + [[loading-states]] + [[a11y-quick]]
- Qualquer arquivo de config / env / hardcoded → rodar [[secrets-scan]]
- `pg_cron` / `supabase/migrations/*cron*.sql` → cross-check com [[cron-health]] e [[deploy-preflight]]
- Adapter de banco em `bank-*` → cross-check com [[bank-homologacao]]

### 2. Rodar em paralelo

Cada skill aplicável → reporta achados.

### 3. Consolidar em formato fixo

```markdown
## Review consolidado — branch `<branch>` vs `origin/main`

### 🔴 BLOQUEIOS (não pode subir)
- [tenant-audit] supabase/functions/foo/index.ts:42 — query sem .eq('escola_id', ...)
- [rls-check] mig 330 — tabela `xpto` sem RLS habilitada
- [secrets-scan] .env.example contém ML_REFRESH_TOKEN literal

### 🟠 NITS (subir + abrir issue)
- [mobile-audit] gerente.html — botão "Adicionar" tem 36px (mín 44)
- [a11y-quick] alunos.html — img sem alt em 3 lugares
- [ui-lumied-style] inline style #ff0000 deveria usar var(--lumied-danger)

### 🟢 OK
- [edge-fn-authz] todas as edge functions modificadas validam JWT + role + escola_id
- [loading-states] novas chamadas async têm estado loading/error
- [migration-rollback] mig 330 tem rollback documentado

### Recomendação
- Bloqueios precisam ser corrigidos antes do merge. Pos fix, rodar `deploy-preflight`.
- Nits: criar issues separadas com label `tech-debt`. Não bloqueiam.
```

### 4. Se zero bloqueios

Sugere próximo passo: `deploy-preflight` → `lumied-deploy`.

### 5. Se há bloqueios

**Não corrija sozinho** — apresenta a lista pro usuário decidir:
- Corrigir agora (mostrar diffs propostos).
- Devolver pro autor (escrever comentário sumarizando).

## Princípios

1. **Severidade real importa.** Style nit ≠ vazamento de dados. Não inverter a ordem.
2. **Não corrigir silenciosamente.** Reportar, aguardar decisão.
3. **Linkar contexto.** Cada bloqueio cita arquivo:linha + qual skill detectou + memória relevante ([[project_tenant_isolation_incident]] etc.).
4. **Diff focado.** Não comentar em código não modificado pelo PR (escopo).
5. **Sem culpa.** "linha X tem padrão Y inseguro" — não "você esqueceu".

## Anti-padrões

- Rodar review mas não consolidar — usuário tem que ir nas 7 saídas separadas.
- Misturar bloqueio com nit no mesmo bullet.
- Pular alguma skill aplicável "porque parece OK olhando".
- Recomendar deploy mesmo com bloqueios — sempre devolver pra fix primeiro.

## Quando o PR é multi-tenant-impacting

Se o diff toca em qualquer tabela com `escola_id`, ou qualquer edge function que lê dados de aluno/financeiro/RH/manut, **escalar a régua**:
- Tenant-audit obrigatório com leitura linha-a-linha das queries.
- RLS-check em modo paranoico.
- Edge-fn-authz cobrindo até paths que parecem read-only.
- Considerar pedir um deploy em staging primeiro (não atalho direto pra prod).

Memória de referência do tipo de incidente que essa régua previne: [[project_tenant_isolation_incident]].
