# CI/CD Shared Workflows (`ci-cd`)

A centralized collection of reusable GitHub Actions workflows to standardise build, test, and deployment pipelines across projects.

---

## 🛠 Available Workflows

| Workflow | Description | Path |
| :--- | :--- | :--- |
| **Deploy React App to OCI** | Builds a React app and deploys static files to an Oracle Cloud VM via SSH/Rsync. | `.github/workflows/deploy-react.yml` |
| *More coming soon...* | *Frameworks like Node.js APIs, Docker, Next.js, etc.* | `.github/workflows/` |

---

## 🚀 How to Use

### 1. Reusable React Deploy to Oracle Cloud (`deploy-react.yml`)

This workflow handles Node environment setup, dependency installation, building the production assets, and syncing them over SSH to your Oracle Cloud server.

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
      deploy-path: '/var/www/my-app' # Destination path on OCI server
    secrets:
      SSH_HOST: ${{ secrets.OCI_SERVER_IP }}
      SSH_USER: ${{ secrets.OCI_SERVER_USER }}
      SSH_KEY: ${{ secrets.OCI_SSH_PRIVATE_KEY }}
      # SSH_PORT: ${{ secrets.OCI_SSH_PORT }} # Optional, defaults to 22