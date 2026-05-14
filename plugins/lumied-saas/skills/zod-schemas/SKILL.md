---
name: zod-schemas
description: Aplica validação de input com Zod (ou similar) em edge functions Supabase do Lumied — antes de tocar banco, valida body/params. Complementa [[edge-fn-authz]] (autz na entrada) e [[tenant-audit]] (isolation na query) cobrindo a camada de schema. Use ao criar edge function nova ([[edge-fn-new]]), revisar PR de função, ou quando o usuário disser "tá vindo body errado", "validação dessa rota".
---

# Zod schemas — edge functions Lumied

Contexto: hoje a maioria das edge functions Lumied lê `await req.json()` e usa direto. Sem validação:
- `escola_id` pode vir vazio, malformado, ou de outro tenant ([[tenant-audit]] pega no banco, mas tarde).
- Campos obrigatórios faltando viram erro 500 confuso.
- Tipos errados (string em vez de uuid, number em vez de string) viram exception de banco.
- Atacante pode mandar payload massivo, prototype pollution, JSON inválido.

Zod (ou Valibot, ou Yup) resolve isso na entrada com mensagem clara e tipos TS gerados.

## Quando rodar

- Edge function nova ([[edge-fn-new]] já deve incluir schema base).
- PR review de função que recebe body / query params (parte do [[pr-review-lumied]]).
- Bug reportado de "campo X não foi validado" / "veio undefined".
- Refator de função legada (Onda 3 [[project_refator_lumied]]).

## O padrão

### Setup mínimo (Deno + Zod)

```ts
import { z } from 'https://deno.land/x/zod@v3.22.4/mod.ts';

const inputSchema = z.object({
  escola_id: z.string().uuid(),
  aluno_id: z.string().uuid(),
  valor: z.number().positive().max(1_000_000),
  vencimento: z.string().regex(/^\d{4}-\d{2}-\d{2}$/),
  descricao: z.string().min(1).max(500),
  metadata: z.record(z.string()).optional(),
});

type Input = z.infer<typeof inputSchema>;
```

### No handler

```ts
Deno.serve(async (req) => {
  // 1. auth ([[edge-fn-authz]]) primeiro
  const { user, escolaId } = await validarJWT(req);

  // 2. parse body
  let raw;
  try {
    raw = await req.json();
  } catch {
    return new Response('JSON inválido', { status: 400 });
  }

  // 3. valida com Zod
  const parsed = inputSchema.safeParse(raw);
  if (!parsed.success) {
    return Response.json(
      {
        error: 'Dados inválidos',
        detalhes: parsed.error.flatten().fieldErrors,
      },
      { status: 400 }
    );
  }

  const input = parsed.data;

  // 4. cross-check tenant ([[tenant-audit]])
  if (input.escola_id !== escolaId) {
    return new Response('escola_id divergente do JWT', { status: 403 });
  }

  // 5. processa
  ...
});
```

## Schemas reutilizáveis (criar `_shared/schemas.ts`)

```ts
import { z } from 'https://deno.land/x/zod@v3.22.4/mod.ts';

export const uuidSchema = z.string().uuid();
export const dataSchema = z.string().regex(/^\d{4}-\d{2}-\d{2}$/); // YYYY-MM-DD
export const datetimeSchema = z.string().datetime();
export const cnpjSchema = z.string().regex(/^\d{14}$/);
export const cpfSchema = z.string().regex(/^\d{11}$/);
export const valorSchema = z.number().positive().max(10_000_000);
export const escolaIdSchema = uuidSchema;

export const paginacaoSchema = z.object({
  page: z.coerce.number().int().min(1).default(1),
  limit: z.coerce.number().int().min(1).max(100).default(20),
});

export const periodoSchema = z.object({
  inicio: dataSchema,
  fim: dataSchema,
}).refine(d => d.inicio <= d.fim, { message: 'inicio deve ser <= fim' });
```

Reutilizar em todas as funções.

## Procedimento de auditoria

### 1. Achar funções sem validação

```bash
# funções que leem body sem validar
grep -rL "z\.object\|inputSchema\|safeParse" supabase/functions/ --include=index.ts
```

### 2. Pra cada uma, classificar

