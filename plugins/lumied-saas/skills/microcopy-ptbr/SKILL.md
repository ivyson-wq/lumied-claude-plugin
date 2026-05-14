---
name: microcopy-ptbr
description: Guia de microcopy PT-BR para o Lumied — tom/voz, mensagens de erro humanizadas, CTAs claros, formatação BR de datas/números/moeda. Materializa [[feedback_limitacoes_visiveis]] (limitação tem que ser explicada) e [[feedback_incidentes_internos]] (cuidado com quem é o público). Use ao revisar texto de UI, criar mensagem de erro, ou quando o usuário disser "esse texto tá ruim", "fica melhor como?".
---

# Microcopy PT-BR — Lumied

Contexto: público do Lumied é heterogêneo — direção (formal), secretaria (operacional), professores (varia), famílias e alunos (público leigo). Cada texto na UI precisa ajustar tom ao público. E erros técnicos vazando direto da API ("ECONNREFUSED") são piores que mensagem em branco. Esta skill define o padrão.

## Quando rodar

- Texto novo na UI (botão, label, erro, tooltip).
- Revisão de tela legada com texto datado.
- Mensagens de e-mail, push, WhatsApp pro responsável.
- Após [[postmortem]] — comunicação ao cliente sobre incidente.

## Tom & voz

### Tom geral: claro, direto, respeitoso, sem inflar.

- ✔ "Aluno cadastrado com sucesso."
- ✘ "Show! 🎉 Mais um aluno na família Lumied! ✨"
- ✘ "O cadastro do aluno foi efetivamente realizado e armazenado em nossa base de dados."

### Por público

| Público | Tom | Exemplo |
|---|---|---|
| Direção / Superadmin | Formal, técnico OK | "Habilite RLS antes de migrar dados sensíveis." |
| Secretaria / Tesouraria | Direto, sem jargão | "Boleto gerado. Será enviado ao responsável em até 5 min." |
| Professor | Direto, foco no aluno | "Nota lançada para 3 alunos. Faltam 12 na turma 5A." |
| Família / Responsável | Cordial, sem termo técnico | "Mensalidade de março paga. Próximo vencimento: 10/04." |
| Aluno | Curto, motivacional moderado | "Atividade entregue. Aguarde correção." |

### Pessoa do verbo

- **Tu/Você**: sempre "você" (BR padrão).
- **Imperativo educado**: "Confirme o pagamento" > "Por favor, confirme o pagamento" (excesso de "por favor" infantiliza).
- **Sistema fala em primeira pessoa do plural** quando admite limite: "Não conseguimos carregar..."

## Mensagens de erro

### Padrão de 3 camadas

```
[Título curto: o que aconteceu]
[Subtítulo: por que / impacto]
[Ação: o que fazer agora]
```

### Bons exemplos

```
✔ Boleto não foi gerado
   O banco rejeitou o CNPJ informado.
   [Editar dados da escola]  [Tentar novamente]
```

```
✔ Você não tem acesso a esta tela
   Esta funcionalidade é restrita ao superadmin.
   [Voltar ao painel]  [Solicitar acesso →]
```

### Anti-padrões

- ✘ "Erro" / "Erro 500" / "Algo deu errado" — sem informação.
- ✘ "ECONNREFUSED 127.0.0.1:5432" — erro técnico cru.
- ✘ "Por favor, contate o administrador do sistema." — sem caminho.
- ✘ "Operação falhou. Tente novamente." — sem motivo, infinito loop.
- ✘ Pedir desculpa em excesso ("Desculpe imensamente o inconveniente...") — soa enganoso.

### Stack trace → mensagem humana

