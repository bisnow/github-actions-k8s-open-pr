# Deploy PR Stack with Helm

Deploys Laravel PR stacks to Kubernetes using Helm. Handles AWS authentication, deployment, verification, and connection testing.

## Usage

```yaml
  deploy-pr-stack:
    name: Deploy PR Stack with Helm
    runs-on: arc-runners-bisnow
    timeout-minutes: 30
    needs: [create-multi-arch-manifest, deploy-cloudformation]
    if: needs.create-multi-arch-manifest.result == 'success' && needs.deploy-cloudformation.result == 'success'
    steps:
      - name: Deploy PR Stack
        uses: bisnow/github-actions-k8s-open-pr@main
        with:
          pr-number: ${{ github.event.pull_request.number }}
          image-tag: ${{ env.TAG }}
          app-name: ${{ env.APP_NAME }}
          cloudformation-stack-name: ${{ needs.deploy-cloudformation.outputs.stack-name }}
          helm-chart-version: 1.1.2
          namespace: "biscred-pr-stacks"
          values-file-path: '.k8s/overlays/pr/values.yaml'
```

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `pr-number` | PR number | Yes | - |
| `image-tag` | Docker image tag | Yes | - |
| `app-name` | Application name | Yes | - |
| `cloudformation-stack-name` | CloudFormation stack name | Yes | - |
| `aws-account` | AWS account | No | `bisnow` |
| `eks-cluster` | EKS cluster | No | `bisnow-non-prod-eks` |
| `namespace` | Kubernetes namespace | No | `biscred-pr-stacks` |
| `helm-chart-version` | Helm chart version | No | `1.1.0` |
| `values-file-path` | Helm values file path | No | `.k8s/overlays/pr/values.yaml` |
| `ecr-registry` | ECR registry URL | No | `560285300220.dkr.ecr.us-east-1.amazonaws.com` |

## Outputs

| Output | Description |
|--------|-------------|
| `app-url` | Deployed application URL |

## Prerequisites

This action requires:
- AWS credentials configured (via OIDC or other method)
- Helm installed on the runner
- Access to the specified EKS cluster
- The Helm chart available in the ECR registry
- The specified namespace exists in the cluster

## What It Does

1. Checks out repository and assumes AWS role
2. Configures kubectl for EKS cluster
3. Authenticates Helm with ECR
4. Retrieves IAM role ARN from CloudFormation stack
5. Deploys application using Helm with specified values
6. Queries ingress for deployed URL
7. Verifies pods are ready
8. Tests database and Redis connections
9. Outputs deployment summary to GitHub Actions

## Versioning

This action uses rolling major version tags. You can pin to:

- A specific version: `@v3.1.0` (exact, never changes)
- A major version: `@v3` (recommended, gets bug fixes and new features)

When a new semantic version tag (e.g., `v3.2.0`) is pushed, a GitHub Actions workflow automatically updates the corresponding major version tag (`v3`) to point to the new release.