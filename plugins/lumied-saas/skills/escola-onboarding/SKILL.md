---
name: escola-onboarding
description: Provisiona tenant novo (escola) end-to-end no Lumied — cria escola_id, módulos ativos, superadmin, RLS verificada, seed de catálogos. Materializa o handoff do GTM PLAYBOOK (assinatura) pra produto rodando. Use quando o usuário disser "nova escola assinou", "provisiona X", "onboarding da escola Y".
---

# Onboarding de nova escola (tenant)

Contexto: depois que a escola assina ([[project_saas_commercialization]] + [[project_tier_starter]]), há um conjunto de passos manuais hoje propenso a esquecimentos (RLS não habilitada, módulo não ativado, superadmin não criado, catálogos vazios). Pilotos atuais: [[project_escolas_piloto]] (Maple Bear Caxias + Bento Gonçalves). Esta skill roda o checklist consistentemente.

## Quando rodar

- Cliente assinou e foi pago.
- Implantação assistida começou.
- Migração de ERP ([[project_migracao_erps]]) vai começar — precisa do tenant existente primeiro.

NÃO usar pra:
- Demo/sandbox — usa `demo` tenant ou variante. Onboarding real cria dados que viram fiscais.
- Reativação de escola arquivada — fluxo é outro (não há skill ainda).

## Pré-requisitos

- Nome da escola, CNPJ, endereço, e-mail do superadmin (geralmente direção).
- Tier definido (Starter R$ 790 / Pro / Enterprise) — fonte: [[project_tier_starter]].
- Módulos contratados (acadêmico, financeiro, RH, almoxarifado, manutenção, ML/almox, ponto, comunicacao, etc.).
- Token Management API Lumied ([[reference_lumied_supabase]]).

## Procedimento

### 1. Criar registro em `escolas`

```sql
INSERT INTO escolas (id, nome, cnpj, tier, created_at)
VALUES (gen_random_uuid(), '<nome>', '<cnpj>', '<tier>', now())
RETURNING id;
```

Anota o `escola_id` retornado — é o tenant que vai aparecer em TODA tabela.

### 2. Ativar módulos contratados

Confere quais módulos o cliente contratou e ativa em `escolas_modulos` (ou tabela equivalente). Modelo da Demo Lumied (mig 284) tem tudo ativado — use como referência do que existe.

```sql
INSERT INTO escolas_modulos (escola_id, modulo, ativo)
VALUES
  ('<escola_id>', 'academico', true),
  ('<escola_id>', 'financeiro', true),
  -- ... só os contratados
;
```

### 3. Criar superadmin

```sql
-- Via supabase.auth.admin.createUser (não SQL direto na auth.users)
-- email: <e-mail do contato direção>
-- senha: gerada e enviada via Resend (não logar em chat)
```

Depois liga em `usuarios`:
```sql
INSERT INTO usuarios (id, escola_id, email, role)
VALUES ('<user_id_do_auth>', '<escola_id>', '<email>', 'superadmin');
```

`ivyson@gmail.com` sempre como segundo superadmin (regra [[user_ivyson]]).

### 4. Seed de catálogos mínimos

Dependendo dos módulos ativos:

- **Acadêmico:** turmas/séries vazias OK; importa via assistente quando dados do ERP chegarem.
- **Financeiro:** banco padrão? Plano de contas? Categorias? Pode esperar a implantação.
- **Almoxarifado:** zero itens é OK; cliente cadastra ou usa importador.
- **RH:** cargos básicos (Professor, Coordenador, Direção, Limpeza, Cozinha) já ajuda.
- **Manutenção:** categorias mínimas (Elétrica, Hidráulica, Mobiliário, TI).

### 5. Verificar RLS

Rode [[rls-check]] focado neste `escola_id`:

```sql
SELECT escola_id, count(*) FROM <cada_tabela_sensivel>
WHERE escola_id = '<escola_id>'
GROUP BY escola_id;
```

Resultado esperado: contagem zero ou positiva, **nunca NULL** ou contagem de outro tenant.

### 6. Smoke test

- Login com o superadmin recém-criado.
- Abre cada módulo ativo — confere se carrega vazio (sem erro de RLS).
- Tenta criar um registro de exemplo (uma turma, um item de almox) e deletar.
- Confere que outro tenant **não enxerga** esse registro.

### 7. Buckets de Storage (se aplicável)

Se módulos com PII (RH, acadêmico/fotos, manutenção/fotos) — seguir [[pii-bucket-audit]]:
- Bucket privado.
- Policy tenant-scoped (path com escola_id).

### 8. Comunicar cliente

- E-mail com link de acesso + senha temporária.
- Convite pra session de implantação.
- Link da Central de Ajuda ([[project_ajuda_pendente]]).

## Anti-padrões

- Criar escola sem ativar módulos contratados — usuário loga e vê tela vazia "sem módulos".
- Pular smoke test de RLS — replica o incidente [[project_tenant_isolation_incident]].
- Mandar senha do superadmin pelo WhatsApp em texto — usar Resend com link de "definir senha".
- Esquecer de adicionar `ivyson@gmail.com` como superadmin (regra [[user_ivyson]]).

## Pós-onboarding

- Salvar memória de projeto `project_<nome-escola>.md` se for piloto/relevante (similar a [[project_escolas_piloto]]).
- Se migração de ERP: pula pra [[erp-import-runbook]].
- Se tem cobranças bancárias: pula pra [[bank-homologacao]].
