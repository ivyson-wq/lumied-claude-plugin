---
name: erp-import-runbook
description: Runbook executável da migração assistida (paga) de ERP escolar pro Lumied — Escolaweb, Sponte, WPensar, Sophia, TOTVS, GVDasa. Diferente do [[erp-importer-template]] que só gera o scaffold de código; este é o procedimento operacional pra quando o cliente já assinou e a janela de corte chegou. Use quando o usuário disser "migrar a escola X do ERP Y", "executar import do Sponte", "começar sprint de migração".
---

# Runbook — Migração assistida de ERP

Contexto: [[project_migracao_erps]] define 7 sprints (1 por ERP, paralelo possível depois do primeiro). Cada migração assistida é **paga na implantação** e inclui histórico financeiro completo, não só master data. O scaffold de código vem de [[erp-importer-template]]. Este runbook é o que se faz **no dia da migração** + na semana antes e depois.

## Quando rodar

- Cliente assinou plano + migração paga.
- Tenant criado via [[escola-onboarding]].
- Janela de corte combinada (idealmente fim-de-semana ou recesso).
- Importer do ERP específico já existe (scaffold + customização do cliente testada em sandbox).

NÃO usar pra:
- ERP novo sem importer ainda — primeiro [[erp-importer-template]], homologa em sandbox, **depois** roda este runbook.
- Cliente sem pagar implantação — onboarding básico, importação manual pelo cliente.

## Pré-janela (D-7 a D-1)

### D-7: pré-validação dos dados do ERP de origem

- [ ] Cliente exportou o backup do ERP (ou nos deu acesso de leitura).
- [ ] Volumes: quantos alunos, turmas, lançamentos financeiros, ano(s) letivo(s)?
- [ ] Validamos estrutura: campos obrigatórios presentes? Caracteres especiais OK? Encoding UTF-8?
- [ ] Comparamos schema do ERP contra mapeamento do importer — alguma coluna inesperada?

### D-3: dry-run em sandbox

- [ ] Importar 10% dos dados num tenant `<escola>-sandbox`.
- [ ] Cliente revisa: nomes, valores, datas batem com o ERP de origem.
- [ ] Discrepância < 1% nos totais financeiros (somatório CR + CP).
- [ ] Performance: 10% levou X minutos → projeção pro 100% cabe na janela.

### D-1: congelar dados de origem

- [ ] Cliente para de lançar no ERP antigo (combinar hora exata, ex: "sexta 18h").
- [ ] Backup definitivo do ERP feito (cliente + nós, redundância).
- [ ] Export final extraído.

## Dia D (janela de corte)

### 1. Snapshot do tenant de destino

```sql
-- Backup do estado atual do tenant no Lumied (caso precise reverter)
-- Usar mecanismo de [[project_backups]] focado neste escola_id.
```

### 2. Aplicar migrations específicas se houver

Se este ERP precisa de coluna/tabela extra (raro, mas acontece — campos legados que viram metadata):
- [ ] Migration está pronta + tem rollback ([[migration-rollback]]).
- [ ] Aplica via Management API ([[reference_lumied_supabase]]).

### 3. Importar dados

Ordem importa (referential integrity):
1. Escolas → já existe via [[escola-onboarding]].
2. Períodos letivos / anos.
3. Cursos / séries / turmas.
4. Pessoas: responsáveis, alunos, funcionários.
5. Matrículas (linha entre aluno × turma × ano).
6. Plano de contas / categorias financeiras.
7. Contratos / mensalidades.
8. Histórico financeiro: contas_receber + contas_pagar + lançamentos pagos.
9. Acadêmico: notas, faltas, ocorrências (se contratado).
10. Anexos / docs.

### 4. Validar pós-import

- [ ] Totais financeiros: soma CR + CP no Lumied = soma no ERP de origem (delta < 1%).
- [ ] Contagem de alunos ativos: bate.
- [ ] Login do superadmin abre a tela com dados visíveis.
- [ ] Spot check: 5 alunos aleatórios, conferir manualmente nome/turma/valor da mensalidade.
- [ ] Spot check: 5 lançamentos pagos no ERP devem aparecer como pagos no Lumied.
- [ ] Tenant isolation: outro tenant **não** vê esses dados ([[tenant-audit]] manual).
- [ ] RLS habilitada nas tabelas tocadas ([[rls-check]]).

### 5. Smoke test funcional

- [ ] Família consegue logar e ver a mensalidade aberta correta.
- [ ] Tesouraria consegue baixar um pagamento.
- [ ] Professor consegue ver sua turma e lançar nota.
- [ ] Emissão de boleto novo funciona ([[bank-homologacao]] se banco mudou).

### 6. Comunicar cutover

- [ ] Cliente confirma "tudo OK, podemos abrir pros usuários".
- [ ] E-mail/comunicado pros responsáveis: "a partir de segunda usem o portal X".

## Pós-janela (D+1 a D+7)

- [ ] Monitorar tickets na Central de Ajuda ([[project_ajuda_pendente]]).
- [ ] Conferir cron de notificações + boletos disparou corretamente no primeiro ciclo.
- [ ] Se [[project_sienge_reconcile]]-like for relevante: cobertura inicial?
- [ ] Coletar feedback do cliente — alimenta playbook pro próximo sprint.

## Rollback (se algo der MUITO errado)

- Restore do snapshot do tenant de destino (passo 1).
- Cliente volta a usar o ERP antigo (por isso o backup do ERP em D-1 é crítico).
- [[postmortem]] obrigatório.

Casos onde rollback é necessário:
- Dados financeiros importaram com valores zerados / errados em massa.
- Tenant isolation furou (vazamento — STOP imediato).
- Performance no Lumied virou inutilizável (>5s por página) com o volume importado.

## Anti-padrões

- Pular dry-run em sandbox — "vai dar certo" não é estratégia.
- Importar com cliente lançando no ERP em paralelo (precisa congelar D-1).
- Não validar totais financeiros (cliente descobre 30 dias depois que falta R$ X mil).
- Importar fora de janela acordada (família tentando pagar e sistema dando erro).
- Cobrar pela migração e deixar [[escola-onboarding]] incompleto (módulo não ativado).
- Esquecer [[migration-rollback]] preparado pras migs específicas do importer.

## Por ERP — particularidades conhecidas

### Escolaweb
- Tier alvo: Starter R$ 790 ([[project_tier_starter]]) — anti-Escolaweb.
- Export geralmente em CSV. Encoding pode ser Latin1.

### Sponte
- Mais complexo (financeiro robusto). Stress-test em sandbox.
- API REST disponível pra alguns clientes — preferir API a CSV.

### WPensar / Sophia / TOTVS / GVDasa
- A documentar à medida que cada sprint roda. Adicionar achados aqui após cada migração.

## Pós-migração: virar memória

- Criar memória `project_migracao_<escola>_<erp>.md` se houve aprendizado novo.
- Atualizar [[project_migracao_erps]] com status do sprint.
- Se houve incidente: [[postmortem]].
