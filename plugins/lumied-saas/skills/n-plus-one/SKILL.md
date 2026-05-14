---
name: n-plus-one
description: Detecta query dentro de loop em edge functions Supabase/Deno do Lumied — N queries em vez de 1 join/`in()`. Impacto direto em IO ([[project_lumied_io_storage]]) e latência. Use ao revisar edge function nova, quando [[db-index-audit]] aponta IO alto sem causa clara, ou quando o usuário disser "essa função tá demorando muito".
---

# N+1 detection — edge functions Lumied

Contexto: padrão recorrente em código escrito rápido — buscar N itens da tabela A, depois pra cada item buscar 1 item da tabela B. Resultado: 1+N queries. Em listagem de 50 alunos com responsável = 51 queries. Em endpoint chamado por 10 escolas = 510 queries por chamada. É um dos principais drenos de IO no Supabase Free.

## Quando rodar

- PR review de `supabase/functions/**` (parte do [[pr-review-lumied]]).
- [[db-index-audit]] aponta IO alto mas índices estão OK.
- Função reportada como lenta (>1s).
- Após [[cost-audit]] mostrar query count Supabase alto.

## Como detectar

### Padrão típico (anti-padrão)

```ts
// SUSPEITO — busca N+1
const { data: alunos } = await supabase
  .from('alunos').select('*').eq('escola_id', escolaId);

for (const aluno of alunos) {
  // ← 1 query por aluno
  const { data: responsavel } = await supabase
    .from('responsaveis').select('nome, email')
    .eq('id', aluno.responsavel_id).single();

  aluno.responsavel = responsavel;
}
```

### Refator pra 2 queries

```ts
// MELHOR — 2 queries totais
const { data: alunos } = await supabase
  .from('alunos').select('*').eq('escola_id', escolaId);

const responsavelIds = [...new Set(alunos.map(a => a.responsavel_id).filter(Boolean))];

const { data: responsaveis } = await supabase
  .from('responsaveis').select('id, nome, email')
  .in('id', responsavelIds);

const respMap = new Map(responsaveis.map(r => [r.id, r]));

for (const aluno of alunos) {
  aluno.responsavel = respMap.get(aluno.responsavel_id) ?? null;
}
```

### Ou 1 query com nested select (Supabase)

```ts
// MELHOR AINDA (geralmente) — 1 query com join
const { data: alunos } = await supabase
  .from('alunos')
  .select(`
    *,
    responsavel:responsaveis(nome, email)
  `)
  .eq('escola_id', escolaId);
```

Nested select usa FK relationship configurada no Supabase. Funciona se a FK existe e RLS permite. Cuidado: nested select pode causar **own** N+1 internamente se a relação é 1:N grande (pode trazer dados duplicados ou explodir payload).

## Procedimento

### 1. Grep por padrões suspeitos

```bash
# Loops com await dentro
grep -rE "for\s*\(.*of.*\)|forEach|Promise\.all" supabase/functions/ \
  | xargs -I{} grep -l "supabase\.from\|supabase\.rpc"
```

Pra cada match, abrir o arquivo e procurar:
- `for (const x of list)` seguido de `await supabase.from(...)` dentro.
- `list.forEach(async ...)` seguido de `await supabase...` — bonus: `forEach` com async **não espera**, é bug duplo.
- `Promise.all(list.map(async (x) => await supabase...))` — N queries em paralelo, melhor que serial mas ainda N.

### 2. Classificar cada match

- **🔴 N+1 confirmado:** loop sobre N items + 1 query por item dentro.
- **🟠 N queries em paralelo:** `Promise.all` com query por item — não é N+1 no sentido estrito (não bloqueia), mas ainda é N queries; refatorar pra `.in()` se possível.
- **🟢 Falso positivo:** loop é pra processar dados em memória, sem query Supabase dentro.

### 3. Sugerir refator

