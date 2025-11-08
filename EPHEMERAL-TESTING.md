# Ephemeral Cluster Testing with FluxCD

This guide explains how to use ephemeral GKE clusters for automated testing with FluxCD deployments.

## Overview

Ephemeral clusters are temporary Kubernetes clusters created on-demand for testing purposes and destroyed after tests complete. This approach provides:

- **Isolated Testing**: Each test run gets a clean, isolated environment
- **Cost Efficiency**: Clusters only exist during test execution
- **Reproducibility**: Consistent infrastructure for every test
- **Parallel Testing**: Run multiple test suites simultaneously
- **Production Parity**: Test on real GKE infrastructure

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  Application Repository                                     │
│  └─ .github/workflows/test.yml                              │
│     1. Creates ephemeral cluster                            │
│     2. Bootstraps FluxCD                                    │
│     3. Deploys application                                  │
│     4. Runs tests                                           │
│     5. Destroys cluster (always)                            │
└─────────────────────────────────────────────────────────────┘
                         │
                         ├─ (uses reusable workflow)
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  terraform-infra Repository                                 │
│  ├─ create-ephemeral-cluster.yml (reusable)                 │
│  └─ destroy-ephemeral-cluster.yml (reusable)                │
└─────────────────────────────────────────────────────────────┘
                         │
                         ├─ (uses reusable workflow)
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  helm-cd Repository (this repo)                             │
│  ├─ bootstrap-flux-ephemeral.yml (reusable)                 │
│  └─ clusters/ephemeral/                                     │
│     ├─ flux-system/                                         │
│     ├─ infrastructure.yaml                                  │
│     └─ environments.yaml                                    │
└─────────────────────────────────────────────────────────────┘
```

## Prerequisites

### Repository Setup

You need access to three repositories:

1. **terraform-infra**: Infrastructure-as-code for GKE clusters
2. **helm-cd**: FluxCD configuration and Helm releases (this repo)
3. **your-app**: Your application repository (where tests run)

### Required Secrets and Variables

Configure these in your application repository:

**Secrets:**
- `GCP_SA_KEY`: GCP Service Account key (JSON) with permissions:
  - `roles/container.admin` (GKE cluster management)
  - `roles/compute.networkAdmin` (VPC management)
  - `roles/iam.serviceAccountUser`
- `FLUX_GITHUB_TOKEN`: GitHub Personal Access Token with `repo` scope

**Variables:**
- `GCP_PROJECT_ID`: Your GCP project ID

### GCP Project Setup

Ensure your GCP project has:
- GKE API enabled
- Compute Engine API enabled
- Sufficient quota for clusters and nodes
- Cloud Storage bucket for Terraform state (optional but recommended)

## Quick Start

### 1. Copy the Example Workflow

Copy `.github/workflows/example-ephemeral-test.yml` from this repo to your application repo:

```bash
# In your application repository
mkdir -p .github/workflows
cp ../helm-cd/.github/workflows/example-ephemeral-test.yml \
   .github/workflows/ephemeral-test.yml
```

### 2. Update Repository References

Edit `.github/workflows/ephemeral-test.yml` and replace:

```yaml
env:
  TERRAFORM_INFRA_REPO: 'your-org/terraform-infra'  # Your infra repo
  HELM_CD_REPO: 'your-org/helm-cd'                  # Your FluxCD repo
```

Update workflow references:
```yaml
uses: your-org/terraform-infra/.github/workflows/create-ephemeral-cluster.yml@main
uses: your-org/helm-cd/.github/workflows/bootstrap-flux-ephemeral.yml@main
uses: your-org/terraform-infra/.github/workflows/destroy-ephemeral-cluster.yml@main
```

### 3. Customize Your Tests

Modify the `run-tests` job in the workflow to include your specific tests:

```yaml
- name: Run Integration Tests
  run: |
    # Your test commands here
    npm test
    # or
    pytest tests/integration/
    # or
    go test ./...
```

### 4. Trigger the Workflow

The workflow can be triggered:
- **Automatically** on pull requests: `on: pull_request`
- **Manually**: `on: workflow_dispatch`
- **On push** to specific branches

## Customizing FluxCD Configuration

### Option 1: Use Default Ephemeral Config (Simplest)

The default configuration deploys a sample Nginx application. No changes needed!

### Option 2: Customize for Your Application

Edit the files in `environments/ephemeral/`:

**environments/ephemeral/app-release.yaml**
```yaml
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: my-app
  namespace: flux-system
spec:
  chart:
    spec:
      chart: my-app-chart
      version: '>=1.0.0'
      sourceRef:
        kind: HelmRepository
        name: my-repo
  targetNamespace: ephemeral
  values:
    # Your app configuration
    image:
      tag: ${{ github.sha }}  # Use commit SHA
```

**environments/ephemeral/istio-gateway.yaml**
```yaml
hosts:
  - "test.example.com"  # Change to your domain
```

### Option 3: Use Dynamic Configuration

Create a temporary branch or path for each test run:

```yaml
- name: Prepare FluxCD Config
  run: |
    # Create a unique branch for this test
    git checkout -b test-pr-${{ github.event.pull_request.number }}

    # Update app version to current commit
    sed -i "s/tag: .*/tag: ${{ github.sha }}/" \
      environments/ephemeral/app-release.yaml

    git commit -am "Update to commit ${{ github.sha }}"
    git push origin test-pr-${{ github.event.pull_request.number }}

- name: Bootstrap FluxCD
  uses: your-org/helm-cd/.github/workflows/bootstrap-flux-ephemeral.yml@main
  with:
    flux_branch: test-pr-${{ github.event.pull_request.number }}
