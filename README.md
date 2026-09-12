# CI/CD Shared Workflows (`ci-cd`)

A centralized collection of reusable GitHub Actions workflows to standardise build, test, and deployment pipelines across projects.

### Releasing reusable workflows

Clients should pin reusable workflows to `@v1` or `@latest`. Every push to `main`, including a merged pull request, moves both tags to the newest commit:

```bash
git push origin main
```

The `Update reusable workflow tags` workflow automatically maintains `v1` and `latest`. These are intentionally floating tags: clients receive every merged change as soon as the tag update completes. Use a semantic version tag when an immutable release reference is required.

## Generic Docker Infrastructure Deployment

`docker-deploy.yml` is the generic deployment workflow for repositories that own Docker infrastructure. The caller repository owns the desired state (`compose/`, `config/`, scripts, and environment structure); this repository owns the deployment mechanics:

- validates the Compose configuration
- serializes deployments per environment
- syncs the repository to the remote host over SSH
- writes an environment file from a tracked `.env.default` schema and non-secret repository variables
- runs `docker compose pull` and `up -d` for the complete project or selected services
- checks running and unhealthy selected services
- restores the previous files and restarts the previous stack if deployment fails

The workflow is intentionally unaware of individual services, registries, domains, or application secret names.

### Caller workflow

In the infrastructure repository, add `.github/workflows/deploy.yml`:

```yaml
name: Deploy infrastructure

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read

jobs:
  deploy:
    uses: hacksawrazor/ci-cd/.github/workflows/docker-deploy.yml@v1
    with:
      environment: aws-prod
      deploy-path: /opt/infra-docker
      compose-file: compose/production.yml
      compose-project: infra
      environment-file: .env
      environment-vars-json: ${{ toJson(vars) }}
      # Omit to deploy every service. Otherwise use space-separated names.
      services: postgres redis
      pull-images: true
      health-check: true
    secrets:
      SERVER_HOST: ${{ secrets.AWS_SERVER_HOST }}
      SERVER_USER: ${{ secrets.AWS_SERVER_USER }}
      SERVER_SSH_KEY: ${{ secrets.AWS_SERVER_SSH_KEY }}
      SERVER_SSH_PORT: ${{ secrets.AWS_SERVER_SSH_PORT }}
```

    Use the same workflow for OCI by changing the target secrets and `environment` value. The workflow reads `${environment-file}.default`, overlays matching non-secret variables from `environment-vars-json`, and preserves defaults for variables that are unavailable.

The reusable-workflow caller syntax does not reliably pass caller GitHub Environment secrets through `secrets: inherit`. Map repository or organization secrets explicitly as above, or use distinct names such as `AWS_*` and `OCI_*` in the caller repository. Do not put application secrets in this repository.

### Compose validation

For pull requests, use the smaller validation workflow:

```yaml
name: Validate infrastructure

on:
  pull_request:

permissions:
  contents: read

jobs:
  compose:
    uses: hacksawrazor/ci-cd/.github/workflows/docker-validate.yml@v1
    with:
      compose-file: compose/production.yml
      compose-project: infra
```

### Deployment contract

The remote host must provide Docker Engine, the Docker Compose plugin, and an SSH account able to write `deploy-path`. The workflow excludes `.git/`, `.github/`, and its own `.deploy-backups/` directory from synchronization. Backups are kept on the server only for the duration of the deployment and are removed after success.

Use `dry-run: true` to sync and validate the project without pulling images or changing running services. Use `prune: true` only when the remote host is dedicated to this deployment, because it runs `docker image prune -f`.

### Deploy individual services

For independent service projects, point `compose-file` and `environment-file` at the same service directory and omit `services`:

```yaml
with:
  environment: aws-prod
  deploy-path: /opt/infra-docker
  compose-file: services/postgres/compose.yml
  compose-project: postgres
  environment-file: services/postgres/.env
  environment-vars-json: ${{ toJson(vars) }}
```

This produces `/opt/infra-docker/services/postgres/compose.yml` and `/opt/infra-docker/services/postgres/.env`. The workflow creates missing parent directories and passes the environment file to Compose with `--env-file`, so it is used for variable interpolation as well as container environment configuration.

`services` remains available for a shared Compose file. An empty value deploys the complete project; a value such as `postgres redis` pulls, starts, and health-checks only those Compose services. Selected services use `--no-deps`, so dependencies must already be running or be included in the same deployment.