- **🔴 Sem validação alguma** — `await req.json()` direto pra DB.
- **🟠 Validação ad-hoc** — `if (!body.escola_id) ...` espalhado, frágil.
- **🟢 Schema completo** — Zod (ou equivalente) com types gerados.

### 3. Sugerir schema

Pra cada 🔴/🟠, propor o schema baseado no que a função usa:

```
## supabase/functions/gerar_boleto/index.ts

Hoje:
  const { aluno_id, valor, vencimento } = await req.json();
  // sem checar tipos, obrigatoriedade, formato

Sugerido:
  const inputSchema = z.object({
    aluno_id: uuidSchema,
    valor: valorSchema,
    vencimento: dataSchema,
    descricao: z.string().max(200).optional(),
  });
  const parsed = inputSchema.safeParse(await req.json());
  if (!parsed.success) return badRequest(parsed.error);
```

## Anti-padrões

- Confiar em TypeScript types no body — TS desaparece em runtime; valida não.
- Validação manual com `if (!x) throw` — espalhado, sem mensagem clara, sem types.
- Usar Zod só pra campos óbvios — esquecer length max, regex, refine cruzado.
- Schema duplicado em cada função — extrair `_shared/schemas.ts`.
- Validar mas continuar usando `body.x` em vez de `parsed.data.x` — perde o tipo.
- Não validar `escola_id` cross-check com JWT — falha [[tenant-audit]].
- Mensagem de erro Zod cru vazando — `parsed.error.message` é verboso. Use `flatten()` ou mapeie pra mensagem em PT-BR ([[microcopy-ptbr]]).

## Padrões úteis Zod no contexto Lumied

### Coerce de query string

```ts
// ?page=2&limit=50 — strings
const q = z.object({
  page: z.coerce.number().int().min(1).default(1),
  limit: z.coerce.number().int().min(1).max(100).default(20),
}).parse(Object.fromEntries(url.searchParams));
```

### Discriminated union (multi-tipo de input)

```ts
const acaoSchema = z.discriminatedUnion('tipo', [
  z.object({ tipo: z.literal('baixar'), pagamento_id: uuidSchema }),
  z.object({ tipo: z.literal('cancelar'), motivo: z.string().min(3) }),
]);
```

Útil em endpoints que recebem `{ tipo: 'baixar' | 'cancelar', ... }` (action handlers — cross-check [[feedback_diplomas_allowlist]] sobre allowlist de actions).

### Refinement cruzado

```ts
const schema = z.object({
  vencimento: dataSchema,
  pago_em: dataSchema.optional(),
  status: z.enum(['aberto', 'pago']),
}).refine(d => {
  if (d.status === 'pago' && !d.pago_em) return false;
  return true;
}, { message: 'pago_em obrigatório quando status=pago' });
```

### Transform pra normalizar

```ts
const cpfNormalizado = z.string().transform(s => s.replace(/\D/g, '')).pipe(cpfSchema);
// aceita "123.456.789-00" ou "12345678900"
```

## Output esperado

```
## Zod audit — supabase/functions/

🔴 Sem validação:
1. gerar_boleto — recebe valor, vencimento, descricao sem validar tipo
2. importar_alunos — array de objetos sem schema
3. bank_inter_webhook — assinatura OK mas body sem schema

🟠 Validação ad-hoc:
1. enviar_cobranca — checa `if (!email)` mas não valida formato
2. autorizar_saida — escola_id checado mas tipo number aceita string

🟢 OK:
- escola_onboarding — schema completo
- ml_oauth_refresh — schema completo

Sugestão:
- Criar _shared/schemas.ts com primitivos (uuidSchema, dataSchema, cnpjSchema, etc.)
- Refatorar 🔴 primeiro (alto impacto), 🟠 depois.
```

## Cross-check

- [[edge-fn-authz]] — auth **antes** de validação de schema (rejeita não-autenticado antes de parse).
- [[tenant-audit]] — escola_id no schema **e** validar contra JWT.
- [[edge-fn-new]] — incluir schema base no boilerplate.
- [[pr-review-lumied]] — adicionar zod-schemas ao checklist da review.
- [[microcopy-ptbr]] — mensagens de erro mapeadas em PT-BR antes de retornar pro front.
- [[test-coverage]] — schemas são fáceis de testar; teste com payload inválido = retorno 400 estruturado.
