# SUSE AI Fleet Repository

This repository contains Kubernetes manifests for deploying the SUSE AI stack using Rancher Fleet. The deployment includes:

- NVIDIA GPU Operator
- Ollama
- Milvus
- Open WebUI

## Repository Structure

```
fleet-resources/
├── fleet.yaml                  # Main Fleet configuration with deployment order
├── gpu-operator-direct/        # Direct deployment of GPU Operator
│   ├── fleet.yaml              # Namespace targeting configuration
│   └── operator/               # GPU Operator resources
│       ├── gpu-operator.yaml   # HelmChart resource for GPU Operator
│       └── test-pod.yaml       # Test pod to validate GPU access
├── suse-ai-setup/              # SUSE AI namespace and secret setup
│   ├── fleet.yaml              # Empty namespace configuration
│   ├── namespace.yaml          # suse-ai namespace definition
│   └── registry-secret.yaml    # Registry secret template
├── suse-ai/                    # Resources for suse-ai namespace
│   └── charts/
│       ├── ollama/             # Ollama chart configuration
│       │   └── fleet.yaml
│       ├── milvus/             # Milvus chart configuration
│       │   └── fleet.yaml
│       └── open-webui/         # Open WebUI chart configuration
│           └── fleet.yaml
```

## Prerequisites

1. Rancher with Fleet enabled
2. Kubernetes cluster registered with Rancher
3. OCI registry token for dp.apps.rancher.io
4. NVIDIA GPUs in your cluster for Ollama

## Setup Instructions

### 1. Create Registry Secret

Before deploying via Fleet, you need to set up the registry authentication token by replacing the placeholder in the `suse-ai-setup/registry-secret.yaml` file:

```yaml
stringData:
  .dockerconfigjson: |-
    {
      "auths": {
        "dp.apps.rancher.io": {
          "auth": "${REGISTRY_TOKEN}"  # Replace this with your token
        }
      }
    }
```

### 2. Add Repository to Fleet

1. In the Rancher UI, navigate to "Continuous Delivery" → "Git Repos"
2. Click "Add Repository"
3. Fill in the form:
   - Name: `suse-ai-fleet`
   - Repository URL: Your GitHub repository URL
   - Branch: `main` (or your default branch)
   - Path: `fleet-resources`
   - Cluster Selector: Match labels that correspond to your target cluster
   - For authentication, select the appropriate method for your GitHub repo

### 3. Configure Cluster Labels

Ensure your target cluster has the label `environment: production` to match the Fleet configuration, or modify the `fleet.yaml` file to match your existing labels.

### 4. Monitor Deployment

1. Navigate to "Continuous Delivery" → "Git Repos"
2. Click on your newly added repository
3. Monitor the deployment status of each bundle
4. Note that the bundles will deploy in the proper order:
   - First, the GPU operator direct deployment
   - Then, the suse-ai namespace and secret setup
   - Finally, the SUSE AI applications (Ollama, Milvus, Open WebUI)

## Deployment Order

The deployment is structured to respect these dependencies:

1. `gpu-operator-direct` - Deploys the NVIDIA GPU Operator directly with HelmChart resource
2. `suse-ai-namespace-setup` - Creates the suse-ai namespace and registry secret
3. `suse-ai-deployment` - Deploys all SUSE AI applications (depends on suse-ai namespace setup)

This order ensures all prerequisites are met before deploying applications.

## Troubleshooting

- If charts fail to deploy due to authentication issues, verify that the `application-collection` secret is correctly configured
- For GPU-related issues with Ollama, ensure your nodes have proper NVIDIA drivers installed and the GPU is accessible to the container runtime
- For GPU Operator issues, check the logs in the gpu-operator namespace
- Check Fleet logs for any errors during chart deployment
- To verify GPU operator installation, look for the test pod in the gpu-operator namespace:
  ```
  kubectl -n gpu-operator get pods nvidia-gpu-test
  kubectl -n gpu-operator logs nvidia-gpu-test
  ```

## Notes about GPU Configuration

The GPU Operator is configured with the following settings for RKE2:

- Driver installation is disabled (assuming pre-installed drivers)
- Toolkit environment variables are set for RKE2 containerd configuration
- The nvidia runtime is set as the default for containerd
- Version is pinned to 24.6.0 which has been tested and found to be stable 