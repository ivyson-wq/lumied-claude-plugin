---
name: edge-fn-new
description: Scaffold de edge function Supabase nova já com boilerplate Lumied — JWT check, role guard, escola_id do JWT (não do body), CORS, error handler padrão. Reduz risco de criar função vulnerável e dá consistência. Use quando o usuário pedir "cria edge function X" ou "preciso de um endpoint pra Y".
---

# Edge function scaffold — Lumied

Contexto: complementa [[edge-fn-authz]] (que audita). Esta skill **cria** com o padrão correto desde o início. Toda edge function nova no Lumied deve seguir este template — ele já passa em `tenant-audit`, `edge-fn-authz` e `secrets-scan`.

## Onde criar

```
supabase/functions/<nome>/index.ts
```

Convenção de nome: `snake_case`, descritivo. Prefixos por área:
- `alm_*` — almoxarifado.
- `pdi_*` — PDI / RH.
- `boletins_*` — notas/frequência.
- `cron_*` — disparados por pg_cron (auth via `x-cron-key`).
- `bank_*` — adapters de banco.
- `webhook_*` — recebe POST externo (assinatura ao invés de JWT).

## Template base (ação user-facing)

```ts
// supabase/functions/<nome>/index.ts
import { serve } from 'https://deno.land/std@0.208.0/http/server.ts'
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2'

const CORS = {
  'Access-Control-Allow-Origin': '*',
  'Access-Control-Allow-Methods': 'POST, OPTIONS',
  'Access-Control-Allow-Headers': 'authorization, x-client-info, apikey, content-type',
}

const json = (body: unknown, status = 200) =>
  new Response(JSON.stringify(body), {
    status,
    headers: { ...CORS, 'Content-Type': 'application/json' },
  })

const SUPABASE_URL = Deno.env.get('SUPABASE_URL')!
const SUPABASE_ANON_KEY = Deno.env.get('SUPABASE_ANON_KEY')!
const SUPABASE_SERVICE_ROLE_KEY = Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')!

const ALLOWED_ROLES = ['gerente', 'secretaria'] // <-- ajustar por função

const ACTIONS = ['listar', 'criar', 'atualizar'] // <-- allowlist (vide [[feedback_diplomas_allowlist]])

serve(async (req) => {
  if (req.method === 'OPTIONS') return new Response(null, { headers: CORS })
  if (req.method !== 'POST') return json({ error: 'method_not_allowed' }, 405)

  try {
    // 1. Auth: JWT
    const auth = req.headers.get('Authorization') ?? ''
    if (!auth.startsWith('Bearer ')) return json({ error: 'unauthorized' }, 401)

    const userClient = createClient(SUPABASE_URL, SUPABASE_ANON_KEY, {
      global: { headers: { Authorization: auth } },
    })
    const { data: { user }, error: authErr } = await userClient.auth.getUser()
    if (authErr || !user) return json({ error: 'unauthorized' }, 401)

    // 2. Authz: role
    const role = user.app_metadata?.role ?? user.user_metadata?.role
    if (!ALLOWED_ROLES.includes(role)) return json({ error: 'forbidden' }, 403)

    // 3. Tenant: escola_id SEMPRE do JWT, NUNCA do body
    const escolaId = user.app_metadata?.escola_id ?? user.user_metadata?.escola_id
    if (!escolaId) return json({ error: 'missing_escola' }, 400)

    // 4. Payload
    const body = await req.json().catch(() => ({}))
    const { action, ...payload } = body
    if (!ACTIONS.includes(action)) return json({ error: 'Ação desconhecida' }, 400)

    // 5. Service client (bypass RLS — use com cuidado, todas queries devem filtrar escola_id manualmente)
    const admin = createClient(SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY)

    // 6. Dispatcher
    switch (action) {
      case 'listar':
        return await handleListar(admin, escolaId, payload)
      case 'criar':
        return await handleCriar(admin, escolaId, user.id, payload)
      case 'atualizar':
        return await handleAtualizar(admin, escolaId, payload)
    }

    return json({ error: 'not_implemented' }, 501)
  } catch (e) {
    console.error('[<nome>] erro:', e)
    return json({ error: 'internal_error', detail: String(e?.message ?? e) }, 500)
  }
})

// --- Handlers ---

async function handleListar(admin: any, escolaId: string, _payload: any) {
  const { data, error } = await admin
    .from('minha_tabela')
    .select('*')
    .eq('escola_id', escolaId)  // <-- SEMPRE filtrar tenant
    .order('criado_em', { ascending: false })
    .limit(100)
  if (error) throw error
  return json({ data })
}

async function handleCriar(admin: any, escolaId: string, userId: string, payload: any) {
  // validar payload primeiro
  if (!payload.nome) return json({ error: 'nome obrigatório' }, 400)
  const { data, error } = await admin
    .from('minha_tabela')
    .insert({
      escola_id: escolaId,   // <-- nunca aceitar do client
      criado_por: userId,
      nome: payload.nome,
    })
    .select()
    .single()
  if (error) throw error
  return json({ data })
}

async function handleAtualizar(admin: any, escolaId: string, payload: any) {
  if (!payload.id) return json({ error: 'id obrigatório' }, 400)
  const { data, error } = await admin
    .from('minha_tabela')
    .update({ nome: payload.nome })
    .eq('id', payload.id)
    .eq('escola_id', escolaId)  // <-- guard: só mexe no próprio tenant
    .select()
    .single()
  if (error) throw error
  if (!data) return json({ error: 'not_found' }, 404)
  return json({ data })
}
```

