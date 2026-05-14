---
name: a11y-quick
description: Audit rápido de acessibilidade — labels em inputs, contraste mínimo, foco visível, navegação por teclado, alt em imagens, ARIA básico. Não é WCAG completo, é o mínimo decente. Use ao revisar tela nova, ou quando o usuário disser "isso é acessível?".
---

# Acessibilidade básica — Lumied

Contexto: Lumied tem usuários idosos (avós cuidando do neto), com baixa visão, e até screen reader. Não precisa cobrir WCAG AAA, mas o mínimo decente impede situações de "minha mãe não consegue ler o boletim".

## Checklist mínimo

### 1. Labels em inputs

```html
<!-- ✅ -->
<label for="cpf">CPF</label>
<input id="cpf" type="text" inputmode="numeric">

<!-- ✅ alternativa -->
<label>CPF <input type="text"></label>

<!-- 🔴 placeholder NÃO substitui label -->
<input placeholder="CPF">
```

Inputs sem label → screen reader fala "edit text" sem dizer pra quê.

### 2. Contraste

Mínimo WCAG AA: **4.5:1** pra texto normal, **3:1** pra texto grande (≥18px ou ≥14px bold).

Checagem rápida sem ferramenta:
- Texto cinza claro `#999` em fundo branco = ~2.8:1 → 🔴 reprovado.
- Texto `#666` em fundo branco = ~5.7:1 → ✅ ok.
- Texto branco em laranja `#ff9800` = ~2.5:1 → 🔴 reprovado (use laranja mais escuro pra CTA).

### 3. Foco visível

```css
/* 🔴 nunca remover sem alternativa */
*:focus { outline: none; }

/* ✅ remover só pra mouse, manter pra teclado */
*:focus-visible { outline: 2px solid var(--lumied-primary); outline-offset: 2px; }
```

### 4. Navegação por teclado

- Tab percorre todos os interactives em ordem lógica.
- Enter ativa botões; Space também em botões; Enter envia forms.
- Esc fecha modais.
- Setas em listas/grids se aplicável.
- Sem trapas de foco (modal abre mas Tab escapa pra trás).

Antipattern: `<div onClick>` em vez de `<button>`. `div` não é tabbable nem ativável por teclado.

### 5. Alt em imagens

```html
<!-- ✅ informativa -->
<img src="logo.png" alt="Logo Lumied">

<!-- ✅ decorativa -->
<img src="divider.svg" alt="" role="presentation">

<!-- 🔴 ausente -->
<img src="foto.jpg">
```

### 6. Heading hierarchy

- Uma `<h1>` por página.
- Não pular níveis: `h1 → h2 → h3`, não `h1 → h4`.
- Usar headings semanticamente, não pra tamanho.

### 7. Botões vs links

- `<button>` pra ações (submit, abrir modal, toggle).
- `<a href>` pra navegação (mudar URL).
- 🔴 `<a href="#" onclick="...">` quando deveria ser button.

### 8. ARIA mínimo

- Modal: `role="dialog" aria-modal="true" aria-labelledby="...título"`.
- Toast/alert: `role="alert"` ou `aria-live="polite"`.
- Loading: `aria-busy="true"` no container.
- Botão de ícone só (sem texto): `aria-label="Fechar"`.

### 9. Forms

- Erros associados ao campo: `<input aria-describedby="erro-cpf"><span id="erro-cpf">CPF inválido</span>`.
- Campos obrigatórios: `<input required aria-required="true">`.
- Botão submit faz submit; não confiar só em listener de click.

### 10. Tabelas

- `<th>` com `scope="col"` ou `scope="row"`.
- `<caption>` quando útil (especialmente em boletins/notas).

## Procedimento

1. Identificar arquivo target.
2. Rodar greps:
   - `<input` → cada um tem `<label>` próximo (mesmo `for` ou wrap)?
   - `<img` → todos com `alt=`?
   - `<button>` vs `<div onclick`?
   - `outline: none` ou `outline: 0` no CSS?
   - `<h1>` aparece quantas vezes?
3. Conferir cores no estilo: tem `color: #aaa` ou similar em fundo claro?
4. Mentalmente: clica Tab partindo do topo — chega em todos os interactives em ordem lógica?
5. Reporte:
   ```
   pages/aluno.html
     🔴 L42 <input type="text" placeholder="Nome"> sem label
     🔴 L67 <div onclick="abrirModal()"> deveria ser <button>
     🟡 L89 cor #aaa em fundo branco — contraste ~2.6:1 < 4.5
     🟡 L103 múltiplos <h1> — só um por página
     ✅ modal tem role="dialog" + aria-modal
   ```

## Ferramentas auxiliares

Se o user quiser audit mais profundo:
- **axe DevTools** (extensão Chrome) — roda no browser.
- **Lighthouse** (DevTools) — score de accessibility.
- **NVDA** (Windows) ou **VoiceOver** (Mac) — testar com screen reader real.

## Quando NÃO ser ortodoxo

- Mockups internos (admin global usado só pelo dono).
- Telas com prazo apertado pra escola piloto — entregar, voltar depois pra polir.
- Não compensar a11y AAA em página comercial estática (foque AA básico).