The infrastructure repository controls per-container configuration in Compose. For example:

```yaml
services:
  postgres:
    image: postgres:17
    env_file:
      - config/postgres.env

  oauth:
    image: quay.io/oauth2-proxy/oauth2-proxy:v7
    env_file:
      - config/oauth.env
```

Keep non-secret configuration defaults in tracked `.env.default` files. Pass `${{ toJson(vars) }}` through `environment-vars-json` to overlay matching GitHub repository or environment variables. Only keys present in the default file are emitted. The generated file is transferred to the configured remote `environment-file` before Compose starts.

For separate deployment triggers, create one caller job per service. Jobs using the same `environment` are serialized by the deployment lock, which prevents two rsync operations from modifying the same deployment directory at once:

```yaml
jobs:
  postgres:
    uses: hacksawrazor/ci-cd/.github/workflows/docker-deploy.yml@v1
    with:
      environment: aws-prod
      deploy-path: /opt/infra-docker
      compose-file: compose/production.yml
      compose-project: infra
      environment-file: .env
      environment-vars-json: ${{ toJson(vars) }}
      services: postgres
    secrets:
      SERVER_HOST: ${{ secrets.AWS_SERVER_HOST }}
      SERVER_USER: ${{ secrets.AWS_SERVER_USER }}
      SERVER_SSH_KEY: ${{ secrets.AWS_SERVER_SSH_KEY }}

  oauth:
    needs: postgres
    uses: hacksawrazor/ci-cd/.github/workflows/docker-deploy.yml@v1
    with:
      environment: aws-prod
      deploy-path: /opt/infra-docker
      compose-file: compose/production.yml
      compose-project: infra
      environment-file: .env
      environment-vars-json: ${{ toJson(vars) }}
      services: oauth
    secrets:
      SERVER_HOST: ${{ secrets.AWS_SERVER_HOST }}
      SERVER_USER: ${{ secrets.AWS_SERVER_USER }}
      SERVER_SSH_KEY: ${{ secrets.AWS_SERVER_SSH_KEY }}
```

The deployment workflow is assembled from these reusable composite actions:

| Action | Responsibility |
| :--- | :--- |
| `configure-ssh` | Creates the temporary SSH key and known-hosts files. |
| `sync-docker-project` | Backs up the remote directory, synchronizes repository files, and transfers the generated environment payload. |
| `deploy-docker-project` | Validates Compose, pulls and starts all or selected services, checks health, prunes optionally, and rolls back on failure. |

---

## 🛠 Available Workflows

| Workflow | Description | Path |
| :--- | :--- | :--- |
| **Deploy React App to OCI** | Builds a React app and deploys static files to an Oracle Cloud VM via SSH/Rsync. | `.github/workflows/deploy-react.yml` |
| **Generic Docker Compose Deployment** | Syncs any Docker infrastructure repository to a remote host, deploys it, checks health, and rolls back files on failure. | `.github/workflows/docker-deploy.yml` |
| **Generic Docker Compose Validation** | Validates a caller repository's Compose configuration without deploying it. | `.github/workflows/docker-validate.yml` |
| **Dynamic Docker Deploy to EC2** | Builds image to GHCR and deploys via Docker Compose to AWS EC2 using OIDC & individual secrets. | `.github/workflows/deploy-docker-ghcr.yml` |
| **Dynamic Docker Deploy to OCI** | Builds image to GHCR and deploys via Docker Compose to an Oracle Cloud VM over SSH. | `.github/workflows/deploy-docker-oci.yml` |
| **Existing Docker Image Deploy to OCI** | Pulls an existing GHCR image tag and deploys it via Docker Compose without rebuilding. | `.github/workflows/deploy-docker-oci-existing.yml` |

---

## Shared Composite Actions

The reusable workflows are assembled from step-level composite actions under `.github/actions/`:

| Action | Responsibility |
| :--- | :--- |
| `build-docker-image` | Logs in to a registry, builds an image, and pushes immutable and `latest` tags. |
| `prepare-docker-deployment` | Normalizes or generates Compose configuration and creates `.env` from the schema and secrets JSON. |
| `deploy-docker-compose` | Copies deployment files, prepares bind-mount permissions, and runs Docker Compose on a remote SSH host. |
| `build-webapp` | Sets up Node.js, installs dependencies, and runs the configured webapp build command. |