| Erro técnico | Mensagem ao usuário |
|---|---|
| `duplicate key value violates unique constraint` | "Já existe um aluno com este RA. Verifique antes de cadastrar." |
| `null value in column "escola_id"` | "Não foi possível identificar a escola. Faça login novamente." |
| `permission denied for table alunos` (RLS) | "Você não tem acesso a estes alunos." |
| `JWT expired` | "Sua sessão expirou. Faça login novamente." |
| `Network error` / timeout | "Conexão lenta. Tente novamente em alguns instantes." |

## Formatação BR

- **Data:** `DD/MM/AAAA` (08/03/2026). Em e-mail: "08 de março de 2026".
- **Hora:** `HH:mm` 24h (14:30). Não usar AM/PM.
- **Número:** separador de milhar `.`, decimal `,` (1.234,56).
- **Moeda:** `R$ 1.234,56` com espaço entre R$ e valor.
- **CPF/CNPJ:** sempre formatado com pontos/traços/barra (123.456.789-00, 12.345.678/0001-90).
- **Telefone:** `(54) 99999-0000` (DDD entre parênteses).
- **CEP:** `95000-000`.

## CTAs (botões)

### Verbo no infinitivo + objeto

- ✔ "Cadastrar aluno", "Gerar boleto", "Enviar cobrança", "Voltar"
- ✘ "OK", "Cancelar" (genéricos demais quando há alternativa específica)
- ✘ "Clique aqui", "Aqui" (sem sujeito do verbo)

### Pares de ação (modal/diálogo)

| Pergunta | Botão primário | Botão secundário |
|---|---|---|
| "Excluir aluno?" | "Excluir" (vermelho) | "Cancelar" |
| "Salvar alterações?" | "Salvar" | "Descartar" |
| "Enviar boleto por e-mail?" | "Enviar" | "Cancelar" |

Botão destrutivo tem texto **explícito** (Excluir, Apagar, Desativar), nunca "OK".

## Notificações / toasts

- **Success:** "Boleto enviado." (passado, conciso)
- **Info:** "Aguardando confirmação do banco." (presente)
- **Warning:** "Mensalidade vence amanhã." (factual)
- **Error:** padrão de 3 camadas (acima).

## E-mail / WhatsApp ao responsável

Padrão Lumied (cordial, sem promo):

```
Olá, [Nome do responsável].

A mensalidade de [Aluno] do mês de [Mês/Ano] está disponível.

Valor: R$ 850,00
Vencimento: 10/04/2026

[Pagar com PIX]  [Baixar boleto]

Em caso de dúvida, responda este e-mail ou fale com a secretaria.

— [Nome da escola]
```

Evitar:
- "Prezado(a) responsável" — frio
- "Estamos felizes em informar..." — inflar
- "Não responda este e-mail" se o e-mail aceita resposta — mentira

Cross-check [[feedback_incidentes_internos]]: alertas internos (bloqueio físico) **não** vão pra família.

## Anti-padrões gerais

- Emoji em UI séria (financeiro, RH, manutenção). Aceitável em comunicação pra aluno (gamificação leve).
- Texto em UPPER CASE pra ênfase — grita. Use peso da fonte.
- Inglês misturado ("Atualizar profile", "Fazer login"). Padroniza: "Atualizar perfil", "Entrar".
- Abreviar palavras pra "caber" ("Mens." em vez de "Mensalidade") — soa preguiçoso.
- "Em breve" sem prazo — vazio.
- Tom impessoal pro responsável ("O usuário deve...") — soa de manual.

## Output da auditoria

```
## Tela: cobranca-modal.html

🔴 Bloqueios:
- L42: "Erro ao gerar" → "Boleto não foi gerado. O banco rejeitou o CNPJ informado."
- L67: "Por favor, tente novamente." → especificar **o que** tentar de novo
- L89: botão "OK" no modal de exclusão → "Excluir"

🟠 Nits:
- L23: "12345" sem formatação → "R$ 12.345,00"
- L55: "ECONNREFUSED" vazando → mapa de erro técnico→humano

🟢 OK:
- Tom adequado ao público (secretaria)
- CTAs em infinitivo
```
