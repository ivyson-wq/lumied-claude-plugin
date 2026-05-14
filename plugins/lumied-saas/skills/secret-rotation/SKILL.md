---
name: secret-rotation
description: Plano de rotação de secrets (Supabase service_role, CRON_INTERNAL_KEY, tokens de bancos, ML_*, Resend, Vercel env) — sequência segura para substituir sem causar downtime. Use periodicamente, após repo virar público, ou quando `secrets-scan` detectar vazamento.
---

# Secret rotation — Lumied / Construfare

Contexto: complementa [[secrets-scan]]. Quando um secret vaza ou caduca, rotacionar errado = downtime (mudou no Vercel mas não no Supabase, ou vice-versa). Esta skill descreve a ordem segura.

## Inventário de secrets

| Secret | Onde vive | Onde é usado | Janela de rotação |
|---|---|---|---|
| `SUPABASE_SERVICE_ROLE_KEY` | Supabase dashboard | Edge functions (env), Vercel env, scripts admin | 6 meses |
| `SUPABASE_MANAGEMENT_API_TOKEN` (sbp_*) | Supabase account settings | Local `.env`, GH Actions secret | 6 meses |
| `CRON_INTERNAL_KEY` | manual (custom) | Edge cron functions + pg_cron headers | 3 meses ou pós-vazamento |
| `INTER_CLIENT_SECRET` / `SICREDI_*` / `BB_*` / `ITAU_*` / `BRADESCO_*` | Bank console | Adapter edge functions, Vercel env, Supabase secrets | conforme banco (alguns 90d) |
| `ML_CLIENT_SECRET`, `ML_REFRESH_TOKEN` | Mercado Livre dev portal | Supabase secrets (almoxarifado) | refresh ~6h, secret 1 ano |
| `RESEND_API_KEY` | Resend dashboard | Supabase secrets (emails) | 1 ano |
| `SENTRY_AUTH_TOKEN` | Sentry settings | GH Actions secret (sourcemaps) | 1 ano |
| `VERCEL_TOKEN` | Vercel account | Local `.env`, GH Actions | 6 meses |

## Princípio: dual-write durante a rotação

Sempre que possível, **criar a nova chave + manter a antiga ativa**, atualizar todos os consumers, depois revogar a antiga. Single-step rotation = janela de downtime quase certa.

## Sequência genérica

1. **Gerar nova chave** no provider sem revogar a atual.
2. **Listar consumers** (busca `grep` no repo + lista de env vars no Vercel + secrets no Supabase + GH Actions secrets).
3. **Atualizar consumers em ordem**:
   - Local `.env` primeiro (não afeta prod).
   - Supabase secrets (`supabase secrets set CHAVE=valor --project-ref ...`).
   - Vercel env (`vercel env add` ou via dashboard) — depois `vercel --prod` pra picking up.
   - GH Actions secrets (gh secret set).
4. **Verificar** que tudo funciona com a nova chave (smoke test no endpoint crítico).
5. **Revogar** a chave antiga no provider.
6. **Documentar** rotação em `audit_log_global` se Lumied/Construfare tiver tracker, ou em commit message.

## Casos específicos

### CRON_INTERNAL_KEY

Usada em:
- Edge functions `cron-*` (header `x-cron-key`).
- pg_cron jobs (`SELECT net.http_post(..., headers => jsonb_build_object('x-cron-key', '<key>'))`).

Rotação:
1. Definir novo valor: `NEW_KEY=$(openssl rand -hex 32)`.
2. Adicionar no Supabase secrets: `supabase secrets set CRON_INTERNAL_KEY_NEW=$NEW_KEY` (temporário).
3. Edge functions: aceitar **ambas** as keys temporariamente:
   ```ts
   const validKeys = [Deno.env.get('CRON_INTERNAL_KEY'), Deno.env.get('CRON_INTERNAL_KEY_NEW')]
   if (!validKeys.includes(req.headers.get('x-cron-key'))) return 403
   ```
4. Atualizar pg_cron jobs para enviar `NEW_KEY` (via mig update).
5. Deploy edge functions + aplicar mig.
6. Swap: `supabase secrets set CRON_INTERNAL_KEY=$NEW_KEY`, `supabase secrets unset CRON_INTERNAL_KEY_NEW`.
7. Limpar código (remover validKeys array).

### Tokens de bancos (Inter/Sicredi/BB/Itaú/Bradesco)

Cada banco tem janela própria. [[project_banks_multiprovider]].
- Inter: client_secret 1 ano; access_token ~1h (refresh automático).
- Sicredi: certificado mTLS — rotação anual.
- BB: client_secret 6 meses.

Sempre testar com `/api/banks/<banco>/health` antes de descartar a antiga.

### Supabase service_role

Mais perigoso. Edge functions e Vercel functions usam isso. Rotação:
1. Gerar via Supabase dashboard → Settings → API → "Generate new key" (cria nova, mantém antiga 24h ou até revogar).
2. `supabase secrets set SUPABASE_SERVICE_ROLE_KEY=<nova>` + `vercel env add` + redeploy.
3. Smoke test edge functions críticas.
4. Revogar antiga no dashboard.

## Plano de emergência (vazou agora)

Se `secrets-scan` ou alerta detectou vazamento já em prod/git público:

1. **Revogar imediato** no provider (não dual-write — assume worst case).
2. Gerar nova.
3. Atualizar consumers o mais rápido possível.
4. Aceitar janela de downtime curta.
5. Se vazou no git: `git filter-repo` ou BFG pra remover do histórico (mas o secret JÁ está comprometido — só rotação resolve).
6. Disparar `postmortem`.

## Reporte

Pra cada rotação, gerar um sumário:
```
Rotação: CRON_INTERNAL_KEY
Motivo: rotina trimestral (último: 2026-02-15)
Consumers atualizados: 12 edge functions + 8 pg_cron jobs
Verificação: ✅ cron-health passou em todos
Antiga revogada: 2026-05-14 18:42
```