Pra cada 🔴 ou 🟠:
- Opção A: buscar todos os IDs primeiro + `.in()` (2 queries).
- Opção B: usar nested select do Supabase (1 query).
- Opção C: criar SQL function (RPC) que faz JOIN — se a lógica é complexa demais pra .select().

### 4. Output esperado

```
## N+1 audit — supabase/functions/

🔴 Bloqueios:

1. boletins_publicar/index.ts:42
   for (const aluno of alunos) {
     await supabase.from('notas').select(...).eq('aluno_id', aluno.id)
   }
   → Refator: .in('aluno_id', alunos.map(a => a.id)) + agrupa por aluno em memória.

2. relatorio_financeiro_mensal/index.ts:78
   for (const conta of contas) {
     await supabase.rpc('calcula_juros', { conta_id: conta.id })
   }
   → Refator: criar `calcula_juros_bulk(ids[])` ou fazer cálculo client-side.

🟠 Para olhar:

1. importer_escolaweb/index.ts:120 — Promise.all com .insert() por aluno (N writes paralelos)
   → OK pra import único; verificar se cabe em batch insert com `.upsert([...])`.

🟢 OK:
- bank_inter_webhook — sem loops com query dentro
- conciliacao_v2 — usa nested select corretamente
```

## Anti-padrões correlatos

### forEach com async sem await

```ts
// BUG — não espera, retorna antes
items.forEach(async (item) => {
  await processar(item);
});

// CORRETO
for (const item of items) {
  await processar(item);
}
// ou
await Promise.all(items.map(item => processar(item)));
```

### `await` dentro de `.map` retornando promise

```ts
// SUSPEITO — array de promises, perde se não der Promise.all
const resultados = items.map(async (item) => {
  return await supabase.from(...).select(...);
});
// resultados é Promise[], não dados.
```

### Buscar pra count

```ts
// RUIM — traz todos os dados pra contar
const { data } = await supabase.from('alunos').select('*').eq('escola_id', x);
const total = data.length;

// MELHOR
const { count } = await supabase.from('alunos').select('*', { count: 'exact', head: true }).eq('escola_id', x);
```

### Buscar tudo + filtrar em JS

```ts
// RUIM
const { data: alunos } = await supabase.from('alunos').select('*');
const ativos = alunos.filter(a => a.status === 'ativo');

// CERTO
const { data: ativos } = await supabase.from('alunos').select('*').eq('status', 'ativo');
```

(Cross-check com [[tenant-audit]] — `select('*')` sem `.eq('escola_id', ...)` também é leak!)

## Princípios

1. **Conta as queries.** Antes de revisar perf, conta quantas queries a função faz por chamada.
2. **N queries paralelas > N serial, mas 1 query > N queries.** Refator sempre que possível.
3. **`.in()` é amigo.** Cobre 80% dos casos.
4. **Nested select é amigo, mas cuidado.** Pode explodir payload se relação 1:N.
5. **RPC é última opção.** Funciona, mas é menos testável e RLS fica mais complexa.

## Casos específicos Lumied

### Listagens de alunos com dados agregados

Telas como `gerente.html` mostram lista de alunos + total pago + status financeiro. Tendência a ter N+1. Olhar primeiro.

### Importadores ERP ([[erp-import-runbook]])

Importer faz INSERT por linha. OK pra import inicial (raro, single-shot), mas se vira diário (delta sync), refatorar pra batch upsert.

### Reconcile ([[project_sienge_reconcile]])

Loop sobre N contas + 1 query por conta = mata IO. Padrão Construfare usa map em memória pra evitar.

### Webhooks de banco ([[bank-homologacao]])

Webhook geralmente é 1 conta = 1 query. Mas se banco manda batch (Itaú faz), garantir tratamento sem N+1.

## Cross-check

- [[db-index-audit]] — se há N+1 + sem índice = catástrofe.
- [[edge-fn-authz]] — refator de N+1 não pode quebrar validação de JWT/role.
- [[tenant-audit]] — refator não pode remover filtro `escola_id`.
