---
name: idempotency-check
description: Audita endpoints e jobs do Lumied que precisam ser idempotentes — webhooks de banco, importadores ERP, reconcile, cron de outbound, e-mails transacionais. Reenvio duplicado não pode baixar pagamento 2x, importar aluno 2x, ou enviar boleto 2x. Use em PR review de webhooks/jobs, após [[bank-homologacao]], ou quando o usuário disser "se rodar de novo quebra?".
---

# Idempotência — Lumied / Construfare

Contexto: vários pontos do Lumied recebem chamadas que podem repetir:
- Webhooks de banco ([[bank-homologacao]]) — banco reenvia se não receber 200.
- Importadores ERP ([[erp-import-runbook]]) — usuário pode disparar 2x sem perceber.
- Cron jobs ([[cron-health]]) — pg_cron pode disparar duplicado em raro caso de falha de lock.
- Reconcile Sienge ([[project_sienge_reconcile]]) — trigger remoto + cron interno coexistem.
- E-mails / push ([[feedback_alert_cadence]] tem dedupe 96h).

Sem idempotência, chamada repetida = dado errado. Idempotência é "rodar 2x ≡ rodar 1x" pra esse efeito.

## Quando rodar

- PR review de `supabase/functions/**` que recebe webhook ou cria efeitos colaterais.
- Após [[bank-homologacao]] (webhooks de banco são alvo prioritário).
- Antes de habilitar trigger remoto novo / cron novo.
- Investigando bug "registro duplicou", "pagou 2x", "enviou 2 e-mails".

## Padrões de idempotência

### 1. Chave de idempotência explícita (header)

Cliente manda `Idempotency-Key: <uuid>`. Server guarda chave + resultado em tabela:

```sql
CREATE TABLE idempotency_keys (
  key text PRIMARY KEY,
  escola_id uuid REFERENCES escolas(id),
  endpoint text NOT NULL,
  response_body jsonb,
  response_status int,
  criado_em timestamptz DEFAULT now()
);
```

Server:
- Recebe request → `SELECT FROM idempotency_keys WHERE key = X`.
- Se existe: retorna `response_body` + `response_status` (sem reexecutar).
- Se não: executa, salva resultado, retorna.

Padrão Stripe. Útil pra operações iniciadas pelo cliente (botão "Pagar agora" que pode ser clicado 2x).

### 2. Identificador natural do evento (webhook)

Banco manda webhook com `event_id` ou `transaction_id` ou similar. Server checa se já processou:

```sql
CREATE TABLE bank_webhook_events (
  id text PRIMARY KEY,         -- ID do evento no banco
  banco text NOT NULL,
  escola_id uuid NOT NULL,
  payload jsonb,
  processado_em timestamptz DEFAULT now()
);
```

Edge function:
```ts
const eventId = body.event_id || body.transaction_id;
if (!eventId) return new Response('event_id missing', { status: 400 });

const { error } = await supabase
  .from('bank_webhook_events')
  .insert({ id: eventId, banco: 'inter', escola_id: escolaId, payload: body });

if (error?.code === '23505') {
  // já processado — retorna 200 sem refazer
  return new Response('already processed', { status: 200 });
}

// processa pela primeira vez
await processarBaixa(body);
return new Response('ok', { status: 200 });
```

Cross-check [[edge-fn-authz]] — validar assinatura **antes** desse check.

### 3. Estado-target (set, não delta)

Em vez de "incrementar saldo em X" (delta), fazer "definir saldo como Y" (set). Repetir = mesmo resultado.

```ts
// SUSPEITO — não idempotente
await supabase.rpc('adicionar_saldo', { conta_id, valor: 100 });
// Se chamar 2x, saldo +200.

// MELHOR
await supabase
  .from('contas')
  .update({ saldo: 1500, atualizado_em: now })
  .eq('id', conta_id)
  .eq('versao', versaoAtual); // optimistic locking
// Mesmo se chamar 2x, saldo final = 1500.
```

### 4. UPSERT com PK natural

Importadores ERP devem usar `upsert` com chave natural (RA do aluno, CPF, número do lançamento), não INSERT cego:

```ts
// SUSPEITO
await supabase.from('alunos').insert({ ra, nome, ... });
// Re-rodar: viola UNIQUE ou cria duplicado.

// MELHOR
await supabase.from('alunos').upsert(
  { ra, nome, escola_id, ... },
  { onConflict: 'ra,escola_id' }
);
// Re-rodar: atualiza linha existente.
```

### 5. Lock pessimista pra job exclusivo

Cron que não pode rodar 2x em paralelo:

```sql
-- pg_cron já garante 1 instância por job, mas se há trigger remoto + cron:
SELECT pg_try_advisory_lock(1234); -- 1234 = ID arbitrário do job
-- Se retorna false, outro processo está rodando — sair.
```

Ou usar `SELECT ... FOR UPDATE SKIP LOCKED` pra processar fila.

## Procedimento de auditoria

### 1. Listar endpoints/jobs de risco

```bash
grep -rE "POST|webhook|cron|trigger" supabase/functions/ --include=index.ts | head -30
```

Identificar:
- Webhooks externos (banco, Meta, Resend, ML).
- Endpoints que criam/mutam dado.
- Cron jobs (cross-check com [[cron-health]]).

### 2. Pra cada um, verificar:

- [ ] Há identificador único pra evento/ação?
- [ ] Há check antes de processar?
- [ ] Operação é "set" ou "delta"?
- [ ] Há lock pra job concorrente?
- [ ] Se falhar no meio, é seguro re-rodar?

### 3. Classificar

- **🔴 Não idempotente** — pode causar dado duplicado/dobrado.
- **🟠 Parcial** — protege contra alguns casos mas não todos (ex: usa upsert mas dispara e-mail mesmo se já enviou).
- **🟢 OK** — idempotência garantida.

### 4. Output

```
## Idempotência audit — supabase/functions/

🔴 Bloqueios:

1. bank_sicredi_webhook/index.ts
   - Sem check de event_id do banco
   - Reenvio → baixa pagamento 2x
   - Fix: criar tabela bank_webhook_events + insert com UNIQUE

2. enviar_cobranca/index.ts
   - Não tem dedupe de e-mail
   - Cliente clica "Enviar" 2x → 2 e-mails
   - Fix: salvar em outbound_log antes de enviar (UNIQUE em (cobranca_id, tipo))

🟠 Parcial:

1. importer_sponte/index.ts
   - Alunos via UPSERT ✓
   - Mas histórico financeiro via INSERT puro ✘
   - Fix: UPSERT com (numero, escola_id) como conflict

🟢 OK:

- reconcile_sienge — usa map em memória + UPSERT em fila
- backup_escolas — single-shot por dia, lock via pg_cron
- ml_oauth_refresh — set token (não delta)
```

## Anti-padrões

- "Confio no cliente não chamar 2x" — cliente sempre chama 2x (usuário, retry, bug).
- Retornar 500 quando detecta duplicate em vez de 200 — banco vai retentar pra sempre.
- Idempotency-key vazio aceito como válido — chave única pra chamada vazia = ataque óbvio.
- TTL do idempotency_keys infinito — tabela cresce sem parar. Adicionar TTL (ex: 7 dias) + cron de purge.
- Lock sem timeout — se processo trava, ninguém mais roda.
- Salvar resultado **depois** de processar (race: outro request chega no meio). Salvar **antes** com status "processing", depois atualizar.

## Casos específicos Lumied/Construfare

### Webhooks de banco ([[bank-homologacao]])

Todos os 5 adapters ([[project_banks_multiprovider]]) precisam:
- Validar assinatura ([[edge-fn-authz]]).
- Salvar event_id em `bank_webhook_events` antes de processar.
- Idempotência cobrir baixa + atualização de status do boleto.

### Importadores ERP ([[erp-import-runbook]])

- Cliente pode iniciar import 2x no mesmo dia.
- UPSERT com chave natural por entidade.
- Tabela de controle `import_runs` com status `running | done | failed` pra evitar 2 imports concorrentes.

### Reconcile ([[project_sienge_reconcile]])

- Trigger remoto + pg_cron interno — risco de overlap.
- Memory: **NÃO disparar migrate manualmente** ([[project_sienge_reconcile]]).
- Adicionar advisory lock no início do job.

### Outbound (e-mails / push) ([[feedback_alert_cadence]])

- Dedupe 96h já é padrão.
- Tabela `outbound_log` com UNIQUE em (destinatario, tipo, periodo).
- E-mail transacional (boleto) tem dedupe próprio (cobranca_id + tipo).

### Manutenção tirar dúvida ([[project_manut_tirar_duvida]])

- Resposta pode ser submetida 2x. UPSERT com (chamado_id, autor_id, criado_em truncado).

## Pós-audit

- Pra cada 🔴, criar mig com tabela de idempotência + index + RLS.
- [[migration-rollback]] preparado.
- Adicionar teste ([[test-coverage]] Tier 🔴) — webhook duplicado retorna 200 sem efeito colateral.
- Se já houve incidente de dado duplicado: [[postmortem]] referenciando esta skill como ação estrutural.
