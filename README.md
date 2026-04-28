# poc-legacy-relay-repo

GitHub Actions reutilizáveis para sincronizar um **repositório legado** para um **repositório novo**.
Todo merge na branch padrão do repo legado dispara um Pull Request no repo novo
com as mudanças sincronizadas — prontas para revisão humana antes do merge.

---

## Funcionalidades

- 🔄 **Sync automático** — todo merge no legado cria ou atualiza um PR no repo novo
- 🚫 **Exclusão de paths** — configure arquivos/diretórios do legado que **não** devem ir para o novo
- 🛡️ **Preserva arquivos exclusivos do novo** — arquivos que só existem no novo nunca são deletados
- 🔀 **Um PR aberto por vez** — atualiza a branch de sync existente em vez de criar duplicatas
- 🏷️ **Configurável** — personalize nome da branch, título do PR, labels, mensagem de commit e mais

---

## Início rápido — repo legado

Copie [`examples/legacy-repo-sync.yml`](examples/legacy-repo-sync.yml) para
`.github/workflows/sync-to-new.yml` no seu **repo legado** e ajuste os valores:

```yaml
name: Sync para Repo Novo

on:
  push:
    branches:
      - main

jobs:
  sync:
    uses: robertoaokistone/poc-legacy-relay-repo/.github/workflows/sync-legacy-to-new.yml@main
    with:
      new_repo: 'sua-org/repo-novo'
      exclude_paths: |
        .legacy-only/
        legacy-config.txt
        scripts/deploy-legacy.sh
    secrets:
      new_repo_token: ${{ secrets.NEW_REPO_TOKEN }}
```

### Secret obrigatório

Crie um secret chamado **`NEW_REPO_TOKEN`** no repo legado. Duas opções:

**GitHub App (recomendado para cross-org)**
Crie um GitHub App instalado nas duas organizações com as permissões abaixo,
gere um token de instalação e armazene como `NEW_REPO_TOKEN`.
Evita dependência de conta pessoal e funciona entre orgs diferentes.

**Personal Access Token (mais simples para mesma org)**
Use um fine-grained PAT com as permissões abaixo no **repo novo**.

| Permissão | Escopo |
|---|---|
| `contents` | **Write** |
| `pull-requests` | **Write** |

---

## Como funciona

```
Repo legado (merge na main)
        │
        ▼
  [sync workflow]
        │
        ├─ checkout do legado no commit do merge
        ├─ checkout do repo novo
        ├─ rsync legado → novo
        │     ├─ ignora arquivos em exclude_paths
        │     └─ preserva arquivos exclusivos do novo (sem --delete)
        ├─ commit na branch sync/from-legacy
        └─ abre (ou atualiza) um PR no repo novo
                │
                ▼
          Revisão humana → merge
```

1. A cada `push` na branch configurada, o reusable workflow roda no repo legado.
2. O workflow faz checkout do legado no commit do push e do repo novo.
3. A branch `sync/from-legacy` é criada ou resetada para a branch base do novo (estado atual, não delta).
4. `rsync` copia arquivos do legado → novo com estas regras:
   - Arquivos listados em `exclude_paths` **não** são copiados.
   - `.git/` é sempre excluído. **`.github/` NÃO é excluído por padrão** — workflows podem ser sincronizados intencionalmente. Adicione arquivos específicos ao `exclude_paths` (ex: `sync-to-new.yml`, workflows de deploy exclusivos do legado).
   - Arquivos que existem **apenas no novo repo** são preservados (rsync sem `--delete`).
   - Arquivos modificados nos **dois repos** aparecerão com a versão do legado no diff do PR — o revisor decide.
5. Se houver mudanças, um commit é feito e um PR é criado (ou o existente é atualizado).
6. Um humano revisa o PR e faz o merge quando estiver pronto.

