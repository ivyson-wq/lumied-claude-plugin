---
name: postmortem
description: Template guiado pra documentar incidente (tenant leak, cron quebrado, secret vazado, IO/Storage estourado, banco fora). Captura impacto, timeline, causa-raiz, ações imediatas e estruturais. Use quando algo quebrou e dá pra extrair aprendizado — virá memória [[project_*]] depois.
---

# Postmortem template — Lumied / Construfare

Contexto: incidentes acontecem ([[project_tenant_isolation_incident]], [[project_lumied_io_storage]], [[project_backlog_v2_pendencias]] menciona backup quebrado há 22d). Sem documentação, repetem. Esta skill produz um doc curto, focado em aprendizado — não burocracia.

## Quando rodar

- Acabou de resolver um incidente.
- Descobriu que algo está quebrado há dias/semanas.
- Cliente reportou bug com impacto real (dado errado mostrado, função fora do ar).
- Quase-incidente (poderia ter vazado, mas pegou antes).

NÃO usar pra:
- Bug minúsculo de UI sem impacto.
- Mudança planejada de arquitetura.

## Template

Salvar em `docs/postmortems/YYYY-MM-DD-<slug>.md` no repo, OU como memória `project_*` se for crítico pro futuro Claude.

```markdown
# Postmortem: <título curto e descritivo>

**Data do incidente:** YYYY-MM-DD HH:MM BRT
**Data deste doc:** YYYY-MM-DD
**Status:** resolvido / mitigado / em aberto
**Severidade:** 🔴 crítico / 🟠 alto / 🟡 médio / 🟢 baixo
**Autor:** Ivyson

## TL;DR (3 linhas)

O que aconteceu, qual o impacto, o que foi feito.

## Impacto

- Usuários afetados: quem (escolas, papéis, número aproximado).
- Dados afetados: lidos errado / escritos errado / perdidos / expostos.
- Janela: de YYYY-MM-DD HH:MM a YYYY-MM-DD HH:MM (X horas/dias).
- Externo: cliente notou? mídia? LGPD?

## Timeline

- HH:MM — gatilho/deploy/evento que iniciou.
- HH:MM — primeiro sintoma observável (alerta, reclamação, log).
- HH:MM — detecção (como descobrimos).
- HH:MM — diagnóstico inicial.
- HH:MM — mitigação aplicada.
- HH:MM — resolução completa / fim do impacto.

(Use times reais, não estimativas.)

## Causa-raiz

A causa **técnica direta** + a causa **sistêmica** (por que isso era possível).

Exemplo:
- Técnica: edge function `boletins_publicar` não filtrava `escola_id` na query.
- Sistêmica: não existia audit obrigatório pré-deploy nesse momento — o [[tenant-audit]] foi criado *por causa* deste tipo de incidente.

## O que funcionou

- Detecção rápida? Alerta funcionou? Backup existia?
- Pessoas que ajudaram, ferramentas que economizaram tempo.

## O que NÃO funcionou

- Alerta atrasou. Backup era inconsistente. Doc estava errado.
- Sem julgar pessoas — focar em sistema.

## Ações tomadas (imediatas)

- [x] Patch aplicado (commit `abc123`).
- [x] Comunicado clientes afetados em DD/MM.
- [x] Mig de rollback aplicada.

## Ações estruturais (prevenção)

Pra **não acontecer de novo** — não só "ter mais cuidado". Mecânica.

- [ ] Adicionar check X no `deploy-preflight`.
- [ ] Criar skill `Y` (vide [[skill-name]]) que detecta esse padrão.
- [ ] Migration N que força constraint.
- [ ] Alerta em Sentry/UptimeRobot pra esse caminho específico.
- [ ] Update doc Z.

Prazo + responsável pra cada item.

## Memórias a criar/atualizar

Após este postmortem, considerar:
- Criar memória `project_<incidente>.md` se há contexto a preservar pro futuro.
- Atualizar memória existente (ex: [[project_tenant_isolation_incident]] ganhou novas migs).
- Criar memória `feedback_<lição>.md` se virou regra de comportamento.

## Anexos

- Logs relevantes (links pra Supabase logs, Sentry).
- Diffs do fix.
- Screenshot do erro reportado pelo cliente.
- Query SQL que mostra o estrago.
```

## Princípios

1. **Sem culpa.** "Aluno não foi notificado" — não "Fulano esqueceu de adicionar". Foco em sistema.
2. **Curto.** Postmortem de 200 linhas ninguém lê. Mire em 30-80 linhas.
3. **Ações estruturais > prometer cuidado.** "Vou prestar mais atenção" é zero valor. "PR template agora exige checkbox de RLS" é valor.
4. **Datas absolutas.** "Há 3 dias" → "2026-05-11". Postmortems envelhecem.
5. **Linkar memórias.** Use `[[project_*]]` / `[[feedback_*]]` pra conectar com o resto da base.

## Casos específicos do Lumied

### Tenant leak (visual ou de dados)

Sempre 🔴 crítico, sempre LGPD-relevant. Ações estruturais devem incluir:
- Audit em todas as queries similares no codebase.
- Mig de RLS reforçada.
- Skill `tenant-audit` / `rls-check` atualizada se o padrão era novo.

### Cron quebrado há dias

- Por que não tem alerta? (Adicionar em [[cron-health]] se faltava).
- Backup-escolas é o exemplo canônico (22d quebrado).

### Secret vazado em repo público

- Rotação imediata ([[secret-rotation]]).
- `secrets-scan` ganhou novo padrão?
- Avaliar se repo pode voltar privado antes ([[project_private_after_reset]]).

### Banco/IO estourado

- Já temos [[project_lumied_io_storage]] como pattern. Atualizar com novos achados.
- Capacity planning: precisamos antecipar?

### Vercel deploy cap

- [[feedback_agrupar_commits]] + [[vercel-budget]] devem ter pego — por que não pegaram?
- Threshold de alerta funcionou?
