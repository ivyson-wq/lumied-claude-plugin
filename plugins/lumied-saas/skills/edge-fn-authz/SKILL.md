---
name: edge-fn-authz
description: Audita edge functions novas/modificadas para garantir validação correta de JWT + role + escola_id ANTES de qualquer operação no banco. Cobre o gap que `tenant-audit` deixa (RLS na query) focando na camada de auth na entrada. Use em PR review de `supabase/functions/**` ou antes do deploy.
---

# Edge function authz audit — Lumied

Contexto: complementa [[tenant-audit]] (que olha se a query filtra tenant) e [[rls-check]] (que olha o banco). Esta skill foca na **porta de entrada**: a edge function autentica e autoriza ANTES de tocar qualquer dado?

## Checklist obrigatório por edge function

Pra cada `supabase/functions/<nome>/index.ts`, conferir nesta ordem:

### 1. Verificação de JWT presente

```ts
const authHeader = req.headers.get('Authorization')
if (!authHeader?.startsWith('Bearer ')) {
  return new Response('unauthorized', { status: 401 })
}
```

🔴 Sem isso? Função é open-bar.

### 2. Validação do JWT

Duas opções aceitas no Lumied:

**A) Client com JWT do usuário (preferido pra ações user-facing):**
```ts
const supabase = createClient(SUPABASE_URL, SUPABASE_ANON_KEY, {
  global: { headers: { Authorization: authHeader } }
})
const { data: { user }, error } = await supabase.auth.getUser()
if (!user) return new Response('unauthorized', { status: 401 })
```

**B) Service role + parse manual do JWT (apenas pra cron/internal):**
```ts
const cronKey = req.headers.get('x-cron-key')
if (cronKey !== Deno.env.get('CRON_INTERNAL_KEY')) {
  return new Response('forbidden', { status: 403 })
}
```

🔴 Service role direto + dados do `body.escola_id` sem validar = vazamento garantido.

### 3. Extrair `escola_id` da identidade, NÃO do body

```ts
// ✅ CORRETO
const escolaId = user.user_metadata?.escola_id
  ?? user.app_metadata?.escola_id

// 🔴 ERRADO — client manda escola_id no body
const { escola_id } = await req.json()
```

Se a função precisa operar em escola diferente da do usuário (caso central admin), tem que verificar role explicitamente:

```ts
if (escolaIdAlvo !== escolaId && !user.app_metadata?.roles?.includes('admin_central')) {
  return new Response('forbidden', { status: 403 })
}
```

### 4. Validação de role para ações sensíveis

Mapeamento de role → ações no Lumied (referência [[feedback_diplomas_allowlist]]):
- `aluno` / `pai` — só ler próprio dado.
- `prof` — ler dados da turma; escrever em diário/notas.
- `gerente` / `secretaria` — CRUD escola.
- `admin_central` — global, incluindo cross-tenant.
- `cron` — apenas funções com `x-cron-key`.

```ts
const role = user.app_metadata?.role
if (!['gerente', 'secretaria'].includes(role)) {
  return new Response('forbidden', { status: 403 })
}
```

### 5. Allowlist de action (se for handler dispatcher)

Funções tipo `alm_prof_actions` que recebem `{ action, ...payload }` precisam de allowlist:

```ts
const isAlmProfAction = (a: string) =>
  ['listar_pedidos', 'criar_pedido', ...].includes(a)

if (!isAlmProfAction(action)) {
  return new Response(JSON.stringify({ error: 'Ação desconhecida' }), { status: 400 })
}
```

🟡 Atenção [[feedback_diplomas_allowlist]]: action existe no switch mas falta na allowlist → retorna "Ação desconhecida" mesmo funcionando.

### 6. Rate limit / abuse guard (opcional mas recomendado)

Funções que enviam email/SMS/WhatsApp ou são caras devem ter throttle por user_id.

## Procedimento

1. `git diff origin/main...HEAD -- supabase/functions/` → lista de funções novas/modificadas.
2. Pra cada arquivo, percorrer checklist 1-5 (6 é nice-to-have).
3. Reporte:
   ```
   supabase/functions/diplomas_emit/index.ts
     ✅ JWT verificado
     ✅ escola_id vem de user_metadata
     🔴 NENHUMA verificação de role — qualquer aluno pode emitir diploma
     ⚠️  action `revogar_diploma` no switch mas falta no allowlist isAlmGerenteAction
   ```
4. Bloqueie deploy se houver 🔴.

## Anti-padrões frequentes

- "Eu já valido no banco com RLS" — mas se você usa service_role na edge, RLS é ignorado.
- `Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')` + accept body.escola_id sem cross-check.
- Função `cron-*` que aceita request normal além do cron key (combo perigoso).
- Reuse de helper `getEscolaId(req)` que dá fallback no body sem flag.
- CORS `Access-Control-Allow-Origin: *` em função que retorna PII.

## Quando rodar

- Antes de `supabase functions deploy` (inclua em `deploy-preflight`).
- Em PR review.
- Quando o usuário falar "audita autorização", "essa função é segura?", "alguém logado pode chamar isso?".
