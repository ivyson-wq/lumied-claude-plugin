---
name: tuya-onboarding
description: Runbook de onboarding Tuya por escola — Cloud Project único compartilhado entre todos os tenants, 1 conta Smart Life por escola, vinculação via "Link Devices by App Account" e setup do tuya_space_id na ir_politicas. Use ao provisionar climatização IR pra escola nova, ou quando o usuário disser "como ligar Tuya nessa escola", "onboarding climatização", "vincular hub novo", "criar projeto Tuya". Complementa [[escola-onboarding]] e materializa [[project_climatizacao_ir]].
---

# Onboarding Tuya por escola — Lumied climatização IR

Contexto: Lumied controla ACs de sala de aula via hubs Novadigital SRW-TL + Tuya Cloud OpenAPI ([[project_climatizacao_ir]]). A arquitetura é **1 Cloud Project Tuya único pra toda a Lumied** + **N contas Smart Life vinculadas (1 por escola)**. Esta skill cobre o handoff completo pra cada escola nova entrar no sistema.

## Por que esse design (não muda)

- `TUYA_ACCESS_ID` e `TUYA_ACCESS_SECRET` são secrets globais da edge function — não cabe um par por escola sem refator pesado.
- `ir_politicas.tuya_space_id` (1 linha por escola) mapeia cada escola pro seu Home Tuya.
- `ir_setup_tuya_space` lista todos os spaces do Cloud Project e o gerente escolhe o dele.
- Tenant isolation no Lumied: cada query filtra por `escola_id`, hubs são únicos por `tuya_device_id`.

Se um dia precisar isolation físico (escolas concorrentes que exigem), refatorar pra `ir_tuya_credentials` por escola com Vault. Não fazer por padrão.

## Pré-requisitos antes de começar

- [ ] Escola já provisionada via [[escola-onboarding]] (`escola_id` existe).
- [ ] Módulo `climatizacao` ativado em `escolas_features` da escola.
- [ ] Hubs Novadigital SRW-TL fisicamente instalados na escola (1 por sala/ambiente).
- [ ] Wi-Fi 2.4 GHz disponível nas salas (SRW-TL não suporta 5 GHz).
- [ ] Senha do Wi-Fi compartilhada com TI da escola.
- [ ] Cloud Project Lumied no `iot.tuya.com` ativo (region `us`, secrets já no Supabase — verificar `TUYA_ACCESS_ID` em `gh secret list --env supabase` ou via dashboard).

## Passo a passo

### 1. Criar conta Smart Life dedicada da escola

- E-mail recomendado: `climatizacao@<dominio-da-escola>.com.br` (não usar conta pessoal do gerente).
- App: **Smart Life** (preferido) ou **Tuya Smart** (mesma backend, layouts diferentes).
- Plataforma: Android ou iOS, gratuitos.
- Anota credenciais em gerenciador de senhas da Lumied.

### 2. Parear hubs na conta Smart Life

Por cada hub Novadigital SRW-TL:

- [ ] Liga o hub na tomada.
- [ ] Botão reset por 5s até LED piscar rápido (modo pareamento EZ Mode).
- [ ] App → `+` → `Adicionar dispositivo` → categoria `Controle universal` → `SRW-TL` (ou aceita auto-detect).
- [ ] Informa SSID 2.4 GHz + senha → confirma.
- [ ] Renomeia pro padrão: `Sala 101`, `Sala 102`, `Auditório`, etc.

Falha comum: hub não conecta no Wi-Fi → confirmar que router está em 2.4 GHz e que MAC do hub não está bloqueado.

### 3. Cadastrar ACs no app (catálogo Tuya)

Por cada AC de cada hub:

- [ ] App → entra no hub → `Adicionar remoto` → `Ar-condicionado`.
- [ ] Escolhe a marca (Springer, LG, Samsung, Electrolux, Consul, Midea, Daikin, Fujitsu, Carrier, Komeco, Philco, Britânia, Hitachi…).
- [ ] Testa botão `Liga/Desliga` apontando hub pro AC — Tuya cicla por códigos compatíveis, confirma o que funcionou.
- [ ] Renomeia: `AC Sala 101 — Split Springer 12k`.

Se a marca/modelo não está no catálogo (raro hoje, ~95% mapeado): usa modo `Aprender` (aponta o controle remoto original pro hub, grava botão por botão). Mais trabalhoso, evita se possível.

Atalho: se 10 salas têm o **mesmo modelo de AC**, depois de cadastrar um, dá pra "Copiar remoto" no app — Tuya replica.

### 4. Vincular conta Smart Life ao Cloud Project Lumied (ADMIN LUMIED FAZ)

Só você (admin da Lumied) executa esse passo, login no `iot.tuya.com` com a conta-mãe:

