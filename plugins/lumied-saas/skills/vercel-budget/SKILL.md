---
name: vercel-budget
description: Verifica quantos deploys foram feitos no projeto Vercel hoje contra o cap do Free Tier (100/dia) e avisa se está próximo do limite. Use antes de push grande, ao planejar refactor multi-PR, ou quando o usuário disser "tô estourando o Vercel?". Materializa [[feedback_agrupar_commits]].
---

# Vercel deploy budget — Lumied

Contexto: [[feedback_agrupar_commits]] — Vercel Free Tier dá 100 deploys/dia. Lumied historicamente bate isso em dias de refactor intenso. Push agrupado é a regra; mas refactores ainda podem estourar mesmo com batching.

## Caps relevantes

| Plano | Deploys/dia | Build mins/mês | Bandwidth/mês |
|---|---|---|---|
| Free (atual Lumied) | **100** | 6000 | 100GB |
| Pro | ilimitado* | 24000 | 1TB |

\* Pro tem cap de ~6000 deploys/mês na prática.

Quando bate 100: deploys subsequentes ficam em fila ou são rejeitados. Vercel não cobra extra, simplesmente bloqueia.

## Como medir

### Opção A — via API Vercel

```bash
curl -s "https://api.vercel.com/v6/deployments?projectId=$VERCEL_PROJECT_ID&since=$(date -u +%Y-%m-%dT00:00:00Z -d '00:00 BRT')&limit=200" \
  -H "Authorization: Bearer $VERCEL_TOKEN" \
| jq '.deployments | length'
```

Variáveis necessárias:
- `VERCEL_TOKEN` — pessoal ou do projeto.
- `VERCEL_PROJECT_ID` — `prj_...`.

### Opção B — via `vercel` CLI

```bash
vercel ls --since=today | wc -l
```

(Subtrair 1 do header.)

### Opção C — dashboard manual

`vercel.com/<org>/<projeto>/deployments?filter=today` — última opção, mas funciona se token expirou.

## Limites de alerta

| Deploys hoje | Status | Ação |
|---|---|---|
| 0-50 | 🟢 ok | nenhuma |
| 51-79 | 🟡 atenção | sugerir agrupar commits restantes |
| 80-94 | 🟠 risco | bloquear push novo, exigir batch |
| 95-100 | 🔴 crítico | NÃO permitir deploy (a menos que urgência justifique) |
| 100+ | 🛑 fila | Vercel está rejeitando — esperar reset (00h UTC ≈ 21h BRT) |

## Output esperado

```
Vercel budget — 2026-05-14 (hoje)
  Deploys: 67/100 🟡
  Reset: em 4h12 (00h UTC)
  Recomendação: agrupar próximos commits antes de push
  Pico recente: 89 ontem (2026-05-13)
```

Se ≥80:
```
🟠 Vercel budget alto — 87/100
  Restam ~13 deploys até 00h UTC.
  Sugiro:
    - Adiar refactors não-críticos.
    - Agrupar commits pendentes em 1 push só.
    - Ver `git log --oneline origin/main..HEAD` pra ver quantos seriam.
```

## Procedimento

1. Tentar API Vercel primeiro (mais preciso).
2. Se token não disponível, pedir ao usuário pra rodar `vercel ls --since=today | wc -l` no terminal local (`!` prefix).
3. Calcular % do cap.
4. Reportar com cor + recomendação.
5. Se ≥95, oferecer:
   - Esperar reset.
   - Validar que o push pendente vale o risco.
   - Listar quais commits vão virar deploys (cada push = 1 deploy se Vercel auto-deploy estiver on, mais previews = mais deploys).

## Integração com outros skills

- `deploy-preflight` deve chamar esta skill no passo 8.
- Quando `lumied-deploy` for invocado direto, ele mesmo deve rodar este check antes de `git push`.

## Plano caso Free Tier não dê conta

Se Lumied bate 80/100 frequentemente, opções:
- Upgrade Pro ($20/mês) — análise de custo-benefício separada.
- Reduzir auto-deploy: configurar Vercel pra fazer deploy só na branch `main`, não em PRs.
- Mover preview deploys pra Cloudflare Pages.
- Disable Vercel Git integration e fazer deploy manual via CLI (1 deploy = 1 invocação).

## Quando NÃO bloquear

- Hotfix de segurança (sempre passa, mesmo no cap — Vercel costuma honrar).
- Deploy de docs que não dispara build (depende da config).
