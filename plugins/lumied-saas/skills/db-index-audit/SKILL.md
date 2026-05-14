---
name: db-index-audit
description: Audita queries lentas + tabelas grandes no Supabase Lumied/Construfare buscando ausência de índices em colunas usadas em WHERE/JOIN/ORDER. IO/Storage já estourou ([[project_lumied_io_storage]]); falta de índice é causa frequente. Use quando o usuário disser "tá lento", "consultar X demora", após [[cost-audit]] apontar IO alto, ou periodicamente em tabelas que crescem.
---

# Índice de DB — auditoria

Contexto: Free Tier Supabase tem IO limitado. Query sem índice em tabela de 100k+ linhas faz seq scan, queima IO, dispara o cap. Já vimos isso em [[project_lumied_io_storage]] (migs 304/305 + 2 Workers pra mitigar). Esta skill busca queries "calientes" no codebase e cruza com índices existentes pra apontar gaps.

## Quando rodar

- Tela/módulo reportado como lento.
- [[cost-audit]] mostrou IO Supabase alto.
- Tabela cresceu >10x desde criação.
- Antes de [[deploy-preflight]] de mig que adiciona query nova em tabela grande.
- Periodicamente (mensal) em tabelas core (alunos, lançamentos, movimentos).

## Procedimento

### 1. Identificar tabelas grandes

```sql
SELECT
  schemaname,
  relname AS tabela,
  n_live_tup AS linhas_estimadas,
  pg_size_pretty(pg_total_relation_size(relid)) AS tamanho_total
FROM pg_stat_user_tables
ORDER BY n_live_tup DESC
LIMIT 30;
```

Foco em tabelas com >10k linhas. <10k geralmente seq scan é OK.

### 2. Identificar queries lentas

Via Supabase Studio → Logs → SQL queries, ou:

```sql
SELECT
  calls,
  total_exec_time,
  mean_exec_time,
  query
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 20;
```

(Precisa de extensão `pg_stat_statements` habilitada.)

### 3. Olhar índices existentes

```sql
SELECT
  schemaname,
  tablename,
  indexname,
  indexdef
FROM pg_indexes
WHERE tablename = '<nome_tabela>'
ORDER BY indexname;
```

### 4. Cruzar com queries reais no código

```bash
# Onde o nome dessa tabela aparece em edge functions e SQL
grep -rE "from\(['\"]<tabela>['\"]\)|FROM <tabela>|JOIN <tabela>" supabase/
```

Pra cada match, identificar quais colunas estão em WHERE/JOIN/ORDER e se há índice cobrindo.

### 5. EXPLAIN ANALYZE numa query suspeita

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM lancamentos
WHERE escola_id = '<uuid>' AND vencimento >= '2026-01-01'
ORDER BY vencimento DESC LIMIT 50;
```

Sinais de problema:
- `Seq Scan` em tabela grande.
- `Filter:` com muitas rows removed by filter (filtragem em memória, índice ajudaria).
- `cost=...` muito alto na linha do topo.
- `Buffers: shared read=NNNN` alto (IO).

### 6. Padrões de índice no contexto Lumied

#### Tenant-scoped (sempre presente)

Toda tabela com `escola_id` deve ter índice composto **começando** por `escola_id`:

```sql
-- Mínimo
CREATE INDEX idx_<tabela>_escola ON <tabela>(escola_id);

-- Melhor (composto com coluna usada junto)
CREATE INDEX idx_lancamentos_escola_venc ON lancamentos(escola_id, vencimento);
```

#### Foreign keys

Toda FK deve ter índice (PG não cria automático em FK):
```sql
CREATE INDEX idx_lancamentos_aluno ON lancamentos(aluno_id);
```

#### Timestamps usados em ORDER BY

```sql
CREATE INDEX idx_<tabela>_criado_at ON <tabela>(escola_id, criado_at DESC);
```

#### Coluna de status (cardinality baixa, mas filtragem comum)

```sql
-- Partial index é mais leve
CREATE INDEX idx_<tabela>_status_aberto ON <tabela>(escola_id, vencimento)
WHERE status = 'aberto';
```

#### Texto / busca

```sql
-- Para LIKE com prefixo
CREATE INDEX idx_alunos_nome_lower ON alunos(escola_id, lower(nome) text_pattern_ops);

-- Para busca full-text (raro no Lumied)
CREATE INDEX idx_<tabela>_busca_tsv ON <tabela> USING GIN(to_tsvector('portuguese', coluna));
```

### 7. Output esperado

```
## Auditoria de índices — Lumied

Tabelas grandes sem índice adequado:

1. lancamentos (~280k linhas, 45MB)
   - WHERE escola_id + vencimento usado em 12 queries → ✘ sem índice composto
   - Sugestão: CREATE INDEX idx_lancamentos_escola_venc ON lancamentos(escola_id, vencimento);
   - Impacto: ~80% das queries da tabela; reduz IO drasticamente

2. movimentos_almox (~120k linhas, 22MB)
   - WHERE escola_id + item_id usado → ✘ só tem idx em escola_id
   - Sugestão: CREATE INDEX idx_mov_almox_escola_item ON movimentos_almox(escola_id, item_id);

3. ponto_registros (~95k linhas, 18MB)
   - WHERE funcionario_id + data usado → ✘ sem índice
   - Sugestão: CREATE INDEX idx_ponto_func_data ON ponto_registros(funcionario_id, data);

Já indexadas adequadamente:
- alunos (escola_id, ra) ✓
- escolas_modulos ✓
- contas_receber (escola_id, vencimento) ✓ (recente)

Tamanho extra estimado: ~15MB índices novos
Trade-off: aceitável dado IO atual em 84% do cap
```

## Anti-padrões

- Criar índice em toda coluna "por garantia" — INSERT/UPDATE fica lento + storage estoura.
- Não usar índice composto quando WHERE usa 2 colunas — PG não consegue combinar 2 idx single em alguns casos.
- Ordem errada no composto — `escola_id` deve vir primeiro (mais seletivo no contexto tenant).
- Esquecer `CONCURRENTLY` em prod — `CREATE INDEX` sem isso trava a tabela.
- Não testar em sandbox — índice errado pode piorar plano de execução.
- Index hint no código — PG não suporta; planner decide.

## Procedimento de aplicação

1. Gera SQL com [[migration-rollback]] preparado:
   ```sql
   -- Up
   CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_lancamentos_escola_venc
   ON lancamentos(escola_id, vencimento);

   -- Down
   DROP INDEX IF EXISTS idx_lancamentos_escola_venc;
   ```
2. Aplica via Management API ([[reference_lumied_supabase]]) — `CONCURRENTLY` é importante.
3. Registra em `schema_migrations`.
4. Re-roda EXPLAIN ANALYZE da query original — confirma `Index Scan` em vez de `Seq Scan`.
5. Monitora IO no Supabase Studio pelas próximas 24h.

## Casos específicos Lumied/Construfare

### Tabelas conhecidas com pressão de IO

- `lancamentos` (CR/CP) — Construfare e Lumied
- `movimentos_almox`
- `ponto_registros`
- `notificacoes_outbound`
- `health_checks` (alta cardinalidade temporal — partition por mês se crescer demais)

### Reconcile (Construfare)

[[project_sienge_reconcile]] + [[project_construfare_reconcile_coverage]] fazem matching pesado. Índices em:
- `contas_pagar(numero, fornecedor_id)`
- `contas_receber(numero, cliente_id)`
- chave de matching usada no map.

### Cross-check com [[n-plus-one]]

Query lenta nem sempre é falta de índice — pode ser N+1 query. Rodar [[n-plus-one]] junto.
