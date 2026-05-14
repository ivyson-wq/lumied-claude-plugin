---
name: print-layout
description: Padrão de layout pra impressão no Lumied — boletos, espelhos de ponto, diplomas, atestados, relatórios financeiros, declarações. A4, margens, page-break, sem cor desnecessária, tinta econômica. Use ao criar tela que vai ser impressa, ou quando o usuário disser "imprime esse relatório", "tá quebrando a página feio".
---

# Print layout — Lumied

Contexto: escola imprime muito. Boletos (banco/fallback), espelhos de ponto (compliance trabalhista), diplomas, atestados de matrícula, declarações, relatórios financeiros, listas de chamada. Cada um tem requisitos diferentes. UI feita pra tela quase nunca imprime bem. Esta skill define os padrões e checa o que falta.

## Quando rodar

- Tela nova que precisa imprimir / exportar PDF.
- Cliente reportou "saiu cortado", "veio em 10 páginas", "imprime tudo colorido e gasta tinta".
- Auditoria periódica em telas legadas que viraram impressas.

## Princípios

1. **A4 portrait** é o default (210mm × 297mm). Landscape só se conteúdo exige.
2. **Margens:** 1.5cm a 2cm em todos os lados — espaço pra grampo/furo.
3. **Tinta econômica:** preto + cinza em fundos. Cor só se essencial (logos, gráficos).
4. **Sem JS no print:** assume documento estático.
5. **Page-break consciente:** tabela não quebra no meio de uma linha; cabeçalho repete em cada página.
6. **Versão pra impressão ≠ versão da tela.** Não basta `@media print {}`; pode precisar de rota dedicada.

## CSS print baseline

```css
@media print {
  /* Reset agressivo de cores e shadows */
  * {
    background: transparent !important;
    box-shadow: none !important;
    color: black !important;
  }

  /* Esconde UI não-imprimível */
  nav, header, footer.app-footer,
  .no-print, button, .botoes {
    display: none !important;
  }

  /* Page setup */
  @page {
    size: A4 portrait;
    margin: 1.5cm 2cm;
  }

  /* Tipografia para impressão */
  body {
    font-family: Arial, Helvetica, sans-serif;
    font-size: 10pt;
    line-height: 1.4;
  }

  /* Evita corte ruim */
  h1, h2, h3, table, .bloco-aluno {
    page-break-inside: avoid;
  }

  thead {
    display: table-header-group; /* repete cabeçalho */
  }

  tfoot {
    display: table-footer-group;
  }

  /* Quebra forçada quando precisa */
  .page-break {
    page-break-after: always;
  }

  /* Links: mostra a URL */
  a[href]:after {
    content: " (" attr(href) ")";
    font-size: 8pt;
    color: #555 !important;
  }

  /* Mas não pra links internos */
  a[href^="#"]:after,
  a[href^="javascript:"]:after {
    content: "";
  }
}
```

## Padrões por tipo de documento

### Boleto bancário

- Já vem do banco geralmente como PDF pronto. Lumied só faz download/exibe.
- Se temos fallback (boleto Lumied híbrido), seguir layout FEBRABAN — ficha de compensação + recibo do sacado.
- Cross-check [[bank-homologacao]].

### Espelho de ponto

- Cabeçalho: nome da empresa + CNPJ + nome do funcionário + CPF + período + cargo.
- Tabela: dia, entrada, almoço saída, almoço retorno, saída, total horas, saldo, justificativa.
- Rodapé: linha pra assinatura do funcionário + assinatura do RH + data.
- 1 página por funcionário (page-break entre).
- Cross-check [[project_ponto_afd]].

### Diploma

- A4 landscape geralmente.
- Borda decorativa (essa é a única exceção pra cor/elementos visuais).
- Nome do aluno em destaque (centro, ~30pt).
- Período/curso/data de conclusão.
- Assinaturas (direção + secretaria).
- Selo / brasão da escola.
- 1 diploma = 1 página. Sem rodapé "página 1 de N".

### Atestado de matrícula / declaração

- Cabeçalho da escola (logo + endereço + CNPJ + telefone).
- Corpo formal: "Declaramos que o aluno [Nome], RA [X], está regularmente matriculado..."
- Local + data + assinatura.
- 1 página, geralmente.

### Relatório financeiro (mensal/anual)

- Cabeçalho: escola + período + data de emissão.
- Sumário: total receitas / total despesas / saldo (compacto, topo).
- Detalhamento: tabelas por categoria / centro de custo.
- Cabeçalho de tabela repete em cada página.
- Rodapé: "Página X de Y" + data de emissão.
- Numerar páginas.

### Lista de chamada / pauta

- Cabeçalho: turma + data + professor + disciplina.
- Tabela: nome do aluno, RA, espaço pra marcar P/F.
- Linhas espaçadas o suficiente pra escrever à mão.
- Densidade: caber 30-40 alunos em A4.

## Procedimento

1. Identificar o tipo do documento (boleto / espelho / relatório / etc.).
2. Conferir se tem CSS `@media print` ou rota dedicada `/imprimir/...`.
3. Testar:
   - Imprimir via Chrome (Ctrl+P) → preview.
   - Salvar como PDF → conferir tamanho, paginação, margens.
   - Cliente: testar em impressora real ao menos uma vez (HP/Epson/Brother — papel A4).
4. Conferir checklist:
   - [ ] Margens OK
   - [ ] Sem cor desnecessária
   - [ ] Cabeçalho de tabela repete em cada página
   - [ ] Page-break não corta linha
   - [ ] Sem botões / nav visíveis
   - [ ] "Página X de Y" se múltiplas páginas
   - [ ] Logo da escola se aplicável
   - [ ] Mobile não importa aqui — print não vem do mobile
5. Reporte:
   ```
   ## Tela: relatorio-financeiro.html

   🔴 Bloqueios:
   - Botões "Editar" e "Exportar" aparecem na impressão (faltou .no-print)
   - Cabeçalho da tabela não repete em página 2+
   - Tabela quebra no meio de uma linha (sem page-break-inside)

   🟠 Nits:
   - Fundo cinza claro nos zebras → desperdiça tinta
   - Link "Ver detalhes" exibe URL em pequeno (OK, mas verboso)

   🟢 OK:
   - Margens A4
   - Numeração de página
   - Cabeçalho da escola
   ```

## Anti-padrões

- Reusar a página da tela sem versão de print — sai com sidebar, header app, anúncios.
- Cor de fundo escura em bloco grande — gasta tinta.
- Fontes muito pequenas (<9pt) — não lê.
- Tabela com 20 colunas em portrait — não cabe.
- Imagens com `transform: rotate(...)` — alguns navegadores ignoram em print.
- Forçar `<canvas>` ou `<svg>` enorme — impressora trava ou demora.
- Esquecer rodapé com data + nome do operador (compliance — quem imprimiu/quando).

## Página dedicada vs media query

- **Media query (@media print):** suficiente pra documentos simples (lista de chamada, atestado).
- **Rota dedicada (/imprimir/<id>):** melhor pra documentos complexos (relatório financeiro, espelho de ponto) — controle total do layout, sem amarrar à UI da tela.

## Pós-revisão

- Se for documento legal (espelho de ponto, diploma, atestado), guardar versão "padrão ouro" como PDF de referência no repo `docs/print-layouts/`.
- Se mudar layout, abrir [[postmortem]] light se cliente reclamar.
