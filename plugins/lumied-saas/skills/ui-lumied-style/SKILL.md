---
name: ui-lumied-style
description: Audita ou aplica o design system Lumied em telas novas — cores, espaçamento, tipografia, componentes reutilizáveis em vez de inline styles. Use ao criar página HTML/JSX nova, refatorar tela legada, ou quando o usuário falar "deixa com cara de Lumied", "padroniza o visual".
---

# Design system audit — Lumied

Contexto: o Lumied tem padrão visual estabelecido nos painéis (`admin.html`, `gerente.html`, `prof.html`, etc.). Telas novas costumam nascer com inline styles aleatórios — quebra coerência visual e dificulta refator (vide [[project_refator_lumied]]).

## Tokens canônicos Lumied

### Cores (procure no CSS principal, geralmente `style.css` ou `<style>` inline no head)

| Token | Hex aproximado | Uso |
|---|---|---|
| `--lumied-primary` | `#1976d2` (azul) | botões principais, links |
| `--lumied-accent` | `#ff9800` (laranja) | destaques, badges "novo" |
| `--lumied-success` | `#388e3c` | confirmações, status OK |
| `--lumied-warn` | `#f57c00` | avisos não-críticos |
| `--lumied-danger` | `#d32f2f` | erros, deletar |
| `--lumied-bg` | `#f5f5f5` | fundo geral |
| `--lumied-surface` | `#ffffff` | cards, modais |
| `--lumied-text` | `#212121` | corpo |
| `--lumied-text-muted` | `#757575` | secundário |
| `--lumied-border` | `#e0e0e0` | separadores |

> **Antes de auditar:** abra `admin.html` ou `style.css` no projeto e confirme os valores reais; este é guia. Atualize esta skill se o projeto canonizar outro set.

### Espaçamento (escala 4px)

- `--sp-1: 4px` — entre ícone e texto.
- `--sp-2: 8px` — gap entre itens da mesma row.
- `--sp-3: 12px` — padding interno de inputs.
- `--sp-4: 16px` — padding de card, gap entre seções.
- `--sp-6: 24px` — margem entre blocos.
- `--sp-8: 32px` — separação de áreas maiores.

### Tipografia

- Family: system stack (`-apple-system, Segoe UI, Roboto, sans-serif`).
- H1: 24px / 700.
- H2: 20px / 600.
- Body: 14px / 400.
- Small: 12px / 400.
- Line-height: 1.4 default, 1.6 em parágrafos longos.

### Border radius

- Inputs/botões: `6px`.
- Cards/modais: `8px`.
- Pills/badges: `999px`.

## Componentes obrigatórios

Não recriar — usar/promover:

- `.btn.btn-primary`, `.btn.btn-secondary`, `.btn.btn-danger`, `.btn.btn-link`.
- `.card` (com `.card-header`, `.card-body`, `.card-footer`).
- `.alert.alert-info|warn|danger|success` para feedback inline (vide [[feedback_limitacoes_visiveis]]).
- `.input`, `.select`, `.textarea` (sem reimplementar estilo).
- `.skeleton` para loading (vide [[loading-states]]).
- `.empty-state` para listas vazias.
- `.table.table-zebra` para listagens.
- `.modal` + `.modal-backdrop`.

## Anti-padrões a sinalizar

| Pattern | Por que ruim | Fix |
|---|---|---|
| `style="color: #1976d2"` | hardcode de cor | usar var(--lumied-primary) ou classe |
| `style="padding: 13px"` | número arbitrário fora da escala | trocar pra escala 4px |
| `<button onclick="...">` cru | sem estilo | adicionar `class="btn btn-primary"` |
| `font-family: Arial` em uma tela | divergência | herdar do body |
| Modal feito do zero | duplicação | usar `.modal` existente |
| `margin-bottom: 20px; margin-top: 15px;` | inconsistente | escala + gap em parent |
| Ícones de fonts diferentes | mistura visual | uniformizar (Lucide ou Material) |
| `<div>` clicável sem `<button>` | a11y + UX | usar botão real |

## Procedimento (audit)

1. Identificar arquivo(s) target: HTML/JSX/Vue que o usuário pediu.
2. Grep no arquivo por:
   - `style="..."` inline → listar todos.
   - `class=""` ausente em interactives.
   - Cores hex no estilo (regex `#[0-9a-fA-F]{3,6}`).
   - Tamanhos fora da escala 4px.
3. Cruzar com `style.css` canônico — o que tem var/classe disponível?
4. Reporte:
   ```
   pages/nova-tela.html
     🟡 L42 botão "Salvar" sem class — usar .btn.btn-primary
     🔴 L67 cor #2196f3 inline — usar var(--lumied-primary) (#1976d2)
     🟡 L89 padding: 13px — fora da escala (use 12px = --sp-3)
     ✅ L103 card usa .card corretamente
   ```
5. Se o usuário pediu para aplicar, gerar patch consolidando inline styles em classes, adicionando vars onde apropriado.

## Procedimento (apply em tela nova)

1. Começar pelo wrapper: `<div class="page">` ou structure padrão do projeto.
2. Cabeçalho com `.page-header` se existir.
3. Conteúdo em `.card`s, não em `<section>` cru sem estilo.
4. Botões: classe + variant. Nunca onclick handler nu sem visual.
5. Estados de loading/empty/error obrigatórios (vide [[loading-states]]).
6. Responsivo (vide [[mobile-audit]]).
7. Texto i18n se houver pattern no projeto, senão pt-BR direto (Lumied é mono-idioma hoje).

## Quando NÃO aplicar

- Páginas legadas que serão arquivadas (vide ondas de [[project_refator_lumied]] — não pinta uma tela que vai morrer na próxima onda).
- Mockups de prova de conceito que ainda não foram validados com escola piloto.
