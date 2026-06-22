# FinsOpsIQ App CI/CD Architecture

## Workflow Architecture

```text
FinOpsIQ-App
  ├─ ci-scan.yml
  │    └─ calls FinOPsIQ-Workflows/app-ci-scan.yml
  │         ├─ dorny/paths-filter
  │         ├─ SonarCloud per changed service
  │         └─ Snyk per changed service
  │
  ├─ ci-build.yml
  │    └─ calls FinOPsIQ-Workflows/app-ci-build.yml
  │         ├─ dorny/paths-filter
  │         ├─ Docker build only changed services
  │         ├─ Trivy scan
  │         ├─ Push passed images to ACR
  │         └─ Update FinOpsIQ-Helm values-dev.yaml through yq
  │
  └─ release.yml
       └─ calls FinOPsIQ-Workflows/app-release-promote.yml
            ├─ Resolve release tag source SHA
            ├─ Pull existing scanned SHA image from ACR
            ├─ Retag image with release version
            └─ Push release tag without rebuilding
```

## Path Filter Configuration

| Filter | Paths |
|---|---|
| frontend | `frontend/**` |
| api-gateway | `api-gateway/**`, `services/api-gateway/**`, `src/gateway_service/**` |
| auth-service | `auth-service/**`, `services/auth-service/**`, `src/auth_service/**`, `src/onboarding/**` |
| collection-service | `collection-service/**`, `services/collection-service/**`, `src/collection_service/**` |
| processing-service | `processing-service/**`, `services/processing-service/**`, `src/processing_service/**` |
| ai-service | `ai-service/**`, `services/ai-service/**`, `src/ai_service/**` |
| notification-service | `notification-service/**`, `services/notification-service/**`, `src/notification_service/**` |
| shared | `src/common/**`, `src/adapters/**`, `src/config/**`, `requirements*.txt` |

Shared changes build all backend services because shared Python code can affect every backend workload.

## Service Build Matrix

| Service | Dockerfile | Context | Helm key |
|---|---|---|---|
| frontend | `frontend/Dockerfile` | `frontend` | `frontend` |
| api-gateway | `services/api-gateway/Dockerfile` | `.` | `apiGateway` |
| auth-service | `services/auth-service/Dockerfile` | `.` | `auth` |
| collection-service | `services/collection-service/Dockerfile` | `.` | `collection` |
| processing-service | `services/processing-service/Dockerfile` | `.` | `processing` |
| ai-service | `services/ai-service/Dockerfile` | `.` | `ai` |
| notification-service | `services/notification-service/Dockerfile` | `.` | `notification` |

## Image Tagging Strategy

Development build images use the first six characters of the merged commit SHA.

Example:

```text
acrfinopsiqdev.azurecr.io/finopsiq/ai-service:a453bd
```

Release promotion does not rebuild images. It retags the already scanned SHA image.

Example:

```text
ai-service:a453bd → ai-service:v1.0.0
```

## Helm Update Strategy

`ci-build.yml` updates only rebuilt services in:

```text
FinOpsIQ-Helm/charts/finopsiq/values-dev.yaml
```

The workflow uses `yq`:

```text
.services.<helmKey>.image.tag = <sha6>
```

Unchanged services keep their existing image tags.

## Release Promotion Flow

```text
GitHub Release published: v1.0.0
  ↓
Resolve release tag commit SHA
  ↓
Find ACR images with tag <sha6>
  ↓
Pull existing images
  ↓
Retag as v1.0.0
  ↓
Push release tags
  ↓
Update GitHub Release metadata
```

## Slack Notification Format

Failure notifications include:

- Repository
- Workflow URL
- Failed stage
- Service name when applicable
- Commit SHA or release tag
- SonarCloud/Snyk/Trivy summary where applicable

Success notifications include:

- Image tag
- Services built or released
- ACR tags
- Workflow URL

## Required Secrets and Variables

Secrets:

- `SONAR_TOKEN`
- `SNYK_TOKEN`
- `SLACK_WEBHOOK_URL`
- `AZURE_CLIENT_ID`
- `AZURE_TENANT_ID`
- `AZURE_SUBSCRIPTION_ID`
- `HELM_REPO_TOKEN`

Variables:

- `ACR_LOGIN_SERVER`
- `SONAR_ORGANIZATION`
- `SONAR_PROJECT_PREFIX`
- `HELM_REPOSITORY`