- [ ] [iot.tuya.com](https://iot.tuya.com) → seleciona o Cloud Project Lumied.
- [ ] Menu lateral → `Devices` → aba `Link Tuya App Account`.
- [ ] Botão `Add App Account` → QR Code aparece.
- [ ] No celular da escola, **dentro do app Smart Life logado na conta da escola**: `Eu` (canto inf direito) → ícone QR no topo → escaneia o QR do navegador.
- [ ] Confirma autorização no app.
- [ ] Aguarda 30-60s — devices da escola aparecem na lista `Devices > All Devices` do Cloud Project.

Pega o `device_id` de cada hub: clica no device → copia campo `Device ID` (ex: `bf45a7e8c2d1f5...`).

### 5. Setup do space_id na escola (LUMIED — gerente da escola faz)

Pelo painel Lumied:

- [ ] Gerente da escola → portal Gerente → menu `Climatização` → primeira vez aparece prompt "Configurar Tuya".
- [ ] Clica "Configurar Tuya" → backend chama action `ir_setup_tuya_space` → lista todos os spaces visíveis no Cloud Project.
- [ ] Se há só 1 space na conta (caso normal): auto-seleciona e salva em `ir_politicas.tuya_space_id`.
- [ ] Se há múltiplos (raro — conta Smart Life com mais de uma "Casa"): mostra lista, gerente escolhe o nome certo.

Verificação SQL (admin):
```sql
SELECT escola_id, tuya_space_id, tuya_space_id IS NOT NULL AS configurado
FROM ir_politicas WHERE escola_id = '<uuid-escola>';
```

### 6. Cadastrar hubs no Lumied (gerente)

Pra cada hub físico:

- [ ] Painel Climatização → `+ Hub` → preenche:
  - Nome: `Sala 101`
  - `tuya_device_id`: cola o ID copiado do Cloud Project no passo 4
  - `sala_id`: vincula à série/sala cadastrada (FK em `series.id`)
  - `modelo`: `Novadigital SRW-TL` (default)
- [ ] Salva. Botão `Sincronizar` puxa estado online + temp + umidade atuais via `tuyaGetDeviceStatus`.

### 7. Descobrir e vincular dispositivos AC (gerente)

Pelo painel Lumied, dentro de cada hub:

- [ ] Botão `Descobrir remotes` → backend chama action `ir_dispositivo_discover_remotes`.
- [ ] Lista todos os ACs aprendidos no app pra esse hub (do passo 3).
- [ ] Auto-vincula se há 1 AC no Lumied e 1 remote AC no Tuya (categoria 5).
- [ ] Pra vários: gerente escolhe qual `ir_dispositivo` da UI bate com qual `tuya_remote_id`.
- [ ] Confirma → salva `tuya_remote_id` em `ir_dispositivos`.

### 8. Configurar política da escola (gerente)

Painel → `Política de climatização`:

- [ ] Setpoint padrão (ex: 22°C frio, 24°C quente).
- [ ] Horário operacional (ex: seg-sex 07:00-18:00).
- [ ] Pré-cooldown (ligar 15 min antes do horário).
- [ ] Ausência detectada → desliga após X min (default: 30).
- [ ] Janela de temperatura externa pra modo eco (>30°C força frio máx).
- [ ] Feriados sincronizados ([[feriados]] já trazidos da mig 372).

### 9. Smoke test ponta-a-ponta

- [ ] Painel → clica `Ligar` em 1 AC → confirma que AC físico ligou (ir até a sala).
- [ ] Painel → muda temperatura → AC obedece.
- [ ] Cria agendamento de 5 min no futuro → confirma que disparou.
- [ ] Espera 6 min → confirma que tuya_rule_id apareceu na `ir_agendamentos` (sync Tuya bem-sucedido).

Se algum falhar: checar `ir_eventos` filtrando pela escola — última linha mostra erro.

## Failsafes e troubleshooting

- **Bridge offline / internet caída na escola:** agendamentos espelhados no Tuya rodam direto do firmware do hub — escola não fica sem AC.
- **Hub mostra offline:** confirma Wi-Fi da escola, MAC do hub no router, e ping no `iot.tuya.com` da rede da escola.
- **Auth Tuya falha (HTTP 500 no Lumied):** rodar `node tuya-test.mjs` da Lumied com `TUYA_ACCESS_ID/SECRET` — se Tuya retorna 401, credencial expirou ou foi revogada no Cloud Project.
- **Devices não aparecem após Link App Account:** aguardar até 5 min, refresh do navegador, ou re-escanear QR.
- **`ir_setup_tuya_space` retorna "Nenhum space encontrado":** a conta Smart Life da escola não tem um "Home" criado — pedir pro TI da escola criar um Home no app (`Eu` → `Gerenciamento de família` → `Criar família`).
- **AC "liga" no painel mas não fisicamente:** `tuya_remote_id` errado, ou AC fora do alcance IR do hub (máx ~8m, sem obstrução).

## Custos e limites Tuya

- Plano gratuito (Trial Project): 1000 chamadas de API/dia/projeto, válido por 1 ano renovável.
- Devices: ilimitados.
- Após 1 ano: renovar Trial OU migrar pra plano IoT Core ($0.005/device/mês após 100 devices grátis).
- Monitorar via [[cost-audit]] em paralelo com Vercel/Supabase.

## Tabela resumo do que vai onde

| Item | App Smart Life | Cloud Project (admin) | Lumied (gerente) |
|---|:-:|:-:|:-:|
| Conta da escola | ✅ 1x | — | — |
| Pareamento Wi-Fi do hub | ✅ 1x/hub | — | — |
| Cadastro de ACs (catálogo) | ✅ 1x/modelo | — | — |
| Aprender AC fora do catálogo | ✅ se preciso | — | — |
| Link App Account | — | ✅ 1x/escola | — |
| `tuya_space_id` | — | — | ✅ via `ir_setup_tuya_space` |
| Cadastro de hubs no banco | — | — | ✅ `ir_hubs` |
| Vincular `tuya_remote_id` | — | — | ✅ `ir_dispositivo_discover_remotes` |
| Política, agendamento, operação | — | — | ✅ daqui pra frente |

## Sinais de sucesso (declare done quando)

- [ ] Hubs aparecem online em `ir_hubs.online = true` e `ultimo_sync` < 5 min.
- [ ] Pelo menos 1 AC ligou/desligou pelo painel com confirmação física na sala.
- [ ] `ir_agendamentos.tuya_sync_status = 'ok'` em todos os agendamentos criados.
- [ ] Gerente da escola consegue acessar o painel sem ajuda nas próximas 24h (autonomia).

Relacionado: [[project_climatizacao_ir]], [[escola-onboarding]], [[feedback_automerge_no_cascade]], [[cost-audit]]
