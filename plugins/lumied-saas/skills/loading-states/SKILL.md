---
name: loading-states
description: Audita tela para garantir que toda operação async tem feedback visual — loading, empty, error, success states. Materializa o princípio [[feedback_limitacoes_visiveis]] de que falhas previsíveis precisam ser visíveis ao usuário, não só silenciosas no console. Use ao revisar tela nova ou quando o usuário disser "essa tela trava do nada".
---

# Loading states audit — Lumied

Contexto: [[feedback_limitacoes_visiveis]] — toda limitação/falha previsível deve ser explicada na UI, não só no código. Botão sem feedback ao clicar = usuário clica 5 vezes = duplica registro/spam de email/etc.

## Os 4 estados obrigatórios para toda ação async

Para cada operação (fetch, mutação, upload, geração), a tela precisa cobrir:

| Estado | O que mostrar | Anti-padrão |
|---|---|---|
| **idle** | UI normal pronta pra ação | — |
| **loading** | spinner OU skeleton OU "Carregando..." | botão sem mudança = usuário reclica |
| **empty** | mensagem explicando + CTA pra próxima ação | tabela vazia silenciosa |
| **error** | mensagem clara + retry / fallback | console.error e nada na UI |
| **success** (quando aplicável) | toast/inline confirmando + UI atualizada | só recarregar a página inteira |

## Padrões Lumied

### Botão com loading

```html
<button class="btn btn-primary" id="btn-salvar">
  <span class="btn-label">Salvar</span>
  <span class="btn-spinner hidden">⏳</span>
</button>
```

```js
async function salvar() {
  const btn = document.getElementById('btn-salvar')
  btn.disabled = true
  btn.querySelector('.btn-label').classList.add('hidden')
  btn.querySelector('.btn-spinner').classList.remove('hidden')
  try {
    await api.salvar(...)
    toast('Salvo!', 'success')
  } catch (e) {
    toast('Falhou: ' + (e?.message || 'erro desconhecido'), 'error')
  } finally {
    btn.disabled = false
    btn.querySelector('.btn-label').classList.remove('hidden')
    btn.querySelector('.btn-spinner').classList.add('hidden')
  }
}
```

### Tabela com 4 estados

```html
<div id="tabela-pedidos">
  <div class="state-loading">⏳ Carregando pedidos…</div>
  <div class="state-empty hidden">
    Nenhum pedido ainda.
    <button class="btn btn-primary">Criar primeiro pedido</button>
  </div>
  <div class="state-error hidden">
    <p>Não consegui carregar. <a href="#" onclick="recarregar()">Tentar de novo</a></p>
  </div>
  <table class="state-data hidden">...</table>
</div>
```

### Skeleton (preferível ao spinner pra listas)

```html
<div class="skeleton-row" aria-hidden="true">
  <div class="skel-cell w-40"></div>
  <div class="skel-cell w-20"></div>
</div>
```

### Toast / feedback inline

Usar componente `.alert` ou `.toast` do design system ([[ui-lumied-style]]) — não `alert()` nativo (UX feia + bloqueia thread).

## Falhas previsíveis a expor explicitamente

Inspirado em [[feedback_limitacoes_visiveis]]:

- **Sem internet** → "Você está offline. Vou tentar de novo quando voltar."
- **Sessão expirada** → "Sua sessão acabou. Faça login de novo." + redirect.
- **Permissão negada** → "Você não tem permissão pra isso. Peça pro gerente."
- **Quota/limite atingido** → "Limite do plano atingido — entre em contato com o admin." (vide [[project_lumied_io_storage]]).
- **Banco em manutenção** → "Estamos com problema temporário com o Supabase. Tentando novamente em X segundos…"
- **Cron rodando** → "Importação em andamento (~5 min). Pode fechar a aba, te aviso por email." ([[project_sienge_reconcile]] vai longo).
- **Validação de formulário** → erro POR CAMPO com mensagem clara, não banner genérico.

## Checklist

Pra cada arquivo HTML/JSX com interatividade:

1. Grep `addEventListener('click'`, `onClick=`, `<form` → toda ação interativa.
2. Pra cada uma, achar a função handler. Conferir:
   - Tem `try/catch`?
   - Catch atualiza UI ou só `console.error`?
   - Tem indicador de loading enquanto roda?
   - Sucesso é confirmado visualmente?
3. Grep `fetch(`, `supabase.from(`, `axios.` → toda chamada async.
4. Conferir se o caller tem estados.
5. Grep `alert(`, `confirm(`, `prompt(` → substituir por toast/modal estilizado.
6. Reporte:
   ```
   pages/pedidos.html
     🔴 L42 form submit sem catch — erro vai pro console invisível
     🔴 L67 tabela carrega sem skeleton + sem estado empty
     🟡 L89 alert('Salvo!') — usar toast
     ✅ L103 botão "Excluir" desabilita durante request
   ```

## Anti-padrões frequentes

- `await fetch(...)` sem try/catch.
- Botão `disabled` apenas mas sem indicação visual (cursor: not-allowed nem aparece em mobile).
- "Loading..." que nunca some se a request falhar (timeout ausente).
- Empty state idêntico ao loading (usuário não sabe se carregou ou está vazio).
- Erro genérico "Algo deu errado" sem código nem ação. Mostrar `error.message` quando seguro.
- Confirmação destrutiva via `confirm()` nativo (não traduz, não estiliza, irmaã do alert).