```

## Advanced Usage

### Testing with Custom Infrastructure

Modify the cluster creation to include additional infrastructure:

```yaml
create-cluster:
  uses: your-org/terraform-infra/.github/workflows/create-ephemeral-cluster.yml@main
  with:
    cluster_identifier: pr-${{ github.event.pull_request.number }}
    machine_type: 'e2-standard-4'  # Larger nodes
    min_nodes: 2                    # More nodes
    max_nodes: 5
    enable_bastion: true            # Enable bastion for debugging
```

### Running Tests Against Different Environments

Create multiple test jobs for different configurations:

```yaml
test-matrix:
  strategy:
    matrix:
      config:
        - name: minimal
          machine_type: e2-small
          nodes: 1
        - name: production-like
          machine_type: e2-standard-4
          nodes: 3
```

### Debugging Failed Tests

If tests fail and you need to debug:

1. **Keep the cluster alive temporarily:**
   ```yaml
   cleanup:
     if: success()  # Only cleanup on success
   ```

2. **Get cluster credentials locally:**
   ```bash
   gcloud container clusters get-credentials ephemeral-pr-123-cluster \
     --region us-central1 \
     --project your-project-id
   ```

3. **Check logs:**
   ```bash
   kubectl logs -n ephemeral --all-containers -l app=your-app
   flux logs --all-namespaces
   ```

4. **Manual cleanup when done:**
   ```bash
   # Run the destroy workflow manually from GitHub UI
   ```

### Cost Optimization

Ephemeral clusters are designed to be cost-effective:

- **Preemptible nodes**: 80% cheaper (enabled by default)
- **Zonal clusters**: Cheaper than regional
- **Auto-cleanup**: Always destroy after tests
- **Right-sized**: Use minimal machine types

**Estimated costs** (us-central1):
- `e2-medium` (2 vCPU, 4GB): ~$0.03/hour (preemptible)
- 1-hour test run: ~$0.03
- Typical PR with 5 test runs: ~$0.15

### Monitoring and Alerts

Set up alerts for:
- Orphaned clusters (not destroyed)
- Long-running tests (timeout)
- High costs

```yaml
- name: Check Test Duration
  run: |
    DURATION=${{ github.event.workflow_run.duration }}
    if [ $DURATION -gt 7200 ]; then  # 2 hours
      echo "::error::Test exceeded 2-hour limit"
    fi
```

## Troubleshooting

### Cluster Creation Fails

**Check quota:**
```bash
gcloud compute project-info describe --project=YOUR_PROJECT
```

**Common issues:**
- Insufficient IP address space
- GKE API not enabled
- Invalid service account permissions

### FluxCD Bootstrap Fails

**Check prerequisites:**
```bash
flux check --pre
```

**Common issues:**
- Invalid GitHub token
- Repository doesn't exist
- Branch protection rules blocking commits

### Application Not Deploying

**Check FluxCD reconciliation:**
```bash
flux get all
kubectl get helmreleases -n flux-system
```

**Common issues:**
- Invalid Helm chart reference
- Missing Helm repository
- Namespace not created

### Tests Fail to Connect

**Verify ingress:**
```bash
kubectl get svc -n istio-system istio-ingressgateway
kubectl get gateway,virtualservice -n ephemeral
```

**Test from within cluster:**
```bash
kubectl run test --rm -it --image=curlimages/curl -- \
  curl http://my-app.ephemeral.svc.cluster.local
```

### Cleanup Fails

If cleanup fails, manually destroy resources:

```bash
# Delete cluster
gcloud container clusters delete ephemeral-pr-123-cluster \
  --region us-central1

# Delete network
gcloud compute networks delete ephemeral-pr-123-vpc
```

## Best Practices

1. **Always Use `if: always()` for Cleanup**
   ```yaml
   cleanup:
     if: always()  # Run even if tests fail
   ```

2. **Use Unique Identifiers**
   - PR number: `pr-${{ github.event.pull_request.number }}`
   - Commit SHA: `commit-${{ github.sha }}`
   - Run ID: `run-${{ github.run_id }}`

3. **Keep Tests Fast**
   - Use minimal infrastructure
   - Run only essential tests on ephemeral clusters
   - Use unit tests locally, integration tests on ephemeral

4. **Monitor Costs**
   - Set up billing alerts
   - Track cluster creation/destruction
   - Use labels for cost attribution

5. **Security**
   - Rotate service account keys regularly
   - Use least-privilege IAM roles
   - Scan images before deployment
   - Enable binary authorization (optional)

## Example Workflows

### PR-Based Testing

```yaml
on:
  pull_request:
    types: [opened, synchronize, reopened]
```

### Scheduled Testing

```yaml
on:
  schedule:
    - cron: '0 2 * * *'  # Nightly at 2 AM
```

### Manual Testing with Parameters

```yaml
on:
  workflow_dispatch:
    inputs:
      machine_type:
        description: 'GKE machine type'
        required: false
        default: 'e2-medium'
      test_suite:
        description: 'Test suite to run'
        required: false
        default: 'all'
```

## Next Steps

- Review the [FluxCD documentation](https://fluxcd.io/)
- Explore [GKE best practices](https://cloud.google.com/kubernetes-engine/docs/best-practices)
- Set up [monitoring and logging](https://cloud.google.com/stackdriver/docs)
- Implement [progressive delivery with Flagger](https://fluxcd.io/flagger/)

## Support

For issues:
- **terraform-infra**: Infrastructure and cluster creation
- **helm-cd**: FluxCD configuration and deployments
- **your-app**: Application-specific test failures
