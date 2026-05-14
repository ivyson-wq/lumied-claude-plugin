---
name: empty-states
description: Audita ou aplica padrão de estado vazio no Lumied — primeira tela após onboarding, lista zerada, busca sem resultado, erro de carregamento. Hoje muitas telas mostram só `[]` ou tela em branco. Use ao criar tela com listagem, no smoke test pós [[escola-onboarding]], ou quando o usuário disser "tela tá branca", "sem feedback".
---

# Empty states — Lumied

Contexto: empty state é a primeira impressão do usuário num módulo recém-ativado. Tenant novo via [[escola-onboarding]] entra em "Alunos" e vê uma tabela vazia. Sem CTA, sem explicação, sem caminho — usuário acha que está quebrado. Esta skill catalogou os tipos de empty state e o que cada um precisa.

## Quando rodar

- Tela nova com listagem (sempre).
- Pós [[escola-onboarding]] smoke test — abrir cada módulo e ver o que aparece.
- Após [[loading-states]] — empty é o "depois" do loading que termina com zero.
- Usuário reclamou "abri e não tinha nada", "achei que tava quebrado".

## Os 4 tipos de empty state

### 1. First-time empty (tenant novo, nunca usou)

Usuário acabou de ganhar acesso ao módulo e nunca cadastrou nada. **Precisa de CTA forte pra começar.**

```
┌──────────────────────────────────────────────┐
│                                              │
│        📚  (ilustração / ícone grande)       │
│                                              │
│       Nenhum aluno cadastrado ainda          │
│                                              │
│   Comece criando manualmente ou importe      │
│   do seu ERP atual em poucos minutos.        │
│                                              │
│   [+ Cadastrar aluno]  [↑ Importar do ERP]   │
│                                              │
│       Saiba mais na Central de Ajuda →       │
│                                              │
└──────────────────────────────────────────────┘
```

Elementos:
- Ilustração ou ícone (não foto stock genérica).
- Título: descritivo, não decorativo ("Nenhum aluno cadastrado ainda" > "Tudo vazio").
- Subtítulo: explica próximo passo.
- 1-2 CTAs primárias (criar manual + importar).
- Link secundário pra ajuda ([[project_ajuda_pendente]]).

### 2. Filtered empty (tem dados, filtro escondeu tudo)

Usuário aplicou filtro e nada apareceu. **Diferente do first-time** — não é "vazio", é "filtro muito restritivo".

```
┌──────────────────────────────────────────────┐
│                                              │
│    🔍  Nenhum resultado para "joão silva"    │
│                                              │
│    Tente:                                    │
│    • verificar a ortografia                  │
│    • remover filtros (Turma: 5A, Status: ativo) │
│    • busca por RA em vez de nome             │
│                                              │
│       [Limpar filtros]                       │
│                                              │
└──────────────────────────────────────────────┘
```

Elementos:
- Mostra **o que** foi buscado/filtrado.
- Sugestões acionáveis.
- Botão "Limpar filtros" óbvio.

### 3. Permission empty (sem acesso)

Usuário não tem role pra ver. **Não fingir que está vazio** — explicar.

```
🔒  Você não tem acesso ao módulo "Folha de Pagamento".

   Peça ao superadmin da sua escola pra liberar
   o acesso, ou volte ao painel inicial.

   [← Voltar ao painel]
```

Cross-check [[edge-fn-authz]] — se o front mostra "vazio" mas o back devolveu 403, é bug. Front deve mostrar **esta tela**, não a tela de "first-time".

### 4. Error empty (falhou ao carregar)

Backend deu erro. **Não mostrar tabela vazia** — mostrar erro com retry.

```
⚠️  Não conseguimos carregar os alunos.

   Tente novamente em alguns instantes.
   Se o problema persistir, contate o suporte.

   [↻ Tentar novamente]   [Reportar problema →]
```

Materializa [[feedback_limitacoes_visiveis]]. Cross-check [[loading-states]] (tipo "error state").

## Procedimento

Pra cada tela com listagem, garantir que os 4 tipos estão tratados:

```markdown
## Tela: alunos.html

| Cenário                              | Status |
|--------------------------------------|--------|
| Tenant novo (0 alunos)               | 🔴 mostra tabela vazia, sem CTA |
| Busca sem resultado                  | 🔴 mostra "Nenhum resultado" sem sugestões |
| Sem permissão                        | 🟢 mostra erro 403 com mensagem |
| Erro de carregamento                 | 🔴 mostra tabela vazia, sem aviso |

Bloqueios:
1. Adicionar empty state first-time com CTA "+ Cadastrar aluno" e "↑ Importar".
2. Adicionar mensagem "Nenhum resultado para X" + botão limpar filtros.
3. Tratar erro de fetch → mostrar tela de retry.
```

## Princípios

1. **Empty state é uma TELA, não um espaço em branco.** Tem que ocupar a área visualmente.
2. **CTA primária acionável.** "Comece criando" é melhor que "Você não tem alunos".
3. **Diferenciar os 4 tipos.** Tenant novo ≠ filtro vazio ≠ sem permissão ≠ erro.
4. **Não usar humor/decoração demais.** Tom Lumied é profissional, não startup descontraído.
5. **Linkar Central de Ajuda quando faz sentido** ([[project_ajuda_pendente]]).
6. **Mobile:** mesmo padrão, ajustado pra largura. Ilustração menor, CTA full-width.

## Anti-padrões

- Empty state genérico ("Nada por aqui...") em todas as telas — perde valor.
- Mostrar `0 resultados` sem distinguir entre filtro restritivo e dataset vazio.
- Pedir pro usuário "atualizar a página" — feio. Faz fetch automaticamente.
- Empty state com texto sobre o produto Lumied (marketing) em vez de orientar o próximo passo.
- Sem CTA — usuário vê "lista vazia" e fica perdido.
- Loading infinito quando deveria ser empty state.

## Casos específicos

### Após [[escola-onboarding]]
- Toda primeira tela de módulo ativo precisa de first-time empty state com CTA específico daquele módulo.
- Exemplo: módulo financeiro recém-ativado → "Cadastre o plano de contas" + "Importe do ERP".

### Após [[erp-import-runbook]]
- Se import falhou parcial, mostrar **status** ("12 alunos importados, 3 com erro — ver detalhes") em vez de empty.

### Em painéis admin (cron-health, reconcile)
- Empty real = bom sinal ("Nenhum cron quebrado", "Nenhuma divergência"). Texto deve **celebrar**, não alarmar.
