---
name: secrets-scan
description: Varre o repo procurando secrets hardcoded (Supabase service_role, CRON_INTERNAL_KEY, tokens de bancos, ML_*, Resend, Sentry DSN). Use antes de tornar repo público, antes de push grande, ou quando suspeitar de vazamento. Crítico depois que o repo Lumied virou público em 14/05/2026.
---

# Secrets scan — Lumied / Construfare

Contexto: [[feedback_repo_public_secrets]] — o repo Lumied virou público em 2026-05-14 pra escapar do cap GH Actions ([[project_gh_actions_tier.md]]). Voltará privado depois do reset em ~01/06 ([[project_private_after_reset]]). Enquanto isso, qualquer secret hardcoded fica exposto na história git pública.

## Quando rodar

- Antes de `git push` quando o repo está público.
- Antes de tornar qualquer outro repo público.
- Quando o usuário pedir explicitamente ("varre secrets", "tem chave hardcoded?").
- Inclua em `deploy-preflight`.

## O que procurar

### Tokens críticos (rotacionar IMEDIATAMENTE se achados)

| Padrão | Onde costuma vazar | Severidade |
|---|---|---|
| `eyJ[A-Za-z0-9_-]{20,}\.eyJ[A-Za-z0-9_-]{20,}\.` | Supabase service_role / anon JWT | 🔴 alta se service_role |
| `sbp_[a-f0-9]{40,}` | Supabase Management API token | 🔴 alta |
| `CRON_INTERNAL_KEY\s*[:=]\s*["'][^"']{8,}` | edge functions, scripts | 🔴 alta |
| `[A-Z0-9]{20,}-[A-Z0-9]{20,}` (Inter/banco) | adapters de banco | 🔴 alta |
| `re_[A-Za-z0-9]{20,}` | Resend API key | 🟡 média |
| `https://[a-f0-9]+@[a-z0-9.]+\.ingest\.sentry\.io` | Sentry DSN (público ok, mas auth token não) | 🟢 baixa |
| `ML_CLIENT_SECRET` / `ML_REFRESH_TOKEN` | Mercado Livre OAuth | 🟡 média |
| `ghp_[A-Za-z0-9]{36}` | GitHub PAT | 🔴 alta |
| `AKIA[0-9A-Z]{16}` | AWS access key | 🔴 alta |

### Padrões textuais suspeitos

- `password\s*[:=]\s*["'][^"']+["']`
- `secret\s*[:=]\s*["'][^"']+["']` (excluir comentários / placeholders `<...>`)
- `Authorization:\s*Bearer\s+[A-Za-z0-9_-]{20,}` em arquivos não-test

## Procedimento

1. Grep cada padrão acima em `**/*.{ts,js,tsx,jsx,sql,sh,html,json,yml,yaml,toml,env*}` — excluindo `node_modules/`, `.next/`, `dist/`, `coverage/`.
2. Pra cada match, classificar:
   - **VAZA** — secret real hardcoded → rotação obrigatória + remover do código.
   - **PLACEHOLDER** — `<your-key-here>`, `process.env.FOO`, `Deno.env.get(...)` → ok.
   - **TEST** — fixture em arquivo `*.test.*` ou `*spec*` → revisar mas baixa prioridade.
3. Cheque também `.env*` files: estão no `.gitignore`? Verifique `git log --all --full-history -- .env*` pra ver se algum vazou no passado.
4. Reporte:
   ```
   🔴 supabase/functions/foo/index.ts:42  CRON_INTERNAL_KEY hardcoded → ROTAR
   🟡 scripts/deploy.sh:15               token Resend em comentário → remover
   🟢 .env.example                       só placeholders OK
   ```
5. Se houver VAZA: **bloqueie o push** e ofereça plano de rotação:
   - Gerar nova chave no provider.
   - Substituir em todos os ambientes (Vercel env, Supabase secrets, GH Actions secrets).
   - Reescrever histórico se vazou em commit antigo (`git filter-repo` ou BFG).
   - Documentar em `incident-postmortem` se já foi pushed.

## Falsos positivos comuns

- JWT de teste em fixtures — ok se em `*.test.*` ou `__fixtures__/`.
- Hash bcrypt/argon2 — não é secret, é resultado.
- URLs públicas de Sentry/PostHog — DSN público é safe-by-design.
- Tokens em `*.md` de documentação **devem** ser placeholders.

## Anti-padrões a destacar

- Token no body de uma migration SQL.
- Token em `console.log()` (vaza pro Supabase logs visíveis a colaboradores).
- `process.env.FOO || 'fallback-real-token'` — fallback hardcoded é o pior.
