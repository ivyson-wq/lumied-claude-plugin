---
name: cost-audit
description: Auditoria de custos do Lumied/Construfare nos tiers Free — Vercel deploys/dia, Supabase IO+Storage+egress, Cloudflare Workers requests/dia, GitHub Actions minutes/mês. Identifica quem está próximo do cap e propõe mitigação concreta. Use periodicamente, antes de refactor grande, ou quando o usuário disser "tô estourando alguma coisa?".
---

# Auditoria de custos — Lumied / Construfare

Contexto: operamos quase tudo em Free Tier. Já estouramos antes:
- [[project_lumied_io_storage]] — IO + Storage do Supabase chegaram no cap.
- [[project_gh_actions_tier]] — 2000/2000 min em 7 dias.
- [[feedback_agrupar_commits]] — Vercel Free permite 100 deploys/dia.
- [[project_private_after_reset]] — Repo virou público pra esquivar cap.

Esta skill verifica todos os caps de uma vez, dá número absoluto + % do cap + tendência.

## Quando rodar

- Antes de planejar refactor multi-PR ([[project_refator_lumied]] tem 6 ondas).
- Quando usuário pergunta "tô bem de cota?", "vou estourar X?".
- Mensalmente (preventivo).
- Quando uma skill específica ([[vercel-budget]]) já apontou alerta — esta skill cruza com os outros provedores.

## Provedores e onde checar

### 1. Vercel (100 deploys/dia Free)

Skill dedicada: [[vercel-budget]]. Cap reseta a cada 24h rolling. Projetos a checar:
- Lumied principal
- Insta Publisher ([[project_insta_publisher]])
- Construfare painel reconcile ([[project_construfare_reconcile_coverage]])
- Outros workers/dashboards

### 2. Supabase Lumied (Free: 500MB DB + 1GB Storage + 5GB egress + 2M edge fn invocations/mês)

Verifica via Management API ([[reference_lumied_supabase]]):
```bash
curl -H "Authorization: Bearer $TOKEN" \
  https://api.supabase.com/v1/projects/$PROJECT_REF/usage
```

Métricas a olhar:
- `db_size_bytes` vs 500MB
- `storage_size_bytes` vs 1GB
- `egress_bytes` mês corrente vs 5GB
- `edge_function_invocations` mês vs 2M
- IOPS — limite efetivo Free é baixo, gargalo real ([[project_lumied_io_storage]])

### 3. Supabase Construfare (mesmo cap)

[[reference_construfare_supabase]]. Reconcile diário ([[project_sienge_reconcile]]) pode gerar volume — checar.

### 4. Cloudflare Workers (100k requests/dia Free)

Workers conhecidos no projeto:
- Lumied Bridge ([[project_lumied_bridge]])
- Reval proxy ([[project_reval_proxy]])
- Outros relays/workers do bank-multiprovider ([[project_banks_multiprovider]])

Check via `wrangler analytics` ou dashboard CF.

### 5. GitHub Actions (2000 min/mês Free repo privado)

Skill relacionada: [[project_gh_actions_tier]] — já fizemos as mitigações principais (CI consolidado, paths-ignore, outbound-pulse migrado pra pg_cron mig 329).

Repo Lumied está **público** ([[project_private_after_reset]] — voltar privado ~01/06/2026), e Actions em repo público é **ilimitado**. Mas Construfare/outros podem ainda contar.

### 6. Resend (3000 e-mails/mês Free)

Usado em outbound-pulse, SDR agent ([[project_sdr_agent]]), notificações tenant. Cap pode apertar quando crescer base.

### 7. ML / Resend / outros tokens externos

Sem cap monetário direto, mas rate limit e validade — cross-check com [[project_ml_oauth_setup]] (refresh token rotation) e [[secret-rotation]].

## Procedimento

1. Pra cada provedor → buscar uso atual + cap.
2. Calcular % e tendência (vs semana passada se tiver histórico).
3. Reportar em tabela:

   ```
   PROVIDER           USAGE          CAP        %      STATUS    AÇÃO
   Vercel (Lumied)    47/100/d       100/d      47%    🟢 OK     -
   Supabase Lumied    420MB/500MB    500MB      84%    🟠 ALTO   purge mig X, ver project_lumied_io_storage
   GH Actions         1850/2000/m    2000/m     93%    🔴 RISCO  acelerar voltar privado, ver project_gh_actions_tier
   CF Workers         8k/100k/d      100k/d     8%     🟢 OK     -
   Resend             1200/3000/m    3000/m     40%    🟢 OK     -
   ```

4. Pra cada 🟠 ou 🔴, propor mitigação concreta:
   - Não "considerar otimizar" — sim "migrar tabela X pra storage", "purgar logs antigos mig Y", "agrupar commits ([[feedback_agrupar_commits]])".

## Mitigações conhecidas por provedor

- **Vercel deploys:** [[feedback_agrupar_commits]] — push único por dia, paths-ignore no GH Actions.
- **Supabase IO/Storage:** migs de purge ([[project_lumied_io_storage]] — 304/305), mover logs frios pra storage, pg_cron de cleanup.
- **CF Workers:** cache KV ([[project_reval_proxy]] padrão), throttle no entry point.
- **GH Actions:** repo público temporário ([[project_private_after_reset]]), migrar workflows recorrentes pra pg_cron como fizemos com outbound-pulse (mig 329).
- **Resend:** consolidar e-mails diários em batch, dedupe ([[feedback_alert_cadence]] 96h).

## Anti-padrões

- Reportar % sem ação — usuário precisa saber **o que fazer**.
- Bater em cap sem alerta prévio (precisa rodar essa skill preventivamente).
- Otimização micro quando o problema é macro (purgar 1MB quando precisa migrar 200MB).
- Esquecer caps que não custam $ mas custam disponibilidade (rate limit ML, refresh token expirado).

## Output esperado

Resumo de 1 parágrafo + tabela. Se algum 🔴, abrir como prioridade no topo. Se tudo 🟢, dizer "tudo dentro do cap, próxima auditoria em ~30 dias".
