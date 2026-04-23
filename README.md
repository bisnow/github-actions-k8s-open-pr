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
| `atomic` | Deploy with `helm --atomic --cleanup-on-fail`. See [Deploy reliability](#deploy-reliability) below. | No | `'true'` |
| `sealed-secrets-timeout` | Seconds to wait for the sealed-secrets controller to materialize each target Secret. | No | `'60'` |

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
5. Applies sealed secrets and waits for the sealed-secrets controller to materialize the
   target `Secret`s (bounded by `sealed-secrets-timeout`)
6. Runs preflight recovery to unstick a previously wedged Helm release (see
   [Deploy reliability](#deploy-reliability))
7. Deploys application using Helm with `--atomic --cleanup-on-fail` (unless `atomic: 'false'`),
   dumping describe/events/logs on failure before exiting
8. Resolves the deployed URL using a three-tier lookup:
   - **Standard Ingress** (chart v1 / ALB)
   - **Traefik `IngressRoute` CRD** (chart v2) — extracts the hostname from the `Host(...)` match rule
   - **Estimated URL** — falls back to `{app-name}-pr-{pr-number}.non-prod.bisnow.cloud` if neither resource is found, and emits a warning in the job summary
9. Tests database and Redis connections
10. Outputs deployment summary to GitHub Actions

## Deploy reliability

The action includes three reliability behaviors to keep a stuck or broken deploy from
wedging the Helm release for subsequent CI runs.

### 1. Preflight recovery

Before each deploy, the action runs `helm status <release>` and checks for a pending
state (`pending-upgrade`, `pending-install`, `pending-rollback`) — the state Helm is left
in when a previous `helm upgrade --wait` is interrupted (timeout, cancelled workflow,
runner killed). If a pending state is detected:

- If a prior `deployed` revision exists, the action runs `helm rollback` to that
  revision.
- Otherwise (first-time install that never succeeded), it deletes the pending-state
  release secrets (`-l owner=helm,name=<release>,status=pending-*`) so the next
  `helm upgrade --install` can proceed.

This step is idempotent and never fails the workflow on its own — if recovery doesn't
work, the real deploy step still runs and surfaces the underlying error.

### 2. Atomic deploys (default)

With `atomic: 'true'` (default), the action passes `--atomic --cleanup-on-fail` to
`helm upgrade --install`. A failed deploy is automatically rolled back and does **not**
leave the release in a `pending-upgrade` state.

**Tradeoff:** atomic rollback also removes the broken pods, so you lose the in-cluster
state useful for post-mortem `kubectl describe` / `kubectl logs` against a live pod.
The failure-diagnostics step (below) captures describe/events/logs **before** the
rollback runs, so you still get the key signals in the workflow output. If you need the
broken pods to stick around for live `kubectl` inspection, set `atomic: 'false'` for
that run.

### 3. Failure diagnostics

When `helm upgrade` returns a non-zero exit code, the action captures — inside collapsible
`::group::` blocks and before re-exiting with the original code:

- `kubectl describe deploy` for matching deployments
- The last 100 events in the namespace (sorted by `.lastTimestamp`)
- For each not-ready container across matching pods: both `kubectl logs --previous` and
  `kubectl logs --tail=200`

This means failures like "app boots, throws an exception, container crashloops" show up
with the actual PHP/app error in the workflow log — no `kubectl` access required to
diagnose.

### 4. Real sealed-secrets wait

`Apply Sealed Secrets` no longer sleeps for a fixed interval. Instead it enumerates the
`SealedSecret` resources from the kustomization and polls the namespace (up to
`sealed-secrets-timeout` seconds, default 60) until the target `Secret`s exist. On
timeout, the step fails with a clear message listing which Secrets are still missing.

## Versioning

This action uses rolling major version tags. You can pin to:

- A specific version: `@v3.1.0` (exact, never changes)
- A major version: `@v3` (recommended, gets bug fixes and new features)

When a new semantic version tag (e.g., `v3.2.0`) is pushed, a GitHub Actions workflow automatically updates the corresponding major version tag (`v3`) to point to the new release.