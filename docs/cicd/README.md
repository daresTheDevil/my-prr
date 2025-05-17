# PRRIP CI/CD Strategy

This document outlines the Continuous Integration and Continuous Deployment (CI/CD) strategy for the Pearl River Resort Integration Platform (PRRIP).

## Goals

*   Automate the build, test, and deployment process.
*   Ensure consistent and reliable releases.
*   Enable rapid iteration and feedback.
*   Maintain a clear audit trail of changes and deployments.
*   Facilitate a smooth transition from initial GitLab CI setup to future GitHub Actions / Azure DevOps.

## Phased Approach

### Phase 1: GitLab CI (Initial Implementation)

GitLab CI will be used for the initial CI/CD pipeline due to its robust features and potential existing familiarity or infrastructure.

**Key Stages & Jobs:**

1.  **Lint & Static Analysis:**
    *   Run ESLint, Prettier, TypeScript compiler checks.
    *   Ensure code quality and consistency.
2.  **Unit & Integration Tests:**
    *   Execute automated tests (e.g., Vitest, Playwright as included in Epic Stack).
    *   Verify application correctness.
3.  **Build:**
    *   Build the Remix application (`pnpm build`).
    *   Create a production-ready Docker image for the application.
4.  **Push to Registry:**
    *   Tag the Docker image appropriately (e.g., with Git commit SHA, version number).
    *   Push the image to a container registry (e.g., GitLab Container Registry, Docker Hub, Azure Container Registry).
5.  **Changelog Generation:**
    *   Automatically generate/update changelogs based on Conventional Commits.
6.  **Deployment (to Staging/Production Kubernetes):**
    *   Initially, this might be a manual trigger from GitLab CI after successful pipeline completion.
    *   Scripts (e.g., `kubectl apply -f k8s-manifests/`) will be used to deploy the new image to the Kubernetes cluster.
    *   Secrets and configurations will be managed via Kubernetes secrets and configmaps, populated from environment variables in GitLab CI.

**`.gitlab-ci.yml` (Conceptual Outline):**

```yaml
stages:
  - validate
  - test
  - build
  - publish
  - deploy # Manual trigger initially

variables:
  IMAGE_TAG: $CI_COMMIT_SHA
  # Other variables for registry, K8s context, etc.

lint:
  stage: validate
  script:
    - pnpm install
    - pnpm lint

typescript_check:
  stage: validate
  script:
    - pnpm install
    - pnpm typecheck

unit_tests:
  stage: test
  script:
    - pnpm install
    - pnpm test:unit # Assuming a script for unit tests

integration_tests:
  stage: test
  script:
    - pnpm install
    # Setup test DB, Redis, RabbitMQ if needed for integration tests
    - pnpm test:integration # Assuming a script for integration tests

build_docker:
  stage: build
  script:
    - pnpm install
    - pnpm build
    - docker build -t $CI_REGISTRY_IMAGE:$IMAGE_TAG .
  # Needs Docker-in-Docker or similar setup on GitLab runner

publish_docker:
  stage: publish
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker push $CI_REGISTRY_IMAGE:$IMAGE_TAG
  needs: [build_docker]

# ... (Changelog job) ...

deploy_staging:
  stage: deploy
  script:
    - echo "Deploying $IMAGE_TAG to Staging Kubernetes..."
    # - ./scripts/deploy.sh staging $IMAGE_TAG
  when: manual
  environment:
    name: staging

deploy_production:
  stage: deploy
  script:
    - echo "Deploying $IMAGE_TAG to Production Kubernetes..."
    # - ./scripts/deploy.sh production $IMAGE_TAG
  when: manual
  environment:
    name: production
```

### Phase 2: Migration to GitHub Actions / Azure DevOps (Future)

As the project matures or if there's a strategic shift towards Azure, the CI/CD pipeline will be migrated.

**Key Considerations for Migration:**

*   **GitHub Actions:**
    *   Workflows defined in YAML files within `.github/workflows`.
    *   Leverage GitHub-hosted runners or self-hosted runners.
    *   Strong integration with GitHub repositories, issues, and pull requests.
    *   Good support for Azure deployments via Azure Login action and Azure CLI.
*   **Azure DevOps Pipelines:**
    *   Pipelines defined in YAML or via the classic editor.
    *   Hosted agents (Microsoft-hosted) or self-hosted agents.
    *   Deep integration with Azure services (Azure Repos, Azure Boards, Azure Artifacts, Azure App Service, AKS).
    *   Excellent for managing complex release workflows and approvals.

**General Steps for Migration (Applicable to both):**

1.  **Translate GitLab CI Stages:** Map existing stages (lint, test, build, publish, deploy) to the equivalent constructs in the target platform.
2.  **Authentication & Secrets:** Configure secure access to container registries, Kubernetes clusters, and Azure services.
3.  **Environment Variables:** Adapt environment variable management.
4.  **Runner/Agent Configuration:** Ensure runners/agents have the necessary tools (Node.js, pnpm, Docker, kubectl, Azure CLI).
5.  **Infrastructure as Code (IaC):** Consider adopting IaC tools like Bicep or Terraform for managing Azure resources, integrated into the pipeline.
6.  **Deployment Strategies:** Implement more advanced deployment strategies like blue/green or canary deployments if required.

**Example Snippets (Conceptual):**

**GitHub Actions (`.github/workflows/main.yml`):**
```yaml
name: PRRIP CI/CD

on: [push]

jobs:
  # ... (lint, test, build jobs similar to GitLab CI) ...

  deploy_to_aks:
    runs-on: ubuntu-latest
    needs: [build_job] # Assuming a build job that produces a Docker image
    steps:
      - name: Azure Login
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}
      - name: Deploy to AKS
        uses: Azure/k8s-deploy@v4 # Example action
        with:
          # ... (configuration for AKS cluster, image, manifests) ...
```

**Azure DevOps Pipelines (`azure-pipelines.yml`):**
```yaml
trigger:
- main

pool:
  vmImage: 'ubuntu-latest'

stages:
- stage: Build
  jobs:
  - job: BuildApp
    steps:
    # ... (lint, test, build steps) ...
    - task: Docker@2
      inputs:
        # ... (build and push Docker image to ACR) ...

- stage: Deploy
  jobs:
  - job: DeployToAKS
    steps:
    - task: KubernetesManifest@0 # Example task
      inputs:
        # ... (configuration for AKS service connection, manifests, image) ...
```

## Environment Management

*   **Development:** Local Docker Compose, `.env` files (gitignored).
*   **Staging & Production:** Environment variables injected into Kubernetes pods by the CI/CD system (from GitLab CI variables, GitHub Secrets, or Azure DevOps variable groups/Key Vault).

This phased approach allows PRRIP to start with a functional CI/CD pipeline quickly and evolve it as the project and its deployment targets mature.