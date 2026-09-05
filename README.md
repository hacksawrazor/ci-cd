# CI/CD Shared Workflows (`ci-cd`)

A centralized collection of reusable GitHub Actions workflows to standardise build, test, and deployment pipelines across projects.

---

## 🛠 Available Workflows

| Workflow | Description | Path |
| :--- | :--- | :--- |
| **Deploy React App to OCI** | Builds a React app and deploys static files to an Oracle Cloud VM via SSH/Rsync. | `.github/workflows/deploy-react.yml` |
| **Dynamic Docker Deploy to EC2** | Builds image to GHCR and deploys via Docker Compose to AWS EC2 using OIDC & individual secrets. | `.github/workflows/deploy-docker-ghcr.yml` |
| **Dynamic Docker Deploy to OCI** | Builds image to GHCR and deploys via Docker Compose to an Oracle Cloud VM over SSH. | `.github/workflows/deploy-docker-oci.yml` |

---

## Shared Composite Actions

The reusable workflows are assembled from step-level composite actions under `.github/actions/`:

| Action | Responsibility |
| :--- | :--- |
| `build-docker-image` | Logs in to a registry, builds an image, and pushes immutable and `latest` tags. |
| `prepare-docker-deployment` | Normalizes or generates Compose configuration and creates `.env` from the schema and secrets JSON. |
| `deploy-docker-compose` | Copies deployment files and runs Docker Compose on a remote SSH host. |
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
