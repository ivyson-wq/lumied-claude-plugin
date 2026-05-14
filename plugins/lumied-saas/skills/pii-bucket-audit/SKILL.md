---
name: pii-bucket-audit
description: Audita buckets Supabase Storage do Lumied para garantir que buckets com PII (fotos de alunos, docs RH, comprovantes, contratos, atestados) são privados + têm policies de acesso tenant-scoped. Extensão das migs 278-282. Use após criar bucket novo, ou periodicamente para confirmar drift.
---

# PII bucket audit — Lumied

Contexto: [[project_backlog_v2_pendencias]] migs 278-282 fecharam o gap inicial tornando buckets PII privados. Buckets novos ou criados manualmente via Studio podem nascer públicos e vazar.

## Buckets críticos (PII) — devem ser PRIVADOS

| Bucket | Conteúdo | Severidade |
|---|---|---|
| `alunos-fotos` | Fotos de menores | 🔴 LGPD |
| `rh-docs` | RG, CPF, contratos funcionários | 🔴 LGPD |
| `atestados` | Atestados médicos | 🔴 LGPD/sigilo médico |
| `comprovantes` | Comprovantes de pagamento | 🟡 PII financeira |
| `contratos` | Contratos pais/escola | 🟡 PII financeira |
| `boletins` | Notas, frequência, observações | 🔴 LGPD menores |
| `face-id` | Templates biométricos | 🔴 LGPD biométrico |
| `manut-anexos` | Fotos de manutenção (podem ter pessoas) | 🟡 |

## Buckets públicos aceitáveis

| Bucket | Conteúdo |
|---|---|
| `logos` | Logos das escolas (público intencional) |
| `marketing-static` | Imagens do site/landing |
| `cardapio-fotos` | Fotos de cardápio (sem pessoas) |

## Como auditar

### 1. Listar buckets e sua visibilidade

```sql
SELECT id, name, public, file_size_limit, allowed_mime_types
FROM storage.buckets
ORDER BY name;
```

Qualquer linha onde `public=true` em bucket da lista PII → 🔴 ALERTA.

### 2. Verificar policies de `storage.objects`

Cada bucket privado precisa de policy que filtre por tenant. Padrão Lumied:

```sql
CREATE POLICY "tenant_isolation_select" ON storage.objects FOR SELECT
USING (
  bucket_id = 'alunos-fotos'
  AND (storage.foldername(name))[1] = (current_setting('request.jwt.claims', true)::json->>'escola_id')
);
```

Convenção: primeiro segmento do path = `escola_id`. Ex: `<escola_id>/<aluno_id>/avatar.jpg`.

### 3. Conferir URLs públicas em uso

Grep no código por `getPublicUrl(` apontando para bucket privado — bug: a URL gerada não vai funcionar OU vai funcionar (se bucket virou público acidentalmente).

```bash
# certo pra bucket privado:
supabase.storage.from('alunos-fotos').createSignedUrl(path, 3600)

# errado:
supabase.storage.from('alunos-fotos').getPublicUrl(path)
```

### 4. CORS / hotlink

Buckets públicos legítimos (logos, marketing) devem ter CORS restritivo ao domínio Lumied + caches longos. Buckets privados não usam CORS público.

## Procedimento

1. Executar query do passo 1 via Management API.
2. Para cada bucket marcado público, verificar se está na whitelist.
3. Para cada bucket privado, listar policies de `storage.objects` filtradas por `bucket_id`.
4. Grep no código por uso incorreto (`getPublicUrl` em bucket PII).
5. Reporte:
   ```
   🔴 bucket `comprovantes`         public=true mas é PII financeira → tornar privado
   🟢 bucket `alunos-fotos`         privado + 4 policies tenant-scoped OK
   🟡 bucket `face-id`              privado mas SEM policy de UPDATE (qualquer um do tenant pode sobrescrever)
   🔴 código `pais.html:142`        usa getPublicUrl('boletins') — vai vazar se bucket ficar público
   ```
6. Para 🔴: gerar SQL de fix imediato:
   ```sql
   UPDATE storage.buckets SET public = false WHERE id = 'comprovantes';
   -- + criar policies de SELECT/INSERT/UPDATE/DELETE tenant-scoped
   ```

## Quando rodar

- Após criar bucket via migration ou Studio.
- Trimestralmente como audit.
- Antes de auditoria LGPD externa.
- Quando o usuário falar "audita storage", "tem foto vazando?".
