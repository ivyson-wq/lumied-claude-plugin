---
name: bank-homologacao
description: Checklist de homologação por banco brasileiro (Inter/Sicredi/BB/Itaú/Bradesco e além) após adapter scaffold — gera/concilia boleto/PIX, valida webhook de baixa, testa retorno CNAB, prova rollback. Materializa o gap entre "adapter deployado" e "banco aprovado em prod". Use após [[bank-adapter-template]] criar o scaffold, ou quando o usuário disser "homologa o banco X".
---

# Homologação por banco — Lumied multi-provider

Contexto: temos 5 adapters deployados (Inter, Sicredi, BB, Itaú, Bradesco — [[project_banks_multiprovider]]). Adapter deployado ≠ banco aprovado em prod. Cada banco tem fluxo de homologação próprio (ambiente sandbox, certificado, faixa de números, callback). Esta skill garante que nenhum step crítico fica de fora antes de virar a chave pra produção.

## Quando rodar

- Adapter novo recém-deployado ([[bank-adapter-template]] terminou).
- Banco mudou versão de API / certificado expirou.
- Antes de migrar uma escola de banco antigo pra novo.
- Auditoria periódica anual (certificados, callbacks ainda batendo).

## Checklist mestre (todos os bancos)

### Credenciais & ambiente

- [ ] Certificado mTLS (se aplicável: Inter, Itaú, Bradesco) válido e com data de expiração mapeada.
- [ ] Secret no Supabase (não no código) — cross-check com [[secrets-scan]].
- [ ] Ambiente sandbox configurado separado de prod.
- [ ] Faixa de nosso número / conta convênio definida.
- [ ] Variáveis de ambiente no edge function alinhadas com [[reference_lumied_supabase]].

### Emissão

- [ ] Boleto bancário gerado: PDF válido, código de barras lê, linha digitável correta.
- [ ] PIX (cobrança imediata `cob`): QR Code lê, BR Code copia/cola funciona, txid único.
- [ ] PIX (cobrança com vencimento `cobv`) se contratado.
- [ ] Multa + juros após vencimento aplicados conforme regra da escola.
- [ ] Desconto pré-vencimento se contratado.

### Recebimento / baixa

- [ ] Webhook configurado no banco apontando pra edge function correta.
- [ ] Webhook autentica (HMAC, IP whitelist, token, mTLS — depende do banco).
- [ ] Edge function de webhook valida assinatura ANTES de baixar — sem isso é spoof livre.
- [ ] Baixa atualiza `contas_receber` com `pago_em`, `valor_pago`, `forma_pagamento`.
- [ ] Idempotência: webhook duplicado não baixa duas vezes.
- [ ] Tenant isolation: webhook valida `escola_id` ([[tenant-audit]] obrigatório).

### Conciliação

- [ ] Retorno CNAB 240/400 (se banco usa) parseado corretamente.
- [ ] Cross-check com [[project_construfare_reconcile_coverage]] padrão de matching.
- [ ] Discrepância < 1% entre emitido vs baixado em sandbox.

### Cancelamento / baixa manual

- [ ] Cancelar boleto antes do vencimento funciona.
- [ ] Após vencimento, cancelar funciona ou retorna erro tratado.
- [ ] Baixa manual (sem webhook, ex: pago no caixa) reflete corretamente.

### Limites & rate

- [ ] Rate limit da API mapeado (req/s, req/min, req/dia).
- [ ] Comportamento ao bater rate (retry, queue, fallback).
- [ ] Tamanho máximo de batch (alguns bancos limitam emissão em lote).

### Compliance / LGPD

- [ ] PII enviado pro banco está minimizado (só o necessário).
- [ ] Logs do banco não vazam dados sensíveis em texto claro.
- [ ] Política de retenção de dados de cobrança definida.

## Checklist específico por banco

### Inter
- [ ] OAuth 2.0 client_credentials configurado (token expira em 1h, refresh automático).
- [ ] Certificado .crt + .key salvos como secret.
- [ ] Webhook URL HTTPS válida e respondendo 200 em <5s.
- [ ] Conta de cobrança ativada no Internet Banking PJ.

### Sicredi
- [ ] Convênio cobrança gerado e número de convênio salvo.
- [ ] Header `x-api-key` + cert mTLS.
- [ ] Carteira (109 ou outra) confirmada com gerente.

### Banco do Brasil
- [ ] Aplicação no BB Developers aprovada (pode demorar dias).
- [ ] Convênio BB-Cobrança ativo na conta da escola.
- [ ] PIX-Cob com chave registrada.

### Itaú
- [ ] OAuth + mTLS (dois cofres: app + cert).
- [ ] Carteira 109/175 conforme contrato.
- [ ] Endpoint `cash-management/v2` vs `cobranca-boletos/v1` claros no adapter.

### Bradesco
- [ ] Certificado A1/A3 válido.
- [ ] Webhook → endpoint específico, não genérico (Bradesco fechou geral em 2024).
- [ ] Faixa nosso número não conflita com outras escolas no mesmo CNPJ.

## Procedimento (rodar pra cada banco)

1. Pega o banco e a escola alvo.
2. Roda o checklist mestre.
3. Roda o checklist específico do banco.
4. Pra cada item ❌, documenta o que falta e quem resolve (cliente vs nós vs banco).
5. Output:
   ```
   Banco: Sicredi
   Escola: Maple Bear Caxias
   Status geral: 🟡 PARCIAL (12/18 OK)

   Bloqueios:
   - Carteira ainda não confirmada com gerente (ação: cliente)
   - Webhook não está validando assinatura (ação: nós — fix em supabase/functions/bank-sicredi-webhook)

   Próximos passos:
   1. ...
   ```

## Anti-padrões

- Pular validação de assinatura no webhook — spoof livre, pode causar baixa indevida.
- Esquecer idempotência — webhook reenviado baixa duas vezes (cliente vê dívida sumir e voltar).
- Misturar sandbox com prod no mesmo edge function sem flag clara.
- Salvar certificado no repo (mesmo público) — sempre como secret. Roda [[secrets-scan]] depois.
- Homologar sem [[migration-rollback]] preparado — adapter em prod sem plano de reverter.

## Pós-homologação

- Memória de projeto `project_banco_<nome>_<escola>.md` se for primeira ativação do banco.
- Cross-check com [[deploy-preflight]] antes de virar pra prod.
- Adicionar entry no painel admin de status de bancos (se existir).
