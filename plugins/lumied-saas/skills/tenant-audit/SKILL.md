---
name: tenant-audit
description: Audita código novo (edge functions, SQL, queries) buscando ausência de filtro `tenant_id` ou `escola_id`, que é a causa-raiz do incidente de vazamento de dados Lumied 16/04/2026. Use sempre antes de deploy de edge function nova/modificada, ou quando o usuário pede "audita isso", "tem risco de vazar", "checa isolation".
---

# Tenant isolation audit

Contexto: [[project_tenant_isolation_incident]] — em 16/04/2026 um vazamento visual cruzou demo→Maple Bear porque queries em edge functions não filtravam por `escola_id` / `tenant_id`. As migs 243/244 + trigger fecharam parte; sobraram **7 edge functions ambíguas pendentes** + qualquer função nova entra na lista de risco.

## Quando rodar

- Antes de `supabase functions deploy` ([[lumied-deploy]] passo 4).
- Ao revisar PR que toca `supabase/functions/**` ou `supabase/migrations/**`.
- Quando o usuário pede explicitamente.

## O que checar

### Em `supabase/functions/<nome>/index.ts`

Procure queries Supabase sem filtro de tenant:

```ts
// SUSPEITO — sem .eq('escola_id', ...) ou .eq('tenant_id', ...)
const { data } = await supabase.from('alunos').select('*')

// SUSPEITO — RPC que recebe params do client sem validação de tenant
const { data } = await supabase.rpc('algum_rpc', { ...body })
```

Padrões aceitáveis:
- Filtro explícito: `.eq('escola_id', escolaId)` onde `escolaId` veio do JWT/sessão (não do body).
- Tabela é global por design (ex: `escolas`, `tenants`, `health_checks`).
- Service role + função RPC que internamente filtra (precisa abrir o SQL da função).

### Em `supabase/migrations/*.sql`

- Toda tabela nova com dados sensíveis precisa de coluna `escola_id uuid REFERENCES escolas(id)` + RLS policy:
  ```sql
  ALTER TABLE nova_tabela ENABLE ROW LEVEL SECURITY;
  CREATE POLICY tenant_isolation ON nova_tabela
    USING (escola_id = current_setting('app.escola_id')::uuid);
  ```
- Funções `SECURITY DEFINER` precisam validar `escola_id` no corpo.

## Procedimento

1. `git diff origin/main...HEAD -- supabase/functions/ supabase/migrations/` — pega só o delta da branch.
2. Pra cada arquivo modificado, listar queries/operações.
3. Pra cada uma, classificar: **OK** / **SUSPEITO** / **NÃO APLICÁVEL**.
4. Reporte em formato:
   ```
   FILE: supabase/functions/foo/index.ts
     L42  SUSPEITO  supabase.from('alunos').select() sem .eq('escola_id', ...)
     L67  OK        usa escolaId do session.user.user_metadata
   ```
5. Se houver SUSPEITO, **bloqueie o deploy** e ofereça o fix. Não aplique sem confirmar com o usuário.

## Anti-padrões conhecidos

- Receber `escola_id` no body sem validar contra o JWT.
- Service role key vazando pro client.
- Trigger `BEFORE INSERT` que não força `escola_id = current_setting(...)`.
- View que faz JOIN entre tabelas tenant-scoped sem WHERE.
