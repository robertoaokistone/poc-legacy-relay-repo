# Runbook — legacy-relay

Guia operacional para configurar, validar e operar o sync entre um repo legado e um repo novo usando a action `legacy-relay`.

---

## Pré-requisitos

| Requisito | Detalhe |
|---|---|
| Repos no GitHub | Legado e novo podem estar em orgs diferentes |
| Token de acesso | PAT ou GitHub App com `contents: write` + `pull-requests: write` no repo novo |
| Runner | `ubuntu-latest` (rsync, gh CLI e jq já instalados) |
| Relay repo | `robertoaokistone/poc-legacy-relay-repo` público (ou privado com acesso cross-org habilitado) |

> **Cross-org**: Para repos em organizações diferentes, prefira um **GitHub App** instalado nas duas orgs. Um PAT pessoal funciona mas cria dependência de conta individual.

---

## 1. Configurar o secret no repo legado

```bash
# Usando o token atual (não expõe o valor)
gh auth token | gh secret set NEW_REPO_TOKEN --repo org/repo-legado

# Ou via UI: Settings → Secrets → Actions → New repository secret
# Nome: NEW_REPO_TOKEN
# Permissões no repo novo: contents: write, pull-requests: write
```

---

## 2. Criar o workflow no repo legado

Copie o arquivo abaixo para `.github/workflows/sync-to-new.yml` no repo legado:

```yaml
name: Sync para Repo Novo

on:
  push:
    branches: [main]  # ajuste para sua branch padrão

jobs:
  sync:
    uses: robertoaokistone/poc-legacy-relay-repo/.github/workflows/sync-legacy-to-new.yml@main
    with:
      new_repo: 'sua-org/repo-novo'

      # Arquivos/dirs do legado que NÃO devem ir pro novo.
      # .github/ NÃO é excluído por padrão — workflows podem ser sincronizados.
      # Liste aqui apenas o que é exclusivo do legado.
      exclude_paths: |
        .k8s/
        .github/workflows/deploy-legado.yml
        .github/workflows/sync-to-new.yml

    secrets:
      new_repo_token: ${{ secrets.NEW_REPO_TOKEN }}
```

---

## 3. Estrutura esperada dos repos

```
repo-legado                          repo-novo
───────────────────────────────      ──────────────────────────────────
src/                   ──sync──▶     src/
.github/CODEOWNERS     ──sync──▶     .github/CODEOWNERS
.github/workflows/
  validate.yml         ──sync──▶     .github/workflows/validate.yml
  deploy-legado.yml    ✗ excluído
  sync-to-new.yml      ✗ excluído    .github/workflows/deploy-novo.yml  ← exclusivo
.k8s/                  ✗ excluído    charts/                            ← exclusivo
                                     new-only-config.yaml               ← exclusivo
```

---

## 4. Como funciona

1. Push na branch padrão do legado dispara o workflow `sync-to-new.yml`
2. O reusable workflow faz checkout do relay repo no mesmo ref (`github.workflow_ref`)
3. A composite action:
   - Faz checkout do legado (no commit do push) e do novo repo
   - Reseta a branch `sync/from-legacy` para `origin/main` do novo (estado atual, não delta)
   - Executa `rsync -a --checksum` sem `--delete` (preserva arquivos exclusivos do novo)
   - Aplica as exclusões de `exclude_paths`
   - Commita e faz `force-with-lease` na `sync/from-legacy`
   - Cria ou atualiza um único PR aberto no novo repo

```
push → legado
          │
          ▼
    [sync workflow]
          │
          ├── resolve ref do relay (github.workflow_ref)
          ├── checkout relay no mesmo ref
          └── composite action
                  │
                  ├── checkout legado + novo
                  ├── reset sync/from-legacy → origin/main
                  ├── rsync (sem --delete, com exclusões)
                  ├── commit + force-with-lease push
                  └── gh pr create (ou atualiza PR existente)
```

---

## 5. Conflitos e hotfixes

Se um arquivo foi corrigido diretamente no novo repo (hotfix) e depois modificado no legado, o PR de sync vai conter a versão do legado. O revisor deve inspecionar e incorporar o hotfix manualmente no PR.

**Diretriz**: não corrigir no legado só para alimentar o sync. O legado está em descontinuação.

---

## 6. Problemas conhecidos

### Workflow file issue (0s de execução)

**Causa**: O repo do relay é privado e o runner do legado não tem acesso.

**Solução A** (POC/rápida): Tornar o relay repo público.
```bash
gh repo edit org/poc-legacy-relay-repo --visibility public --accept-visibility-change-consequences
```

**Solução B** (produção): Habilitar acesso cross-repo em *Settings → Actions → General → Access* do relay repo.

---

### `unknown flag: --json` no `gh pr create`

**Causa**: Versões antigas do gh CLI no runner não suportam `--json` em `gh pr create`.

**Status**: Corrigido na versão atual — `gh pr create` retorna a URL diretamente no stdout.

---

### `Can't find action.yml` para composite action `@main`

**Causa**: O reusable workflow referenciava a composite action com `@main`, mas `main` não tinha os arquivos ainda (somente na branch do PR).

**Solução**: O workflow agora resolve `github.workflow_ref` e faz checkout do relay no mesmo ref antes de executar a action por path local (`./relay-repo/.github/actions/sync-legacy-to-new`).

---

### Repo privado ao chamar reusable workflow de outro repo

**Causa**: O `GITHUB_TOKEN` do repo chamador não tem acesso a repos privados de terceiros.

**Solução**: Deixar o relay repo público, ou usar um GitHub App com acesso nos dois repos.

---

## 7. Validação da POC

```bash
# 1. Verificar se o workflow rodou com sucesso no legado
gh run list --repo org/repo-legado --workflow sync-to-new.yml --limit 5

# 2. Confirmar PR aberto no novo repo
gh pr list --repo org/repo-novo

# 3. Inspecionar o diff do PR (verificar exclusões e preservação de arquivos)
gh pr diff 1 --repo org/repo-novo

# 4. Re-disparar manualmente (commit vazio)
cd repo-legado && git commit --allow-empty -m "chore: aciona sync manual" && git push
```

---

## 8. Arquivamento do legado

Quando o legado estiver pronto para ser arquivado:

1. Mergear ou fechar todos os PRs de sync abertos no novo repo
2. Remover o workflow `sync-to-new.yml` do legado (ou arquivar o repo diretamente)
3. O novo repo segue de forma independente

```bash
gh repo archive org/repo-legado
```

---

## Repos de exemplo (POC)

| Repo | Papel |
|---|---|
| `robertoaokistone/poc-legacy-relay-repo` | Action relay (este repo) |
| `robertoaokistone/poc-legacy-repo` | Repo legado de exemplo |
| `robertoaokistone/poc-new-repo` | Repo novo de exemplo |
