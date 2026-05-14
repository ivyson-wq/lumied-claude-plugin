---
name: cron-health
description: Lista e diagnostica pg_cron jobs nos projetos Supabase do Lumied e Construfare — mostra schedule, último run, status, falhas. Use quando o usuário pergunta "tá tudo rodando?", "algum cron quebrou?", "como tá o sienge reconcile/outbound-pulse/backup", ou após um deploy que mexeu em jobs.
---

# Cron health check (pg_cron)

Contexto: já houve [[project_backlog_v2_pendencias]] com **6 crons quebrados simultaneamente** que só apareceram em auditoria manual. O `backup-escolas` ficou quebrado por 22 dias sem ninguém perceber. Esta skill é o checkup periódico.

## Projetos

- **Lumied**: `brgorknbrjlfwvrrlwxj` — usa `SUPABASE_ACCESS_TOKEN` do [[reference_lumied_supabase]]
- **Construfare**: ref + token em [[reference_construfare_supabase]]

## Query padrão

```sql
SELECT
  j.jobname,
  j.schedule,
  j.active,
  jr.start_time AS last_run,
  jr.status     AS last_status,
  jr.return_message,
  EXTRACT(EPOCH FROM (NOW() - jr.start_time))/3600 AS hours_since_run
FROM cron.job j
LEFT JOIN LATERAL (
  SELECT * FROM cron.job_run_details
  WHERE jobid = j.jobid
  ORDER BY start_time DESC LIMIT 1
) jr ON true
ORDER BY j.active DESC, jr.start_time DESC NULLS LAST;
```

Rodar via Management API:
```bash
SQL='<query acima>'
curl --ssl-no-revoke -s -X POST \
  "https://api.supabase.com/v1/projects/$REF/database/query" \
  -H "Authorization: Bearer $SUPABASE_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d "$(jq -nc --arg q "$SQL" '{query:$q}')"
```

## O que sinalizar

- **CRÍTICO** (vermelho): `active=false` em job que devia rodar, OU `last_status='failed'`, OU `hours_since_run > schedule_esperado * 2`.
- **AVISO** (amarelo): job que nunca rodou (`last_run IS NULL`), ou rodou ok mas faz >24h em job que devia ser horário.
- **OK** (verde): rodou recente, status='succeeded'.

## Jobs conhecidos que devem estar ativos

Lumied:
- `backup-escolas-daily` (03:00 BRT, [[project_backups]]) — **historicamente quebrado**
- `outbound-pulse-*` ([[project_gh_actions_tier]] migrou pra pg_cron)
- `reval-warmup` ([[project_reval_proxy]])
- `ticket-resolver-15min`

Construfare:
- `sienge-reconcile-daily` ([[project_sienge_reconcile]])
- `reconcile-coverage-health` ([[project_construfare_reconcile_coverage]])

## Saída esperada

Tabela compacta:
```
PROJ        JOB                          SCHED        LAST_RUN          STATUS    NOTA
lumied      backup-escolas-daily         0 6 * * *    2026-04-22 06:00  failed    >22d sem rodar OK ⚠
lumied      outbound-pulse-15min         */15 * * *   2026-05-14 14:30  ok        ✓
construfare sienge-reconcile-daily       0 4 * * *    2026-05-14 04:01  ok        ✓
```

Se algum CRÍTICO, ofereça investigar (`SELECT * FROM cron.job_run_details WHERE jobid = X ORDER BY start_time DESC LIMIT 5`) mas **não tente "consertar" disparando manualmente** — [[project_sienge_reconcile]] avisa explicitamente "NÃO disparar migrate manualmente".
