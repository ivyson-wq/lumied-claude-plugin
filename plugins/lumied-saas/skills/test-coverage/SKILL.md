---
name: test-coverage
description: Verifica cobertura de testes do Lumied — quais módulos têm zero testes, quais áreas críticas (tenant isolation, RLS, edge functions de pagamento) estão sem garantia automatizada. Destrava Onda 2 do [[project_refator_lumied]] (testes), que é pré-requisito pras ondas 3 e 4. Use ao iniciar Onda 2, antes de refator grande, ou quando o usuário disser "tá testado?", "cobertura de X".
---

# Test coverage — Lumied

Contexto: [[project_refator_lumied]] tem 6 ondas. Onda 1 (higiene, utils) deployada; Onda 2 = testes — sem ela, ondas 3 (refatorar) e 4 (separar) ficam arriscadas porque não há rede de segurança. Hoje a cobertura é praticamente zero (validado durante Onda 1). Esta skill identifica o que tem que ser testado **primeiro** (alta criticidade) vs depois (baixa criticidade), pra não cair na armadilha de "testar tudo, parar tudo".

## Quando rodar

- Iniciando Onda 2 do refator ([[project_refator_lumied]]).
- Antes de refator de área específica (financeiro, acadêmico, almoxarifado).
- Quando o usuário disser "tem teste pra X?", "se eu mexer aqui quebra?".
- Pós-incidente, perguntar "teste teria pego?" — alimenta [[postmortem]].

## Procedimento

### 1. Mapear o estado atual

```bash
# Existem arquivos de teste?
find . -name "*.test.ts" -o -name "*.test.tsx" -o -name "*.spec.ts" -o -name "*_test.go" 2>/dev/null

# Há framework configurado? (package.json, vitest.config, jest.config)
grep -l "vitest\|jest\|playwright" package.json supabase/functions/*/package.json 2>/dev/null

# CI roda testes?
grep -l "npm test\|vitest\|jest" .github/workflows/*.yml 2>/dev/null
```

Output esperado da Onda 1: `0 arquivos de teste`, `framework não configurado`. Atualizar essa skill quando mudar.

### 2. Lista de áreas críticas (testar primeiro)

Ordenado por **impacto se quebrar** × **frequência de mudança**:

#### Tier 🔴 (testar antes de qualquer refator)
- **Tenant isolation:** edge function lê dados? Filtra `escola_id`? ([[tenant-audit]] manual hoje — virar teste).
- **RLS policies:** mig adiciona tabela tenant-scoped? Policy bloqueia cross-tenant? ([[rls-check]] hoje).
- **Edge functions de pagamento:** emissão boleto/PIX, webhook baixa, conciliação ([[project_banks_multiprovider]], [[project_sienge_reconcile]]).
- **Auth & roles:** quem é superadmin, quem é professor, quem é família — checks de role.
- **Backups:** [[project_backups]] — restore funciona? (Backup testado regularmente > backup feito.)

#### Tier 🟠 (testar na sequência)
- Almoxarifado movimentação ([[project_alm_orfaos]] — auto-create de itens, autosave [[project_autosave_patterns]]).
- Reconcile Construfare ([[project_construfare_reconcile_coverage]]).
- ERP importers ([[project_migracao_erps]] — 7 sprints, cada um precisa de teste de importação).
- Cron jobs ([[cron-health]] cobre health, mas teste unitário do que o cron faz seria útil).

#### Tier 🟡 (testar depois)
- UI components — Playwright fim-a-fim de fluxos críticos (login, lançar nota, registrar movimento almox).
- Utils consolidados na Onda 1.
- Edge functions de notificação (e-mail, push).

#### Tier 🟢 (baixa prioridade)
- Páginas estáticas, dashboards informativos.
- Código já marcado como legacy (5 portais arquivados na Onda 1).

### 3. Sugerir stack mínimo

Pra Lumied (Deno/Node mix):
- **Vitest** pra Node/TS (mais rápido que Jest, ESM-friendly).
- **Deno test** pra `supabase/functions/**` (built-in, sem dep extra).
- **Playwright** pra E2E quando chegar Tier 🟡.

Não introduzir frameworks pesados (Cypress, Storybook) na Onda 2.

### 4. Output esperado

```
## Test coverage — Lumied

Estado atual:
- Arquivos de teste: 0
- Framework: nenhum configurado
- CI: não roda testes

Prioridade 🔴 (testar antes de Onda 3):
1. supabase/functions/boletins_publicar — tenant isolation
2. supabase/functions/bank-inter-webhook — assinatura + idempotência
3. supabase/migrations RLS policies — teste de "user A não vê dados de user B"
4. auth role guards em edge functions críticas
5. backup restore — restore-test em ambiente sandbox

Recomendação:
- Setar Vitest + Deno test ANTES de escrever testes.
- Começar pelo item 1; cada um leva ~2-4h.
- Onda 3 só começa quando 🔴 está coberto.

Não fazer agora:
- E2E playwright (Tier 🟡, deixa pra depois).
- 100% coverage — meta inicial 50% nos paths 🔴.
```

## Princípios

1. **Testar o que dói se quebrar** — não o que é fácil testar. Tenant leak é caro, util de string não é.
2. **Setup primeiro, teste depois.** Sem framework rodando no CI, testes viram bit rot.
3. **Cobertura é métrica orientativa, não meta.** 80% coverage com testes ruins é pior que 30% com testes bons.
4. **Cada incidente gera teste.** [[postmortem]] deve incluir "teste que teria pego".
5. **Pre-merge gate.** Onda 4 deveria ter CI que bloqueia merge se cobertura caiu — não tarefa pra agora.

## Anti-padrões

- Começar testando UI antes de domain logic.
- Mock do Supabase em vez de DB de teste — replica o anti-padrão do [[feedback_repo_public_secrets]] que não é deste contexto, mas a regra geral é: mock esconde bug real (já temos memory dessa lição).
- Snapshot testing como muleta — gera ruído sem detectar regressão real.
- Pular testes "porque o feature é simples" — feature simples vira complexa em 3 meses.

## Pós-coverage

- Atualizar [[project_refator_lumied]] com status "Onda 2 — Tier 🔴 X% coberto".
- Quando Onda 2 fecha, abrir Onda 3 com confiança.
- Salvar `feedback_test_*` se aprendermos padrões reutilizáveis (ex: como mockar webhook de banco).
