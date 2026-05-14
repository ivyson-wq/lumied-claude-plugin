---
name: erp-importer-template
description: Gera scaffold pra importador de novo ERP (Escolaweb, Sponte, WPensar, Sophia, TOTVS, GVDasa) — migration + edge function + UI admin de importação. Use quando começar um novo importer dos 7 sprints de migração de ERPs ([[project_migracao_erps]]).
---

# ERP Importer template

Contexto: [[project_migracao_erps]] — 7 ERPs pra suportar migração assistida e paga, com **histórico financeiro completo** (não só cadastros). Esta skill padroniza o boilerplate.

## Inputs necessários

Pergunte (se não estiver claro):
1. Qual ERP? (escolaweb | sponte | wpensar | sophia | totvs | gvdasa)
2. Formato de entrada? (CSV export | API REST | dump SQL | XLS planilha)
3. Escopo: cadastros + financeiro? só cadastros?

## Estrutura gerada

### 1. Migration

`supabase/migrations/NNN_erp_<nome>_import.sql`:
```sql
-- Staging tables (uma por entidade do ERP, sem RLS apertada — só admin)
CREATE TABLE IF NOT EXISTS erp_<nome>_alunos_raw (
  id bigserial PRIMARY KEY,
  escola_id uuid NOT NULL REFERENCES escolas(id),
  external_id text NOT NULL,
  payload jsonb NOT NULL,
  imported_at timestamptz DEFAULT now(),
  status text DEFAULT 'pending' CHECK (status IN ('pending','mapped','failed','skipped')),
  error text,
  mapped_aluno_id uuid REFERENCES alunos(id),
  UNIQUE (escola_id, external_id)
);

-- Tabela de mapeamento de IDs (importante pra reimportar incrementalmente)
CREATE TABLE IF NOT EXISTS erp_<nome>_id_map (
  escola_id uuid NOT NULL REFERENCES escolas(id),
  entity text NOT NULL,                     -- 'aluno' | 'turma' | 'titulo_cp' etc
  external_id text NOT NULL,
  internal_id uuid NOT NULL,
  imported_at timestamptz DEFAULT now(),
  PRIMARY KEY (escola_id, entity, external_id)
);

-- RLS — só service role acessa
ALTER TABLE erp_<nome>_alunos_raw ENABLE ROW LEVEL SECURITY;
ALTER TABLE erp_<nome>_id_map ENABLE ROW LEVEL SECURITY;
```

### 2. Edge function `erp-<nome>-import`

`supabase/functions/erp-<nome>-import/index.ts`:
```ts
// Recebe arquivo (CSV/XLS) ou trigger de cron
// 1. Parse → erp_<nome>_*_raw com status='pending'
// 2. Validate (campos obrigatórios, FKs do ERP de origem)
// 3. Map → tabelas reais do Lumied (alunos, turmas, contas_pagar, etc) com escola_id forçado do JWT
// 4. Atualiza erp_<nome>_id_map
// 5. Marca status='mapped' ou 'failed'+error

// CRÍTICO ([[project_tenant_isolation_incident]]):
// - escola_id SEMPRE vem do JWT, nunca do body do CSV
// - se o CSV tem escola_id diferente, REJEITAR (não silenciar)
```

### 3. UI admin

`admin-erp-<nome>.html` (espelha `admin-baixas-csv.html` do Construfare):
- Upload do arquivo
- Preview das primeiras 20 linhas com mapping
- Botão "importar staging"
- Tabela com status: pending | mapped | failed | skipped
- Botão "promover mapped → produção"
- Filtro por entity + status
- Export CSV de erros pra cliente corrigir e reimportar

### 4. Histórico financeiro

Se o escopo inclui financeiro, adicionar:
- `erp_<nome>_contas_pagar_raw` + `erp_<nome>_contas_receber_raw`
- Map pra `contas_pagar` / `contas_receber` preservando `data_vencimento`, `valor_original`, `data_pagamento`, `valor_pago`
- Flag `_origem='erp_<nome>'` na linha mapeada

## Convenções dos ERPs

- **Escolaweb / Sponte**: CSVs separados por entidade, encoding latin1 frequente.
- **WPensar / Sophia**: API REST com paginação; precisa rate-limit.
- **TOTVS / GVDasa**: dump SQL ou XLS gigante; processar em chunks.

## Anti-padrões

- Não importar direto pra tabela de produção (sempre staging primeiro).
- Não dropar staging depois — mantém pra auditoria por pelo menos 90d.
- Não confiar no `external_id` ser único globalmente — sempre escope por `escola_id`.
