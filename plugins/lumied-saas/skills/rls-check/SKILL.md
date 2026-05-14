---
name: rls-check
description: Audita migrations SQL novas para garantir que toda tabela com dados sensíveis tem coluna `escola_id`/`tenant_id` + RLS habilitado + policy de isolamento. Complementa `tenant-audit` (que olha edge functions). Use ao revisar mig nova, antes de aplicar no Supabase, ou quando o usuário pedir "checa RLS dessa tabela".
---

# RLS check — Lumied / Construfare

Contexto: [[project_tenant_isolation_incident]] mostrou que filtro na query é insuficiente — sem RLS no banco, qualquer edge function (ou query manual via Studio) pode pular tenant. As migs 243/244 fecharam isso pras tabelas existentes; **toda tabela nova precisa nascer com RLS já correto**.

## Quando rodar

- Antes de aplicar `supabase/migrations/NNN_*.sql` via Management API.
- Em PRs que tocam `supabase/migrations/**`.
- Em `deploy-preflight`.

## Checklist por tabela nova

Para cada `CREATE TABLE` na migration, verificar:

1. **Coluna `escola_id`** existe e é `NOT NULL`?
   ```sql
   escola_id uuid NOT NULL REFERENCES escolas(id) ON DELETE CASCADE
   ```
   Exceções aceitáveis (registrar quais):
   - Tabelas globais: `escolas`, `tenants`, `health_checks`, `schema_migrations`, `audit_log_global`.
   - Tabela ligada a outra tenant-scoped por FK obrigatória (transitividade).

2. **RLS habilitado**?
   ```sql
   ALTER TABLE nova_tabela ENABLE ROW LEVEL SECURITY;
   ```

3. **Policy de leitura/escrita** com filtro `escola_id`?
   ```sql
   CREATE POLICY tenant_isolation_select ON nova_tabela
     FOR SELECT USING (escola_id = (current_setting('request.jwt.claims', true)::json->>'escola_id')::uuid);
   CREATE POLICY tenant_isolation_modify ON nova_tabela
     FOR ALL USING (escola_id = (current_setting('request.jwt.claims', true)::json->>'escola_id')::uuid)
     WITH CHECK (escola_id = (current_setting('request.jwt.claims', true)::json->>'escola_id')::uuid);
   ```
   Variantes aceitas (depende do padrão do projeto): `current_setting('app.escola_id')::uuid` se for set via trigger/handler.

4. **Trigger de força** (defesa em profundidade — padrão Lumied desde mig 244):
   ```sql
   CREATE TRIGGER force_escola_id BEFORE INSERT ON nova_tabela
     FOR EACH ROW EXECUTE FUNCTION enforce_escola_id();
   ```

5. **Índice em `escola_id`** se a tabela é grande:
   ```sql
   CREATE INDEX idx_nova_tabela_escola ON nova_tabela(escola_id);
   ```

## Funções SECURITY DEFINER

Funções `SECURITY DEFINER` ignoram RLS — então **precisam validar escola_id no corpo**:

```sql
CREATE FUNCTION minha_rpc(p_aluno_id uuid)
RETURNS json
SECURITY DEFINER
AS $$
DECLARE
  v_escola uuid := (current_setting('request.jwt.claims', true)::json->>'escola_id')::uuid;
BEGIN
  -- 🔴 SUSPEITO se faltar este check:
  IF NOT EXISTS (SELECT 1 FROM alunos WHERE id = p_aluno_id AND escola_id = v_escola) THEN
    RAISE EXCEPTION 'forbidden';
  END IF;
  -- ... corpo da função
END $$ LANGUAGE plpgsql;
```

## Views

Views herdam RLS da tabela base **somente se** criadas com `WITH (security_invoker=true)` (Postgres 15+). Sem isso, view roda como o owner e vaza tudo.

```sql
CREATE VIEW v_alunos_resumo
  WITH (security_invoker=true)  -- 🔴 obrigatório
AS SELECT ... FROM alunos JOIN ...;
```

## Procedimento

1. `git diff origin/main...HEAD -- supabase/migrations/` para pegar o delta.
2. Pra cada `CREATE TABLE`, rodar checklist acima.
3. Pra cada `CREATE FUNCTION ... SECURITY DEFINER`, verificar guard no corpo.
4. Pra cada `CREATE VIEW`, verificar `security_invoker=true`.
5. Reporte:
   ```
   mig 331_add_xyz.sql:
     ✅ tabela xyz_logs   — escola_id NOT NULL + RLS + policy + trigger
     🔴 tabela xyz_audit  — RLS habilitado mas SEM policy (= bloqueia tudo, ou pior, libera com bypass)
     🟡 view v_xyz        — falta security_invoker=true
   ```
6. Se houver 🔴, **bloqueie a aplicação** e ofereça patch SQL.

## Anti-padrões conhecidos

- `ENABLE ROW LEVEL SECURITY` sem nenhuma `CREATE POLICY` (efeito: bloqueia tudo até pro service_role em certos casos; ou nada, dependendo).
- `USING (true)` em policy "temporária" que vira permanente.
- Policy só pra `SELECT` mas falta pra `INSERT/UPDATE/DELETE`.
- `WITH CHECK` ausente em policy de modify (permite escrever em tenant alheio mesmo bloqueando leitura).
- View materializada sem `security_invoker`.