Reusable workflows reference these actions from `hacksawrazor/ci-cd@main`, so they also work when called by another repository. Provider-specific behavior stays in the workflow: AWS retains OIDC authentication, while OCI uses SSH deployment.

## 🚀 How to Use

### 1. Reusable React Deploy to Oracle Cloud (`deploy-react.yml`)

This workflow handles Node environment setup, dependency installation, building the production assets, and syncing them over SSH to your Oracle Cloud server.

**Note**: Appropriate Read/Write permission required for the user in destination path.

#### Setup in Calling Repository

Create a workflow file in your project (e.g., `.github/workflows/deploy.yml`):

```yaml
name: Deploy Application

on:
  push:
    branches:
      - main

jobs:
  deploy:
    uses: hacksawrazor/ci-cd/.github/workflows/deploy-react.yml@main
    with:
      node-version: '20'
      build-command: 'npm run build'
      build-dir: 'dist'             # Use 'build' for Create React App, 'dist' for Vite
      deploy-path: '/var/www/html/hacksaw/apps/<app-name>' # Destination path on OCI server
    secrets:
      SSH_HOST: ${{ secrets.OCI_SERVER_IP }}
      SSH_USER: ${{ secrets.OCI_SERVER_USER }}
      SSH_KEY: ${{ secrets.OCI_SSH_PRIVATE_KEY }}
      # SSH_PORT: ${{ secrets.OCI_SSH_PORT }} # Optional, defaults to 22
```
### 2. Dynamic Docker Deploy to AWS EC2 via GHCR (deploy-docker-ghcr.yml)
Builds container images using Docker, pushes them to GitHub Container Registry (GHCR), authenticates passwordlessly to AWS using OpenID Connect (OIDC), and deploys using Docker Compose on an AWS EC2 instance.

#### Features
- Dynamic Compose Execution: Uses the project's custom docker-compose.yml if present; otherwise generates a default Compose file dynamically for single-container apps.

- Schema-Based .env Generation: Reads .env.example from the repository, injects individual secrets mapped from GitHub, and uses default fallback values defined in .env.example. Multiline secret values are written with literal `\n` sequences so Docker Compose accepts the generated env file.

- Zero Long-Lived AWS Keys: Authenticates to AWS IAM using GitHub OIDC tokens instead of static access keys.

- Secure Cleanup: Deletes the temporary .env file on the remote server immediately after containers start up.
- Volume Permissions: Resolves Compose bind mounts, creates their host directories, and recursively applies the numeric service `user` ownership plus `u+rwX,go+rX` permissions before containers start. Docker-managed named volumes are created by Docker.
- Manual Compose Runs: Persists `FULL_IMAGE` and `IMAGE_TAG` in the remote `.env`, so `docker compose up -d` can be run manually from the deployment directory.

