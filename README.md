# FinsOpsIQ Reusable Workflow Hub

This repository owns the centralized reusable GitHub Actions workflows for FinsOpsIQ.

Application, Helm, and Infrastructure repositories contain thin caller workflows only. They pass explicit inputs and named secrets to this repository using `workflow_call`.

No caller workflow uses `secrets: inherit`.

## Reusable Workflows

| Reusable workflow | Purpose | Called by |
| --- | --- | --- |
| `.github/workflows/app-pr-validation.yml` | PR validation, SonarCloud, Snyk, path-based Docker build, Trivy, dev image push, Slack | `FinOpsIQ-App/.github/workflows/build.yml` |
| `.github/workflows/app-release-build.yml` | Semantic release, path-based release image build, Trivy, ACR push, selective Helm values update, Slack | `FinOpsIQ-App/.github/workflows/release.yml` |
| `.github/workflows/helm-aks-deploy.yml` | Values-driven Helm deploy, rollout validation, pod health, ingress health | `FinOpsIQ-Helm/.github/workflows/aks-deploy.yml` |

## Workflow Dependency Diagram

```text
FinOpsIQ-App
├─ .github/workflows/build.yml
│  └─ calls FinOPsIQ-Workflows/.github/workflows/app-pr-validation.yml
│     ├─ dorny/paths-filter
│     ├─ SonarCloud
│     ├─ Snyk
│     ├─ Docker build changed images only
│     ├─ Trivy image scan
│     ├─ ACR push sha-<commit>
│     └─ Slack notification
│
└─ .github/workflows/release.yml
   └─ calls FinOPsIQ-Workflows/.github/workflows/app-release-build.yml
      ├─ dorny/paths-filter
      ├─ semantic version
      ├─ Docker build changed images only
      ├─ Trivy image scan
      ├─ ACR push vX.Y.Z
      ├─ GitHub Release
      ├─ selective FinOpsIQ-Helm values-dev.yaml tag update
      └─ Slack notification

FinOpsIQ-Helm
└─ .github/workflows/aks-deploy.yml
   └─ calls FinOPsIQ-Workflows/.github/workflows/helm-aks-deploy.yml
      ├─ read image tags from values-dev.yaml / values-prod.yaml
      ├─ az aks get-credentials
      ├─ helm upgrade --install
      ├─ rollout validation
      ├─ pod readiness
      └─ ingress health

FinOpsIQ-Infra
└─ owns standalone Terraform workflows directly
   ├─ .github/workflows/terraform-infra.yml
   └─ .github/workflows/bootstrap-backend.yml
```

## Explicit Secret Passing

Caller repositories pass only the secrets needed by the selected reusable workflow.

Examples:

```yaml
secrets:
  azure_client_id: ${{ secrets.AZURE_CLIENT_ID }}
  azure_tenant_id: ${{ secrets.AZURE_TENANT_ID }}
  azure_subscription_id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

Application workflows additionally pass:

```yaml
sonar_token: ${{ secrets.SONAR_TOKEN }}
snyk_token: ${{ secrets.SNYK_TOKEN }}
helm_repo_token: ${{ secrets.HELM_REPO_TOKEN }}
slack_webhook_url: ${{ secrets.SLACK_WEBHOOK_URL }}
```

## Path-Based Build Strategy

The application reusable workflows build only changed service images.

| Filter | Paths | Images rebuilt |
| --- | --- | --- |
| `frontend` | `frontend/**` | `frontend` |
| `api_gateway` | `src/gateway_service/**` | `api-gateway` |
| `auth` | `src/auth_service/**`, `src/onboarding/**` | `auth-service` |
| `collection` | `src/collection_service/**` | `collection-service` |
| `processing` | `src/processing_service/**` | `processing-service` |
| `ai` | `src/ai_service/**` | `ai-service` |
| `notification` | `src/notification_service/**` | `notification-service` |
| `shared` | `src/common/**`, `src/adapters/**`, `src/config/**` | all backend services |
| `helm` | `helm/**`, `charts/**` | none |
| `docs` | `docs/**`, `**/*.md` | none |

Release builds update only the Helm image tags for services that were rebuilt.