> **Política de conflitos e hotfixes**
> Se um arquivo foi corrigido diretamente no novo repo (hotfix) e depois modificado no legado,
> o PR de sync vai conter a versão do legado. O revisor deve inspecionar o diff e
> incorporar o hotfix no branch do PR manualmente.
> A diretriz é **não fazer merge automático** dos PRs de sync — sempre revisar.

---

## Inputs do reusable workflow

Usados ao chamar `.github/workflows/sync-legacy-to-new.yml` via `workflow_call`:

| Input | Obrigatório | Padrão | Descrição |
|---|---|---|---|
| `new_repo` | ✅ | — | Repo novo destino (`owner/repo`) |
| `base_branch` | ❌ | `main` | Branch base no repo novo |
| `sync_branch` | ❌ | `sync/from-legacy` | Branch para o PR de sync |
| `exclude_paths` | ❌ | `''` | Paths a excluir do sync (um por linha) |
| `commit_message` | ❌ | `chore: sync changes from legacy repo` | Mensagem de commit |
| `pr_title` | ❌ | `chore: sync changes from legacy repo` | Título do PR |
| `pr_body` | ❌ | *(mensagem padrão de aviso)* | Corpo do PR |
| `pr_labels` | ❌ | `''` | Labels a adicionar ao PR (separadas por vírgula) |

**Secret:** `new_repo_token` *(obrigatório)* — token com `contents:write` + `pull-requests:write` no repo novo.

---

## Inputs da composite action

Para cenários avançados, você pode chamar a composite action diretamente:

```yaml
steps:
  - uses: robertoaokistone/poc-legacy-relay-repo/.github/actions/sync-legacy-to-new@main
    with:
      new_repo: 'sua-org/repo-novo'
      new_repo_token: ${{ secrets.NEW_REPO_TOKEN }}
      exclude_paths: |
        .legacy-only/
        docs/legacy-guide.md
```

Todos os inputs do reusable workflow estão disponíveis, mais:

| Input | Obrigatório | Padrão | Descrição |
|---|---|---|---|
| `legacy_repo` | ❌ | `${{ github.repository }}` | Repo legado de origem |
| `legacy_ref` | ❌ | `${{ github.sha }}` | Git ref para sincronizar |
| `legacy_token` | ❌ | `${{ github.token }}` | Token para ler o repo legado |

**Outputs:** `pr_url` (URL do PR ou vazio), `has_changes` (`true`/`false`).

---

## Requisitos

A composite action requer as seguintes ferramentas no runner:

| Ferramenta | Finalidade |
|---|---|
| `rsync` | Sync de arquivos do legado para o novo |
| `gh` | GitHub CLI — criar/atualizar PR, gerenciar labels |
| `jq` | Parsear respostas JSON do `gh` |

As três estão pré-instaladas nos runners `ubuntu-latest` hospedados pelo GitHub.
Para runners self-hosted, certifique-se de que essas ferramentas estejam disponíveis.

---

## Pinagem de versão

Os exemplos neste documento referenciam `@main`, que sempre usa a versão mais recente.
Para uso em **produção**, pine a uma tag ou SHA específico para evitar mudanças inesperadas:

```yaml
# Reusable workflow — pine a uma tag de release
uses: robertoaokistone/poc-legacy-relay-repo/.github/workflows/sync-legacy-to-new.yml@v1.0.0

# Composite action — pine a uma tag de release
uses: robertoaokistone/poc-legacy-relay-repo/.github/actions/sync-legacy-to-new@v1.0.0
```

---

## Arquivamento do legado

Quando o repo legado estiver pronto para ser arquivado:

1. Mergear ou fechar todos os PRs de sync abertos no repo novo.
2. Remover o workflow de sync do legado (ou simplesmente arquivar o repo no GitHub).
3. O repo novo segue de forma independente.

---

## Exemplos

Veja o diretório [`examples/`](examples/) para workflows prontos para uso.

---

## Runbook operacional

Para passos de configuração, problemas conhecidos e troubleshooting encontrados durante a POC, veja [RUNBOOK.md](RUNBOOK.md).
