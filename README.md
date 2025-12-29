# Deploy PR Stack with Helm

A composite GitHub Action for deploying Laravel PR stacks to Kubernetes using Helm.

## Features

- Deploys Laravel applications to Kubernetes using Helm
- Configures kubectl and authenticates with AWS EKS
- Authenticates Helm with AWS ECR for pulling OCI charts
- Verifies deployment by checking pod readiness
- Tests database and Redis connections
- Queries deployed ingress for actual application URL
- Outputs deployment summary to GitHub Actions summary

## Usage

```yaml
- name: Deploy PR Stack
  uses: ignisware/github-actions-k8s-open-pr@main
  with:
    pr-number: ${{ github.event.pull_request.number }}
    image-tag: ${{ env.TAG }}
    app-name: leads-test
    cloudformation-stack-name: ${{ needs.deploy-cloudformation.outputs.stack-name }}
```

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `pr-number` | Pull request number | Yes | - |
| `image-tag` | Docker image tag to deploy | Yes | - |
| `app-name` | Application name (used for resource queries to prevent namespace collisions) | Yes | - |
| `cloudformation-stack-name` | CloudFormation stack name to query for IAM role ARN | Yes | - |
| `aws-account` | AWS account name | No | `bisnow` |
| `eks-cluster` | EKS cluster name | No | `bisnow-non-prod-eks` |
| `namespace` | Kubernetes namespace | No | `biscred-pr-stacks` |
| `helm-chart-version` | Helm chart version | No | `1.1.0` |
| `values-file-path` | Path to Helm values file | No | `.k8s/overlays/pr/values.yaml` |
| `ecr-registry` | ECR registry URL | No | `560285300220.dkr.ecr.us-east-1.amazonaws.com` |

## Outputs

| Output | Description |
|--------|-------------|
| `app-url` | The deployed application URL (queried from ingress) |

## Prerequisites

This action requires:
- AWS credentials configured (via OIDC or other method)
- Helm installed on the runner
- Access to the specified EKS cluster
- The Helm chart available in the ECR registry
- The specified namespace exists in the cluster

## What It Does

1. **Checks out repository** - Ensures the workflow has access to values files
2. **Assumes AWS role** - Uses bisnow's custom action for AWS authentication
3. **Configures kubectl** - Updates kubeconfig for the specified EKS cluster
4. **Authenticates Helm with ECR** - Logs Helm into the ECR OCI registry
5. **Gets IAM role ARN** - Queries CloudFormation stack outputs for the actual role ARN
6. **Deploys with Helm** - Runs `helm upgrade --install` with the queried role ARN
7. **Gets deployed URL** - Queries the ingress resource for the actual deployed hostname
8. **Verifies deployment** - Waits for pods to be ready
9. **Tests connections** - Validates database and Redis connectivity using Laravel Tinker
10. **Outputs summary** - Creates a GitHub Actions summary with deployment details

## Example with Custom Values

```yaml
- name: Deploy PR Stack
  uses: ignisware/github-actions-k8s-open-pr@main
  with:
    pr-number: ${{ github.event.pull_request.number }}
    image-tag: pr-123-45
    app-name: my-app
    cloudformation-stack-name: ${{ needs.deploy-cloudformation.outputs.stack-name }}
    helm-chart-version: 1.2.0
    namespace: custom-pr-stacks
```
