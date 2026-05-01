# Beetax-Technologies/.github

Defaults org-wide pra repos da Beetax Technologies: **reusable workflows** + community health files.

Apache 2.0. Pode usar fora da org tb (mas o runner self-hosted é nosso).

---

## Reusable workflows disponíveis

### `publish-docker.yml`

Build + push de imagem Docker pra GHCR usando o runner self-hosted BeeHive (CX23 amd64) com QEMU pra cross-build arm64. Multi-arch por default (`linux/amd64,linux/arm64`).

**Por que existe:** padroniza o pipeline `tag → CI → GHCR → Coolify` que a gente quer pra toda app interna. Sem isso, cada repo novo recopia 80 linhas de YAML hardcoded.

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

#### Versionamento do template

- `@v1` -> última v1.x.x compatível (recomendado)
- `@main` -> bleeding edge (pode quebrar)
- `@v1.2.3` -> versão exata (reprodutível, mas precisa atualizar manual)

Breaking changes só em major bump (v2). Qualquer mudança v1 -> v2 vai documentada no CHANGELOG.

---

## Pré-requisitos

1. **Runner self-hosted BeeHive online** (`[self-hosted, beehive]`). Hoje só temos o CX23 amd64. Se cair, workflows ficam em queue.
2. **Acesso ao GHCR** da org `beetax-technologies`. O `secrets.GITHUB_TOKEN` automático cobre repos da org.
3. **Permissions no caller** explícitas:
   ```yaml
   permissions:
     contents: read
     packages: write
   ```
   (reusable workflow não herda automaticamente.)

## Trade-offs conhecidos

- **arm64 via QEMU é lento** (~3-5x build nativo). Se a frequência de tag for alta, considera split: 1 job amd64 nativo + 1 job ARM nativo + manifest. Por ora aceitável.
- **Single runner = SPOF.** Se o CX23 cair, todos os builds param. Backlog item B-020 (revivar runner ARM nativo) defere isso.
- **Sem cleanup pre-checkout.** Se um build deixar lixo no workspace, próximo run pode falhar. Composite action [`Beetax-Technologies/dind-runner-cleanup@v1`](https://github.com/Beetax-Technologies/dind-runner-cleanup) cobre isso quando o caller quer adicionar.

## Roadmap

- [ ] `publish-npm.yml` reusable (Verdaccio + npm.beetax.tech)
- [ ] `deploy-coolify.yml` reusable (chama BeeHive MCP pra deploy automático após publish)
- [ ] `test-node.yml` reusable (Node.js + cache + lint + test pattern)
- [ ] Auto-update via Renovate bot pra @vX -> @vX.Y.Z

## Licença

Apache 2.0. Ver [LICENSE](LICENSE).
