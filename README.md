# CI Workflows

Repositório centralizado de workflows reutilizáveis do GitHub Actions para projetos da Bridgge.

## 📋 Workflows Disponíveis

### Build and Deploy to Coolify

Workflow reutilizável que automatiza o processo de build de imagens Docker, push para o GitHub Container Registry (GHCR) e disparo de deploy no Coolify.

#### Funcionalidades

- ✅ Build de imagens Docker usando Docker Buildx
- ✅ Push automático para GitHub Container Registry (ghcr.io)
- ✅ Cache de build para otimização
- ✅ Tags automáticas (latest e branch-sha)
- ✅ Disparo de deploy no Coolify via webhook
- ✅ Tratamento de erros e retry automático

#### Como Usar

Para usar este workflow em outro repositório, adicione-o ao seu arquivo `.github/workflows/`:

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]

jobs:
  build-and-deploy:
    uses: Bridgge/ci-workflows/.github/workflows/build-deploy-coolify.yml@main
    with:
      image_name: meu-projeto  # Opcional: nome da imagem (padrão: nome do repositório)
      registry: ghcr.io        # Opcional: registry Docker (padrão: ghcr.io)
    secrets:
      COOLIFY_WEBHOOK_URL: ${{ secrets.COOLIFY_WEBHOOK_URL }}
      COOLIFY_TOKEN: ${{ secrets.COOLIFY_TOKEN }}
```

#### Parâmetros de Entrada (Inputs)

| Parâmetro | Tipo | Obrigatório | Padrão | Descrição |
|-----------|------|-------------|--------|-----------|
| `image_name` | string | Não | Nome do repositório | Nome da imagem Docker |
| `registry` | string | Não | `ghcr.io` | Registry Docker para push |

#### Secrets Necessários

| Secret | Obrigatório | Descrição |
|--------|-------------|-----------|
| `COOLIFY_WEBHOOK_URL` | Sim | URL do webhook do Coolify para disparar deploy |
| `COOLIFY_TOKEN` | Sim | Token de autenticação do Coolify |

#### Tags Geradas

O workflow gera automaticamente as seguintes tags para a imagem:

- `latest` - Sempre aponta para a última build
- `{branch}-{sha}` - Tag específica da branch e commit (ex: `main-abc1234`)

#### Requisitos

- Dockerfile no diretório raiz do projeto
- Permissões de escrita no GitHub Container Registry
- Webhook configurado no Coolify

## 🔧 Configuração

### 1. Configurar Secrets no Repositório

No repositório que utilizará o workflow, configure os seguintes secrets:

1. Vá em **Settings** → **Secrets and variables** → **Actions**
2. Adicione:
   - `COOLIFY_WEBHOOK_URL`: URL do webhook do seu projeto no Coolify
   - `COOLIFY_TOKEN`: Token de autenticação do Coolify

### 2. Configurar Permissões

O workflow requer as seguintes permissões:

- `contents: read` - Para fazer checkout do código
- `packages: write` - Para fazer push das imagens Docker

## 📝 Exemplo Completo

```yaml
name: Deploy to Production

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  deploy:
    uses: Bridgge/ci-workflows/.github/workflows/build-deploy-coolify.yml@main
    with:
      image_name: minha-api
      registry: ghcr.io
    secrets:
      COOLIFY_WEBHOOK_URL: ${{ secrets.COOLIFY_WEBHOOK_URL }}
      COOLIFY_TOKEN: ${{ secrets.COOLIFY_TOKEN }}
```

## 🚀 Fluxo de Execução

1. **Checkout** - Faz checkout do código do repositório
2. **Setup Docker Buildx** - Configura o builder Docker
3. **Login GHCR** - Autentica no GitHub Container Registry
4. **Build e Push** - Constrói a imagem Docker e faz push para o registry
5. **Trigger Coolify** - Dispara o webhook do Coolify para iniciar o deploy

## 📚 Recursos

- [GitHub Actions - Reusable Workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows)
- [Docker Buildx](https://docs.docker.com/buildx/working-with-buildx/)
- [GitHub Container Registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)
- [Coolify Documentation](https://coolify.io/docs)

## 📄 Licença

Este repositório é parte do projeto Bridgge.