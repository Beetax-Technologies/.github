# .github

Org-wide GitHub defaults: **reusable workflows** + community health files. Apache 2.0.

External orgs can use these workflows too (the runner labels are org-specific — adjust to your own infrastructure).

---

## Available reusable workflows

### `publish-docker.yml`

Build and push a Docker image to GHCR using a self-hosted runner with QEMU for cross-build arm64. Multi-arch by default (`linux/amd64,linux/arm64`).

**Why:** standardizes the `tag → CI → GHCR → container orchestrator` pipeline for internal apps. Without this each new repo would reimplement ~80 lines of hardcoded YAML.

#### Uso mínimo (1 serviço, Dockerfile na raiz)

```yaml
# .github/workflows/publish.yml no SEU repo
name: Publish

on:
  push:
    tags: ['v*.*.*']
  workflow_dispatch:

jobs:
  publish:
    uses: Beetax-Technologies/.github/.github/workflows/publish-docker.yml@v1
    with:
      service-name: minha-app-nova
    permissions:
      contents: read
      packages: write
```

Tag `v1.0.0` → workflow publica:
- `ghcr.io/beetax-technologies/minha-app-nova:1.0.0`
- `ghcr.io/beetax-technologies/minha-app-nova:latest`

Pra ambos `linux/amd64` e `linux/arm64`.

#### Monorepo: 2+ serviços, tags per-service

```yaml
name: Publish

on:
  push:
    tags:
      - 'v*.*.*'                       # publica TUDO
      - 'api-v*.*.*'                   # publica só api
      - 'worker-v*.*.*'                # publica só worker
  workflow_dispatch:

jobs:
  publish-api:
    if: |
      startsWith(github.ref, 'refs/tags/v') ||
      startsWith(github.ref, 'refs/tags/api-v')
    uses: Beetax-Technologies/.github/.github/workflows/publish-docker.yml@v1
    with:
      service-name: api
      context: services/api
      dockerfile: services/api/Dockerfile
    permissions:
      contents: read
      packages: write

  publish-worker:
    if: |
      startsWith(github.ref, 'refs/tags/v') ||
      startsWith(github.ref, 'refs/tags/worker-v')
    uses: Beetax-Technologies/.github/.github/workflows/publish-docker.yml@v1
    with:
      service-name: worker
      context: services/worker
      dockerfile: services/worker/Dockerfile
    permissions:
      contents: read
      packages: write
```

Tag `api-v1.2.0` -> publica só `ghcr.io/beetax-technologies/api:1.2.0`.
Tag `v3.0.0` -> publica os dois.

#### Inputs

| Input | Tipo | Default | Descrição |
|---|---|---|---|
| `service-name` | string | (obrigatório) | Nome da imagem em GHCR. Vira `ghcr.io/<owner>/<service-name>` |
| `context` | string | `.` | Diretório de build Docker (relativo à raiz do repo) |
| `dockerfile` | string | `Dockerfile` | Path do Dockerfile (relativo à raiz do repo, NÃO ao context) |
| `platforms` | string | `linux/amd64,linux/arm64` | Plataformas comma-separated. Use `linux/amd64` se app não roda em ARM. |
| `registry-owner` | string | `beetax-technologies` | Owner GHCR. Trocar só se forkar o template. |
| `version` | string | (auto) | Override de versão. Default: extrai do git tag (`v1.2.3` ou `<service>-v1.2.3` -> `1.2.3`) |
| `push-latest` | boolean | `true` | Se publica também a tag `:latest` |
| `build-args` | string | `''` | Build args Docker (multilinha `KEY=VALUE` por linha) |

#### Outputs

| Output | Descrição |
|---|---|
| `image` | Imagem completa publicada (`ghcr.io/<owner>/<service-name>:<version>`) |
| `version` | Versão calculada/usada |

#### ⚠️ GOTCHA crítico: prefix `v` é stripado da tag GHCR

A computação de versão **remove o prefix `v`** automaticamente. Tags GHCR ficam **sem `v`**:

| Git tag | GHCR image tag publicada |
|---|---|
| `v1.2.3` | `1.2.3` (sem `v`) |
| `<service>-v1.2.3` | `1.2.3` (sem prefix nem `v`) |
| `1.2.3` (sem `v` no git já) | `1.2.3` |

**When you pin tags in any orchestrator or run a manual pull, use WITHOUT `v`:**

```bash
# ❌ WRONG — doesn't exist on GHCR
docker pull ghcr.io/org/service:v1.2.3

# ✅ RIGHT
docker pull ghcr.io/org/service:1.2.3
```

```json
// Orchestrator config — image tag field
// ❌ WRONG
{"image_tag": "v1.2.3"}

// ✅ RIGHT
{"image_tag": "1.2.3"}
```

> This behaviour was confirmed in production. Apps pinned with the `v` prefix end up in "image not found" loops because the published tag has no `v`. Documented here to prevent regression.

#### Versionamento do template

- `@v1` -> última v1.x.x compatível (recomendado)
- `@main` -> bleeding edge (pode quebrar)
- `@v1.2.3` -> versão exata (reprodutível, mas precisa atualizar manual)

Breaking changes só em major bump (v2). Qualquer mudança v1 -> v2 vai documentada no CHANGELOG.

---

## Prerequisites

1. **Self-hosted runner online** with matching labels. If unavailable, jobs stay queued.
2. **GHCR access**. The automatic `secrets.GITHUB_TOKEN` covers same-org repos.
3. **Explicit permissions in the caller workflow:**
   ```yaml
   permissions:
     contents: read
     packages: write
   ```
   (reusable workflows don't inherit by default.)

## Known trade-offs

- **arm64 via QEMU is slow** (~3-5x native build). For high-frequency tags, consider splitting jobs: one amd64 native + one ARM native + manifest. For low frequency it's acceptable.
- **Sem cleanup pre-checkout.** If a build leaves leftover files in the workspace, the next run can fail. Composite action [`Beetax-Technologies/dind-runner-cleanup@v1`](https://github.com/Beetax-Technologies/dind-runner-cleanup) covers this when the caller opts in.

## Roadmap

- [ ] `publish-npm.yml` reusable (private registry support)
- [ ] `deploy.yml` reusable (post-publish hook for orchestrator)
- [ ] `test-node.yml` reusable (Node.js + cache + lint + test pattern)
- [ ] Auto-update via Renovate bot for `@vX -> @vX.Y.Z`

## Licença

Apache 2.0. Ver [LICENSE](LICENSE).
