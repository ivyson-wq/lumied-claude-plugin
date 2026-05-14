---
name: chrome-extension-pack
description: Empacota e publica a [[project_lumied_chrome_extension]] no Web Store — usa Compress-Archive no Windows (pack.sh com zip não roda no Git Bash Win), valida manifest, gera .zip pronto, abre o devconsole. Não há deploy automatizado; é manual via Web Store. Use quando o usuário disser "publica extensão", "subir nova versão", ou ao bater versão em manifest.json.
---

# Chrome Extension pack & release — Lumied

Contexto: [[project_lumied_chrome_extension]] é MV3 em `maple-bear-rs/chrome-extension/`. O `pack.sh` original usa `zip` (Unix), que **não roda no Git Bash do Windows** por padrão. Usar PowerShell `Compress-Archive` em vez disso. Release é manual: faz upload do .zip no devconsole da Web Store, sem deploy automático.

## Quando rodar

- Subiu versão em `manifest.json`.
- Mudou código da extensão e quer testar localmente / publicar.
- Cliente reportou bug específico da extensão e precisa de patch rápido.

NÃO usar pra:
- Mudanças em outros repos que não a extensão.
- Hotfix urgente que precisa chegar a milhares de instalações em <1h — Web Store tem review (horas a dias).

## Pré-pack

### 1. Validar manifest

```bash
cat maple-bear-rs/chrome-extension/manifest.json
```

Conferir:
- `"manifest_version": 3`
- `"version"`: bump correto (semver: bug=patch, feature=minor, breaking=major).
- `"name"` e `"description"` corretos.
- `"permissions"`: somente o necessário (cada permissão extra acende flag no review).
- `"host_permissions"`: domínios mínimos.
- `"action"` / `"background"` / `"content_scripts"` paths apontam pra arquivos que existem.

### 2. Smoke test local

Em Chrome:
- `chrome://extensions` → modo desenvolvedor.
- "Carregar sem compactação" → seleciona a pasta.
- Testa fluxo principal sem erros no console.

### 3. Rodar lint / typecheck se aplicável

```bash
cd maple-bear-rs/chrome-extension
# Se tem package.json com script:
npm run lint --silent 2>&1 || true
npm run typecheck --silent 2>&1 || true
```

## Pack (Windows / PowerShell)

```powershell
# Estando em maple-bear-rs/chrome-extension/
$version = (Get-Content manifest.json | ConvertFrom-Json).version
$zipName = "lumied-extension-v$version.zip"

# Limpar zips antigos
Remove-Item -ErrorAction SilentlyContinue *.zip

# Compactar somente o que precisa ir pro pacote
# (Excluir node_modules, .git, *.md, screenshots de devel, etc.)
Compress-Archive `
  -Path manifest.json, icons/, popup/, background/, content/, _locales/ `
  -DestinationPath $zipName `
  -CompressionLevel Optimal

Write-Output "Gerado: $zipName"
Get-Item $zipName | Select-Object Name, Length
```

Adapta a lista de `-Path` pro que realmente existe no diretório da extensão.

### Conferir o zip

- Tamanho razoável (< 5MB normalmente; Web Store aceita até 10MB).
- Abrir o zip: confere se `manifest.json` está na raiz (não dentro de uma pasta).
- Se está dentro de uma pasta, refaz com `-Path *` ou ajusta.

## Release no Web Store

Manual (não automatizado):

1. Acessar `https://chrome.google.com/webstore/devconsole`.
2. Selecionar a extensão Lumied.
3. "Pacote" → "Carregar novo pacote" → escolhe o `.zip`.
4. Atualizar descrição / screenshots se houve mudança visual.
5. Em "Sigilo" / "Privacidade", confirmar declarações (token, storage, etc.).
6. "Enviar para revisão".
7. Aguardar review (1-24h normalmente, pode ser mais).

## Pós-release

- Anotar a versão publicada no [[project_lumied_chrome_extension]].
- Se houve mudança de permissão, usuários instalados vão precisar re-aceitar — comunicar.
- Se quebrou: rollback é **publicar a versão anterior** (zip do .git history). Web Store não tem botão "voltar".

## Anti-padrões

- Rodar `bash pack.sh` no Git Bash do Windows ([[project_lumied_chrome_extension]] documenta — `zip` não está disponível).
- Subir sem bumpar versão (Web Store rejeita: "version não pode ser igual à publicada").
- Incluir `node_modules` ou `.git` no zip (zip enorme + review reprova).
- Esquecer de minificar / remover console.log se for sensível.
- Subir com `host_permissions: ["<all_urls>"]` se a extensão só precisa de 1 domínio — review fica mais lento ou rejeita.

## Se a extensão for tornada open-source

- Validar [[secrets-scan]] no diretório antes — não pode ter token Lumied / API key no código bundle.
- Documentar como dev local pode rodar contra um Supabase próprio.

## Linha alternativa: zip via PowerShell one-liner

Se já está claro o que vai entrar:

```powershell
Compress-Archive -Path * -DestinationPath "lumied-extension-v$((Get-Content manifest.json | ConvertFrom-Json).version).zip" -Force
```

(Roda da raiz do diretório da extensão; cuidado pra estar **dentro** dela.)