#### Prerequisites
1. AWS OIDC Provider & Role Setup: Configure an IAM Role in AWS trusting repo:<ORGANIZATION_OR_USERNAME>/* with sts:AssumeRoleWithWebIdentity.

2. Caller Permissions: The calling workflow must explicitly grant id-token: write, contents: read, and packages: write permissions.

#### Setup in Calling Repository
Create a workflow file in your backend project (e.g., `.github/workflows/deploy.yml`):

```yml
name: Deploy Application

on:
  push:
    branches:
      - main

permissions:
  id-token: write
  contents: read
  packages: write

jobs:
  deploy:
    uses: hacksawrazor/ci-cd/.github/workflows/deploy-docker-ghcr.yml@main
    with:
      aws-region: 'ap-south-1'
      aws-role-to-assume: 'arn:aws:iam::<AWS_ACCOUNT_ID>:role/GitHubActionsEC2DeployRole'
      image-name: '${{ github.repository }}'
      deploy-path: '/home/ubuntu/apps/my-backend'
      container-port: '3000:3000' # Optional fallback if no docker-compose.yml exists
      use-sudo: false # Set true for paths such as /opt/apps/my-backend
    secrets:
      SSH_HOST: ${{ secrets.EC2_PUBLIC_IP }}
      SSH_USER: ${{ secrets.EC2_SSH_USER }}
      SSH_KEY: ${{ secrets.EC2_SSH_PRIVATE_KEY }}
      
      # Maps all individual repository secrets dynamically into .env generator
      INDIVIDUAL_SECRETS_JSON: ${{ toJson(secrets) }}
```

#### Multiline Secrets

Docker Compose env files require one physical line per variable. For multiline values such as RSA private keys, store the key as a GitHub secret and reference it through `.env.example`:

```env
PRIVATE_KEY=
```

The deployment action writes newlines as literal `\n` sequences. Your application must restore them before using the key. For example, in Node.js:

```js
const privateKey = process.env.PRIVATE_KEY.replace(/\\n/g, '\n');
```

Do not commit private keys to `.env` or `.env.example`. Rotate a key immediately if it is exposed.

#### Compose Volumes

Before starting the stack, the deployment action resolves the Compose file and prepares each bind-mount source directory on the remote host. If a service declares a numeric `user` such as `1000:1000`, that ownership is recursively applied to the entire bind-mount tree, along with `u+rwX,go+rX` permissions so the container user can write. Named volumes are managed by Docker and are not changed by this step. Permission preparation runs after `docker-compose.yml` and `.env` are copied and fails the deployment if any directory operation fails.

For bind mounts that need a specific application UID/GID, declare it explicitly in the service:

```yaml
services:
  app:
    user: "1000:1000"
    volumes:
      - ./data:/app/data
```

Use the workflow's `use-sudo: true` option when a bind-mount source is outside the deployment user's writable directory.

After deployment, you can manage the stack directly on the server:

```bash
cd /path/to/deployment
docker compose pull
docker compose up -d
docker compose logs -f
```

The remote `.env` contains the image and tag selected by the workflow. To deploy a different image version manually, update `IMAGE_TAG` in `.env` and run `docker compose pull && docker compose up -d`.

### 3. Dynamic Docker Deploy to OCI via GHCR (deploy-docker-oci.yml)
Builds a container image, pushes it to GHCR, and deploys it to an Oracle Cloud VM over SSH using Docker Compose.

#### Setup in Calling Repository
Create a workflow file in your backend project:

```yml
name: Deploy Application

on:
  push:
    branches:
      - main

permissions:
  contents: read
  packages: write

jobs:
  deploy:
    uses: hacksawrazor/ci-cd/.github/workflows/deploy-docker-oci.yml@main
    with:
      image-name: '${{ github.repository }}'
      deploy-path: '/home/ubuntu/apps/my-backend'
      container-port: '3000:3000' # Optional fallback if no docker-compose.yml exists
    secrets:
      SSH_HOST: ${{ secrets.OCI_SERVER_IP }}
      SSH_USER: ${{ secrets.OCI_SERVER_USER }}
      SSH_KEY: ${{ secrets.OCI_SSH_PRIVATE_KEY }}
      SSH_PORT: ${{ secrets.OCI_SSH_PORT }} # Optional, defaults to 22
      INDIVIDUAL_SECRETS_JSON: ${{ toJson(secrets) }}
```

### 4. Deploy an Existing Docker Image to OCI (`deploy-docker-oci-existing.yml`)
Pulls the selected image tag from GHCR and deploys it without building or pushing an image. The workflow still prepares Compose bind-mount permissions and runs `docker compose pull` followed by `docker compose up -d --remove-orphans`.

```yml
name: Deploy Existing Image

on:
  workflow_dispatch:
    inputs:
      image-tag:
        description: Image tag to deploy
        required: true
        default: latest

permissions:
  contents: read
  packages: read

jobs:
  deploy:
    uses: hacksawrazor/ci-cd/.github/workflows/deploy-docker-oci-existing.yml@main
    with:
      image-name: '${{ github.repository }}'
      image-tag: ${{ inputs.image-tag }}
      deploy-path: '/home/ubuntu/apps/my-backend'
      use-sudo: true
    secrets:
      SSH_HOST: ${{ secrets.OCI_SERVER_IP }}
      SSH_USER: ${{ secrets.OCI_SERVER_USER }}
      SSH_KEY: ${{ secrets.OCI_SSH_PRIVATE_KEY }}
      SSH_PORT: ${{ secrets.OCI_SSH_PORT }}
      INDIVIDUAL_SECRETS_JSON: ${{ toJson(secrets) }}
```
