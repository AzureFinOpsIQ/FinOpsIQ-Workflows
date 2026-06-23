# FinsOpsIQ Reusable Workflow Hub

This repository owns the centralized reusable GitHub Actions workflows for FinsOpsIQ.

Application and Helm repositories contain thin caller workflows only. They pass explicit inputs and named secrets to this repository using `workflow_call`.

No caller workflow uses `secrets: inherit`.

## Reusable Workflows

| Reusable workflow | Purpose | Called by |
|---|---|---|
| `.github/workflows/app-ci-scan.yml` | PR service change detection, service lint/test, repository-level SonarCloud Quality Gate, Snyk dependency scan, Slack failures | `FinOpsIQ-App/.github/workflows/ci-scan.yml` |
| `.github/workflows/app-ci-build.yml` | Merged PR build with `build` label, path-based Docker build, Trivy, ACR push, selective Helm dev tag update | `FinOpsIQ-App/.github/workflows/ci-build.yml` |
| `.github/workflows/app-release-promote.yml` | Release tag promotion by retagging existing scanned SHA images without rebuild | `FinOpsIQ-App/.github/workflows/release.yml` |
| `.github/workflows/helm-aks-deploy.yml` | Values-driven Helm deploy, rollout validation, pod health, ingress health | `FinOpsIQ-Helm/.github/workflows/aks-deploy.yml` |

## Workflow Dependency Diagram

```text
FinOpsIQ-App
  -> ci-scan.yml
     -> FinOPsIQ-Workflows/app-ci-scan.yml
        -> dorny/paths-filter
        -> SonarCloud repository scan
        -> Snyk per changed service

  -> ci-build.yml
     -> FinOPsIQ-Workflows/app-ci-build.yml
        -> requires merged PR to main
        -> requires PR label: build
        -> dorny/paths-filter
        -> build changed images only
        -> Trivy image scan
        -> push SHA-tagged images to ACR
        -> update FinOpsIQ-Helm dev-values.yaml with yq

  -> release.yml
     -> FinOPsIQ-Workflows/app-release-promote.yml
        -> resolve release tag source SHA
        -> find existing SHA image in ACR
        -> retag SHA image as release version
        -> push release tag
        -> update release metadata

FinOpsIQ-Helm
  -> aks-deploy.yml
     -> FinOPsIQ-Workflows/helm-aks-deploy.yml
        -> read image tags from dev-values.yaml / prod-values.yaml
        -> helm upgrade --install
        -> rollout, pod, and ingress validation

FinOpsIQ-Infra
  -> owns standalone Terraform workflows directly
```

## Explicit Secret Passing

Caller repositories pass only the secrets needed by each reusable workflow.

Application workflows pass:

```yaml
secrets:
  azure_client_id: ${{ secrets.AZURE_CLIENT_ID }}
  azure_tenant_id: ${{ secrets.AZURE_TENANT_ID }}
  azure_subscription_id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
  sonar_token: ${{ secrets.SONAR_TOKEN }}
  snyk_token: ${{ secrets.SNYK_TOKEN }}
  helm_repo_token: ${{ secrets.HELM_REPO_TOKEN }}
  slack_webhook_url: ${{ secrets.SLACK_WEBHOOK_URL }}
```

## Path-Based Strategy

| Filter | Paths | Action |
|---|---|---|
| `frontend` | `frontend/**` | frontend only |
| `api_gateway` | `api-gateway/**` | api-gateway only |
| `auth` | `auth-service/**` | auth-service only |
| `collection` | `collection-service/**` | collection-service only |
| `processing` | `processing-service/**` | processing-service only |
| `ai` | `ai-service/**` | ai-service only |
| `notification` | `notification-service/**` | notification-service only |
| `shared` | `shared-lib/**, requirements/**` | all backend services |

Helm-only and documentation-only changes do not build images in the application pipelines.

## Image Tagging

CI build tag:

```text
<acr-login-server>/finopsiq/<service>:<first6-merge-commit-sha>
```

Release promotion tag:

```text
<acr-login-server>/finopsiq/<service>:vX.Y.Z
```

Release promotion never rebuilds images; it retags previously scanned SHA images.
