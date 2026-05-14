---
name: migration-rollback
description: Gera o SQL de rollback (down migration) para uma migration `up` antes de aplicá-la, e descreve o procedimento de reverter se já foi aplicada. Crítico no Lumied porque `lumied-deploy` aplica via Management API sem framework de migration nativo. Use antes de cada `supabase/migrations/NNN_*.sql` nova.
---

# Migration rollback — Lumied / Construfare

Contexto: o Lumied usa migrations SQL aplicadas via Management API ([[reference_lumied_supabase]]) sem `supabase db reset` ou framework. Se algo der errado, não tem `down` automático — precisa reverter à mão. Esta skill obriga gerar o rollback ANTES de aplicar, e tê-lo pronto pra se algo quebrar.

## Princípio

Pra cada migration `NNN_descricao.sql`, gerar também `NNN_descricao_rollback.sql` (ou seção `-- ROLLBACK` no topo do arquivo). NÃO commitar sem ele.

## Categorias de mudança e como reverter

### CREATE TABLE

Up:
```sql
CREATE TABLE pedidos_v2 (id uuid PRIMARY KEY, escola_id uuid, ...);
```

Rollback:
```sql
DROP TABLE IF EXISTS pedidos_v2 CASCADE;
```

⚠️ Se a tabela já tem dados em prod, `CASCADE` pode apagar FKs em cadeia. Sempre confirmar que rollback NÃO causa perda inesperada.

### ALTER TABLE ADD COLUMN

Up:
```sql
ALTER TABLE alunos ADD COLUMN data_segundo_responsavel date;
```

Rollback:
```sql
ALTER TABLE alunos DROP COLUMN IF EXISTS data_segundo_responsavel;
```

✅ Seguro se coluna ainda não foi populada. 🟡 perigoso se já tem dados que não vão pra outro lugar.

### ALTER COLUMN TYPE / RENAME

Up:
```sql
ALTER TABLE alunos ALTER COLUMN cpf TYPE varchar(14);
```

Rollback:
```sql
-- precisa do tipo original
ALTER TABLE alunos ALTER COLUMN cpf TYPE text;
```

🔴 Mudanças de tipo podem perder precisão (text→numeric, timestamp→date). Documentar perdas.

### DROP COLUMN / DROP TABLE

Up:
```sql
ALTER TABLE alunos DROP COLUMN telefone_antigo;
```

Rollback **inviável** por causa dos dados — só preservando antes:

```sql
-- BEFORE up: backup
CREATE TABLE _backup_alunos_telefone_antigo AS
SELECT id, telefone_antigo FROM alunos;

ALTER TABLE alunos DROP COLUMN telefone_antigo;

-- ROLLBACK:
ALTER TABLE alunos ADD COLUMN telefone_antigo text;
UPDATE alunos a SET telefone_antigo = b.telefone_antigo
  FROM _backup_alunos_telefone_antigo b WHERE a.id = b.id;
DROP TABLE _backup_alunos_telefone_antigo;
```

### CREATE/REPLACE FUNCTION

Up:
```sql
CREATE OR REPLACE FUNCTION calcular_mensalidade(p_aluno uuid) RETURNS numeric AS $$...$$;
```

Rollback: salvar a versão anterior ANTES.
```sql
-- Antes de aplicar up, capturar:
\df+ calcular_mensalidade
-- ou em SQL:
SELECT pg_get_functiondef(oid) FROM pg_proc WHERE proname='calcular_mensalidade';
```
Cole o `CREATE OR REPLACE FUNCTION ...` antigo no rollback.

### CREATE INDEX

Up:
```sql
CREATE INDEX CONCURRENTLY idx_pedidos_data ON pedidos(data_criacao);
```

Rollback:
```sql
DROP INDEX CONCURRENTLY IF EXISTS idx_pedidos_data;
```

Sempre `CONCURRENTLY` em prod pra não locar tabela.

### CREATE POLICY (RLS)

Up:
```sql
CREATE POLICY tenant_isolation_select ON nova_tabela FOR SELECT USING (...);
```

Rollback:
```sql
DROP POLICY IF EXISTS tenant_isolation_select ON nova_tabela;
```

### Triggers

Up:
```sql
CREATE TRIGGER force_escola_id BEFORE INSERT ON pedidos
  FOR EACH ROW EXECUTE FUNCTION enforce_escola_id();
```

Rollback:
```sql
DROP TRIGGER IF EXISTS force_escola_id ON pedidos;
```

### pg_cron jobs

Up:
```sql
SELECT cron.schedule('backup_daily', '0 3 * * *', $$ ... $$);
```

Rollback:
```sql
SELECT cron.unschedule('backup_daily');
```

### Storage buckets

Up:
```sql
INSERT INTO storage.buckets (id, name, public) VALUES ('nova-bucket', 'nova-bucket', false);
```

Rollback:
```sql
-- ⚠️ só seguro se bucket vazio
DELETE FROM storage.objects WHERE bucket_id = 'nova-bucket';
DELETE FROM storage.buckets WHERE id = 'nova-bucket';
```

### INSERT/UPDATE em dados (data migration)

Up:
```sql
UPDATE escolas SET tier = 'starter' WHERE plano = 'free';
```

Rollback: precisa salvar estado anterior antes.
```sql
CREATE TEMP TABLE _bkp AS SELECT id, tier FROM escolas WHERE plano='free';
-- ... up ...
-- ROLLBACK:
UPDATE escolas e SET tier = b.tier FROM _bkp b WHERE e.id = b.id;
```

## Procedimento

1. Abrir o arquivo `supabase/migrations/NNN_*.sql` pendente.
2. Listar cada DDL/DML.
3. Pra cada um, gerar a operação inversa usando a tabela acima.
4. Inserir no topo do mesmo arquivo numa seção comentada:
   ```sql
   -- =====================
   -- ROLLBACK
   -- =====================
   -- (cole aqui o SQL pra reverter este arquivo)
   -- =====================
   ```
   OU criar arquivo paralelo `NNN_descricao_rollback.sql`.
5. Registrar em `schema_migrations` quando aplicar; **não registrar até confirmar success**.
6. Se rollback for inviável (drop column com dados), avisar o usuário ANTES e exigir confirmação explícita + backup separado.

## Em emergência (mig já aplicada e quebrou)

1. **Não entrar em pânico** — primeiro identificar o que quebrou.
2. Se for performance (index errado): drop ele.
3. Se for lógica (função quebrada): aplicar versão anterior via Management API.
4. Se for dados perdidos: restaurar do backup ([[project_backups]]). Backups Lumied rodam 03h BRT.
5. Disparar [[postmortem]].

## Anti-padrões

- Mig que mistura DDL + grandes DMLs no mesmo arquivo (rollback fica gigante).
- `DROP COLUMN` sem backup prévio.
- Renomeação de coluna `RENAME` (rollback técnico fácil, mas quebra app entre os deploys).
- Migrations não-idempotentes — sempre `IF NOT EXISTS` / `IF EXISTS`.