## Template variante: cron (`cron_*`)

```ts
serve(async (req) => {
  const cronKey = req.headers.get('x-cron-key')
  if (cronKey !== Deno.env.get('CRON_INTERNAL_KEY')) {
    return new Response('forbidden', { status: 403 })
  }
  const admin = createClient(SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY)

  // Cron itera por todas as escolas — não há escola_id no contexto
  const { data: escolas } = await admin.from('escolas').select('id').eq('ativa', true)

  for (const escola of escolas ?? []) {
    try {
      await processar(admin, escola.id)
    } catch (e) {
      console.error('[cron_<nome>] escola', escola.id, e)
      // continuar pra próxima — não interromper o cron inteiro
    }
  }

  return new Response('ok')
})
```

## Template variante: webhook externo

```ts
serve(async (req) => {
  // Validar assinatura HMAC (ou similar)
  const signature = req.headers.get('x-signature') ?? ''
  const body = await req.text()
  if (!await verifySignature(body, signature, Deno.env.get('WEBHOOK_SECRET')!)) {
    return new Response('forbidden', { status: 403 })
  }

  const payload = JSON.parse(body)
  // escola_id precisa vir da mapping do webhook (não confiar no body)
  // ...
})
```

## Procedimento ao criar nova função

1. Pegar do usuário: nome, propósito, role permitido, actions.
2. Criar `supabase/functions/<nome>/index.ts` com template apropriado.
3. Listar tabelas que a função vai tocar — confirmar que todas têm `escola_id` + RLS ([[rls-check]]).
4. Adicionar entrada em `supabase/config.toml` se necessário.
5. Rodar mental [[edge-fn-authz]] no template gerado.
6. Avisar o usuário pra:
   - Implementar handlers reais (template usa `minha_tabela` placeholder).
   - Testar local: `supabase functions serve <nome>` + `curl`.
   - Adicionar à chamada cliente (action no front).
   - Deploy via `lumied-deploy`.

## Checklist final (gere após criar)

- [ ] JWT verificado.
- [ ] Role validada.
- [ ] `escola_id` do JWT, não do body.
- [ ] Toda query tem `.eq('escola_id', escolaId)`.
- [ ] Allowlist de actions explícita.
- [ ] CORS headers.
- [ ] `try/catch` global com log + json error.
- [ ] OPTIONS preflight handled.
