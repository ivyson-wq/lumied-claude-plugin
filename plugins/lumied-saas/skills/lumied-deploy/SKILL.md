---
name: lumied-deploy
description: Roda o fluxo de deploy completo do Lumied/Construfare quando o usuário pede "deploy", "subir", "publicar" mudanças. Empacota git push (batched), supabase functions deploy, e aplicação de migrations via Management API com registro em schema_migrations. Use quando há mudanças locais commitadas ou em andamento que precisam ir pra produção.
---

# Lumied / Construfare deploy

Fluxo padrão que o Ivyson sempre quer ([[feedback_always_deploy]] + [[feedback_agrupar_commits]]):

1. **Commit local múltiplos, push único** — Vercel Free Tier limita a 100 deploys/dia. Nunca pushar commit-a-commit.
2. **Edge functions** depois do push se houve mudança em `supabase/functions/`.
3. **Migrations** via Management API (pooler PG bloqueado — [[reference_lumied_supabase]] / [[reference_construfare_supabase]]). Registrar em `schema_migrations` manualmente.

## Passos

### 1. Decidir escopo

Pergunte se o deploy é Lumied (`maple-bear-rs/`), Construfare (`construfare/`) ou Insta-Publisher (`insta-publisher/`). Se já está claro pelo cwd, não pergunte.

### 2. Checar estado

```bash
git status
git log origin/main..HEAD --oneline
ls supabase/migrations/ | tail -5   # migrations não aplicadas?
```

### 3. Push

Se há vários commits locais:
```bash
git push origin main
```

Antes de pushar, alerte se já há muitos deploys hoje (o hook `pre-push-vercel-budget` deve detectar; se não estiver ativo, peça pra rodar `npx vercel ls --token=$VERCEL_TOKEN | head -20`).

### 4. Edge functions

Só se mudou `supabase/functions/<nome>/`:
```bash
supabase functions deploy <nome> --project-ref <ref>
```

Refs conhecidos:
- Lumied: `brgorknbrjlfwvrrlwxj`
- Construfare: token em [[reference_construfare_supabase]]

### 5. Migrations via Management API

Para cada migration nova (não está em `schema_migrations` ainda):

```bash
SUPABASE_PROJECT_REF=brgorknbrjlfwvrrlwxj
SQL_FILE=supabase/migrations/331_xxx.sql
SQL=$(cat "$SQL_FILE")

curl --ssl-no-revoke -s -X POST \
  "https://api.supabase.com/v1/projects/$SUPABASE_PROJECT_REF/database/query" \
  -H "Authorization: Bearer $SUPABASE_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d "$(jq -nc --arg q "$SQL" '{query:$q}')"
```

Depois registre em `schema_migrations`:
```sql
INSERT INTO schema_migrations (version) VALUES ('331');
```

### 6. Verificar

- `gh run list -L 3` se a CI tá passando ([[project_gh_actions_tier]]).
- Abrir URL prod e fazer smoke test rápido se foi mudança visível ([[feedback_limitacoes_visiveis]]).

## Cuidados

- **Tenant isolation**: se tocou edge function ou SQL novo, rode a skill `tenant-audit` antes do deploy ([[project_tenant_isolation_incident]]).
- **Não --no-verify**, não --force-push em main.
- Se a migration falhar (Management API outage como em [[project_simpax_inventory_pendente]]), pare e avise; não tente várias vezes.
