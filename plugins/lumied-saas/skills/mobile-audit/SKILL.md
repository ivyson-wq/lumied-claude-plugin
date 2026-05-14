---
name: mobile-audit
description: Audita página HTML/JSX para uso em mobile — viewport, tap targets ≥44px, fonts legíveis, sem scroll horizontal, modais usáveis no celular. Crítico porque pais e alunos acessam Lumied do telefone. Use ao criar tela voltada a usuário externo (pais/alunos) ou quando o usuário disser "vê se isso roda no celular".
---

# Mobile audit — Lumied

Contexto: dashboards internos (admin/gerente/prof) abrem em desktop, mas pais/alunos/responsáveis acessam quase sempre do celular. Trair viewport mobile = adoção baixa e reclamação direta do cliente.

## Páginas SEMPRE mobile-first (usuários externos)

- `pais.html`, `aluno.html`, `responsavel.html`
- Páginas de login/cadastro/recuperação de senha
- Confirmação de matrícula
- Boletim/notas (pais consultam pelo telefone)
- Comunicados, agenda do dia
- Pagamento/2ª via boleto (LGPD + UX crítica)
- Landing pages comerciais
- Páginas de tabelas comparativas tipo [[project_tier_starter]] (`/vs/escolaweb/`)

## Páginas desktop-first (mas devem degradar graciosamente)

- `admin.html`, `gerente.html`, `prof.html`, `secretaria.html`
- Painéis de configuração / importação ERP
- Reconcile/conciliação Construfare (planilhas largas — ok rolar horizontal CONSCIENTE)

## Checklist obrigatório

### 1. Viewport meta tag

```html
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
```

🔴 Ausente ou com `user-scalable=no`? Quebra acessibilidade + zoom involuntário em inputs.

### 2. Tap targets ≥ 44×44px

Botões, links, checkboxes interativos:
```css
.btn { min-height: 44px; padding: 12px 16px; }
input[type=checkbox], input[type=radio] { width: 24px; height: 24px; }
a { padding: 12px 0; } /* se link em lista */
```

🔴 Botão com `padding: 4px 8px` numa página de pai/aluno = dedo grosso erra.

### 3. Sem scroll horizontal

```css
html, body { overflow-x: hidden; max-width: 100%; }
img, video, iframe, table { max-width: 100%; height: auto; }
```

Tabelas largas: envolver em `<div class="table-scroll">` com `overflow-x: auto` LOCAL (não global).

### 4. Fontes legíveis

Mínimo:
- Body: 14px (16px ideal).
- Inputs: 16px (16px evita auto-zoom iOS ao focar!).
- Botões CTA: 15-16px.

🔴 `font-size: 11px` em mobile = inacessível.

### 5. Inputs

```html
<input type="email" inputmode="email" autocomplete="email">
<input type="tel" inputmode="tel" autocomplete="tel-national">
<input type="number" inputmode="decimal"> <!-- pra valores monetários -->
<input type="date">
```

Sempre setar `inputmode` apropriado pra abrir teclado certo.

### 6. Modais

- Em mobile, modal deve ocupar quase tela inteira (não ser caixinha no meio com fundo cortado).
- Botão de fechar deve ser visível sem scroll.
- Form em modal: scroll interno, NÃO scroll da página por trás.

```css
@media (max-width: 640px) {
  .modal { width: 100%; height: 100%; border-radius: 0; }
}
```

### 7. Layouts

- Mobile: stack vertical (1 coluna). Desktop: grid/flex.
- Tabela > 3 colunas em mobile: virar lista de cards (uma "linha" por card) OU permitir scroll horizontal explícito.

### 8. Imagens

- `<img loading="lazy">` em galerias.
- Servir tamanhos diferentes via `srcset` se a imagem é grande.
- Avatares em `<img width="40" height="40">` (com attributes) pra evitar layout shift.

### 9. Performance (afeta mobile mais que desktop)

- Bundle JS por página, não one-big.js global se a página é pra pai.
- Defer scripts não críticos: `<script defer>`.
- Evitar `<script>` síncrono no `<head>` sem `async/defer`.

## Procedimento

1. Identificar a página em questão.
2. Rodar `grep` no arquivo procurando:
   - `<meta name="viewport"` — presente?
   - `font-size: \d+px` — algum < 14px?
   - Botões com `padding` pequeno.
   - `<table>` sem wrapper de scroll.
   - `<input type="text"` sem `inputmode`.
3. Abrir o arquivo e simular mentalmente em 360×740 (iPhone SE / Android base).
4. Se o usuário tiver dev server rodando, sugerir abrir DevTools mobile emulator.
5. Reporte:
   ```
   pages/pais.html
     🔴 sem <meta viewport>
     🔴 botão "Confirmar leitura" L142: padding 4px 8px = ~22px height
     🟡 tabela boletim L201 sem wrapper scroll horizontal
     🟡 input CPF L67 sem inputmode="numeric"
     ✅ fontes ≥14px em todo o body
   ```
6. Patch quando o usuário pedir.

## Quando NÃO se preocupar

- Tela de admin global usada só pelo Ivyson em desktop.
- Tela de importação ERP que precisa upload de CSV grande (desktop é melhor).
- Reconcile/conciliação Construfare (planilha = desktop).
- Mockups visuais ainda não validados.
