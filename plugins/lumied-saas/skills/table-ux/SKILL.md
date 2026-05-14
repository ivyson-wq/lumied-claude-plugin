---
name: table-ux
description: Padrão de tabela Lumied — busca, filtros, ordenação, paginação, seleção múltipla, ações em lote, fallback mobile (card). Lumied tem 30+ tabelas (alunos, lançamentos, manut, almox, ponto, conciliação) hoje inconsistentes. Use ao criar tela com listagem, refatorar tabela legada, ou quando o usuário disser "padroniza essa tabela", "essa lista tá ruim de usar".
---

# Tabela — padrão Lumied

Contexto: a UI do Lumied é dominada por tabelas. Cada módulo (acadêmico, financeiro, almox, RH, manut, ponto, comunicação) tem várias. Hoje cada tela inventou seu próprio padrão de busca/filtro/ordem — usuário aprende uma tela e a próxima funciona diferente. Esta skill define o padrão e aponta o que falta numa tela existente.

## Quando rodar

- Criando tela nova com listagem.
- Refatorando tabela legada (Onda 3 do [[project_refator_lumied]]).
- Usuário/cliente reclamou "não acho fulano", "como ordeno?", "queria selecionar várias linhas".
- Em conjunto com [[mobile-audit]] (tabela em mobile é caso especial crítico).

## O padrão

### Estrutura visual

```
┌─────────────────────────────────────────────────────────────────┐
│  Título da listagem                       [+ Nova ação primária] │
├─────────────────────────────────────────────────────────────────┤
│  [🔍 Busca livre]  [Filtro A ▾] [Filtro B ▾]  [↓ Exportar]      │
├─────────────────────────────────────────────────────────────────┤
│  ☐  Coluna A ↕    │  Coluna B ↕  │  Coluna C  │  Ações         │
├─────────────────────────────────────────────────────────────────┤
│  ☐  linha 1...                                                  │
│  ☐  linha 2...                                                  │
├─────────────────────────────────────────────────────────────────┤
│  X de Y resultados   [< anterior] [1] [2] [3] [próxima >]       │
└─────────────────────────────────────────────────────────────────┘
```

### Componentes obrigatórios

1. **Busca livre** (input com debounce ~300ms) — single source of truth, filtra na coluna principal (nome do aluno, descrição do lançamento, etc.).
2. **Filtros estruturados** — selects pra status, período, categoria. Não inventar combobox custom; usar `<select>` ou componente do design system.
3. **Ordenação por coluna** — click no header alterna asc/desc/none. Indicador visual (↑ ↓ ↕).
4. **Paginação** — server-side se mais de ~50 linhas esperadas; client-side se sempre <50. Padrão: 20 ou 50 por página.
5. **Contagem total** — "X de Y resultados" — usuário precisa saber se filtro reduziu.
6. **Estado vazio** — ver [[empty-states]]. Sem resultados na busca ≠ lista vazia inicial.
7. **Loading** — ver [[loading-states]]. Skeleton de linha, não spinner solto.

### Componentes opcionais (quando faz sentido)

- **Seleção múltipla** com checkbox + ações em lote no header da tabela ("Marcar como pago", "Exportar selecionados").
- **Exportar** — CSV/Excel direto da listagem filtrada.
- **Densidade** — toggle compacto/confortável se a tela vive lotada.
- **Colunas configuráveis** — só se a tabela tem 10+ colunas e o usuário precisa esconder. Senão é over-engineering.

### Mobile fallback

Em mobile (<640px), tabela vira **lista de cards**:
- Cada linha vira um card com 2-3 campos chave + botão "Detalhes".
- Filtros ficam num drawer/bottom-sheet.
- Busca permanece no topo, sempre visível.
- Seleção múltipla: long-press ou checkbox por card.

NÃO fazer scroll horizontal de tabela em mobile — é a primeira coisa que cliente reclama. Cross-check com [[mobile-audit]].

## Procedimento

### Pra tela nova

1. Inventário do que a tela precisa exibir.
2. Identifica a coluna principal (alvo da busca livre).
3. Identifica filtros estruturados (>5 valores únicos = filtro; <5 = não vale).
4. Decide: server-side ou client-side?
   - Server se >100 linhas esperadas / dados podem crescer / multi-tenant precisa filtrar `escola_id` no backend ([[tenant-audit]]).
   - Client se <50, dados estáticos, tela admin restrita.
5. Implementa primeiro **sem features opcionais** — confirma que o básico funciona, depois adiciona.

### Pra tela legada

1. Compara contra o padrão — quais componentes obrigatórios faltam?
2. Lista issues priorizadas:
   - 🔴 sem busca + sem filtro → usuário não acha nada
   - 🟠 sem ordenação → frustra
   - 🟡 sem paginação client-side mas server traz tudo → lento
   - 🟢 sem export → nice-to-have
3. Refator incremental — não tenta substituir tabela inteira de uma vez.

## Anti-padrões

- Múltiplas formas de buscar (busca topo + busca por coluna + filtro escondido) — escolher uma.
- Filtro que não persiste no URL — usuário recarrega e perde. Use query string (`?status=ativo&periodo=2026`).
- Paginação server-side mas count total faltando — usuário não sabe quantas páginas.
- Tabela com 30+ colunas em desktop **sem** opção de esconder.
- Ordenação que ordena só a página atual (deveria reordenar todo o dataset).
- Mobile = mesma tabela com `overflow-x: scroll` — quase sempre ruim.
- Confirmar deletar 100 itens com um único alerta genérico — mostrar quais.
- Click na linha inteira E também no botão "Ações" — event bubbling mata.

## Casos específicos do Lumied

### Tabela de alunos
- Busca por nome (parcial) + RA + responsável.
- Filtros: turma, ano letivo, status (ativo/inativo/transferido).
- Click na linha → ficha do aluno.

### Tabela de contas a receber/pagar
- Busca por descrição + valor + sacado.
- Filtros: status (aberto/pago/vencido), período de vencimento, banco.
- Seleção múltipla → ações em lote (baixar pago manual, gerar boleto, enviar cobrança).
- Cross-check [[project_construfare_conciliacao_v2]] que tem bulk ops.

### Tabela de manutenção / almoxarifado
- Busca + filtro por categoria + status (aberto, em andamento, fechado).
- Mobile crítico — funcionários em campo abrem no celular.

### Painel admin (cron-health, reconcile, backups)
- Tabelas dense, técnicas, em desktop. Mobile fallback opcional.
- Refresh automático (polling 30s+) pra acompanhar status.

## Output esperado da auditoria

```
## Tabela: alunos.html

🔴 Bloqueios:
- Sem busca livre — usuário tem que scrollar 800 linhas
- Filtro de turma não persiste no URL

🟠 Nits:
- Mobile: scroll horizontal feio
- Sem contagem total no rodapé

🟢 OK:
- Ordenação por nome funciona
- Loading state OK
- Click na linha abre ficha
```
