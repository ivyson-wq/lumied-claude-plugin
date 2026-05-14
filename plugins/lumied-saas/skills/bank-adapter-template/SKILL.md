---
name: bank-adapter-template
description: Gera scaffold pra novo adapter de banco brasileiro (Inter, Sicredi, BB, Itaú, Bradesco e além) — config + edge function + homologação checklist. Use quando começar um novo banco no [[project_banks_multiprovider]].
---

# Bank adapter template

Contexto: [[project_banks_multiprovider]] — 5 adapters deployados; falta `bank-relay v2` + homologação por banco. Cada banco brasileiro tem peculiaridades de mTLS, certificado, escopo OAuth, formato de boleto/PIX, webhook de retorno.

## Inputs necessários

Pergunte (se não estiver claro):
1. Qual banco? (inter | sicredi | bb | itau | bradesco | santander | safra | …)
2. Que operações? (boleto emissão | boleto consulta | PIX QR estático | PIX QR dinâmico | extrato OFX | TED/transferência)
3. Tem certificado mTLS? (sim/não — define onde armazenar)
4. Sandbox antes de produção? (recomendado: sim)

## Estrutura gerada

### 1. Config

`supabase/migrations/NNN_bank_<nome>_config.sql`:
```sql
-- Schema padrão escola_banco_config (já existe — mig 328)
-- Inserir/atualizar tipo:
INSERT INTO escola_banco_config_tipos (codigo, nome, requer_mtls, requer_oauth, scopes)
VALUES ('<nome>', '<Nome humano>', true|false, true|false, ARRAY['boleto','pix']);
```

### 2. Adapter

`supabase/functions/bank-<nome>-adapter/index.ts`:
```ts
// Padrão dos adapters existentes (espelhe inter-adapter ou sicredi-adapter)
//
// Exports:
//   - emitirBoleto(escolaId, dados): Promise<BoletoResp>
//   - consultarBoleto(escolaId, nossoNumero): Promise<BoletoStatus>
//   - emitirPixQr(escolaId, dados): Promise<PixResp>
//   - listarMovimentos(escolaId, dataIni, dataFim): Promise<Movimento[]>
//
// Cada função:
//   1. Carrega config da escola via escola_banco_config (filtrado por escola_id do JWT — [[tenant-audit]])
//   2. Resolve token OAuth (cache em escola_banco_token, refresh se expirado)
//   3. Chama API do banco com mTLS se aplicável
//   4. Loga em bank_request_log (escola_id, operacao, http_status, elapsed_ms, request_id)
//   5. Retorna resposta normalizada
```

### 3. bank-relay v2

Atualizar `supabase/functions/bank-relay/index.ts`:
```ts
// Roteador: { banco, operacao, params } → adapter correto
// case '<nome>': return await import('../bank-<nome>-adapter/index.ts').then(m => m.<operacao>(escolaId, params))
```

### 4. Webhook (se o banco notifica)

`supabase/functions/bank-<nome>-webhook/index.ts`:
- Validar assinatura/HMAC do banco antes de aceitar
- Resolver `escola_id` pelo header de identificação ou pelo `nosso_numero`
- Atualizar `contas_receber` correspondente
- Logar em `bank_webhook_log`

## Checklist de homologação por banco

Documentar em `docs/bancos/<nome>-homologacao.md`:

- [ ] Cadastro do app no portal do banco
- [ ] Certificado mTLS gerado e instalado (Supabase Vault ou env var, NUNCA commitado)
- [ ] OAuth client_id/secret nos secrets da edge function
- [ ] Sandbox: emitir 1 boleto teste, consultar status, simular pagamento
- [ ] Sandbox: emitir 1 PIX QR, simular pagamento
- [ ] Sandbox: ler 30 dias de movimentação
- [ ] Webhook: receber 1 evento de teste
- [ ] Produção: limites diários definidos (proteção contra erro multiplicador)
- [ ] Reconcile: linhas geradas batem com `bank_request_log`

## Cuidados

- **Certificados/secrets**: Vault Supabase ou `supabase secrets set`. Nunca em código.
- **Tenant**: `escola_id` SEMPRE do JWT, nunca do body ([[project_tenant_isolation_incident]]).
- **Limites**: cada banco tem rate-limit; respeite e cacheie quando possível.
- **Idempotência**: emissão de boleto deve ser idempotente por `(escola_id, ref_externa)` pra cliente reenviar sem duplicar.
