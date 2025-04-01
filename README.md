# SUSE AI Fleet Repository

This repository contains Kubernetes manifests for deploying the SUSE AI stack using Rancher Fleet. The deployment includes:

- NVIDIA GPU Operator
- Cert-Manager (for TLS certificate management)
- Ollama
- Milvus
- Open WebUI

## Repository Structure

The repository follows the standard Fleet pattern for multiple Helm charts:

```
fleet-resources/
├── fleet.yaml                # Main fleet configuration with ordering
├── charts/                   # All Helm charts and resources go here
│   ├── gpu-operator/         # GPU Operator chart
│   │   ├── fleet.yaml        # GPU-specific fleet configuration
│   │   └── values.yaml       # GPU operator values
│   ├── cert-manager/         # Cert-Manager for TLS certificates
│   │   └── fleet.yaml        # Cert-Manager fleet configuration
│   ├── ollama/
│   │   ├── fleet.yaml
│   │   └── values.yaml
│   ├── milvus/
│   │   ├── fleet.yaml
│   │   └── values.yaml
│   ├── open-webui/
│   │   ├── fleet.yaml
│   │   └── values.yaml
│   └── registry-secret.yaml  # Registry secret for OCI
```

## Prerequisites

1. Rancher with Fleet enabled
2. Kubernetes cluster registered with Rancher
3. OCI registry token for dp.apps.rancher.io
4. NVIDIA GPUs in your cluster for Ollama
5. Before deploying, you must manually create the `suse-ai` namespace and setup the registry secret:

   ```bash
   # Create the namespace
   kubectl create ns suse-ai
   
   # Create the registry secret in the suse-ai namespace
   kubectl create secret docker-registry application-collection \
     --docker-server=dp.apps.rancher.io \
     --docker-username=APPCO_USERNAME \
     --docker-password=APPCO_USER_TOKEN \
     -n suse-ai
   ```
   Replace `APPCO_USERNAME` and `APPCO_USER_TOKEN` with your actual credentials.

## Setup Instructions

### 1. Create Registry Secret

Before deploying via Fleet, you need to set up the registry authentication token by replacing the placeholder in the `charts/registry-secret.yaml` file:

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
   - First, the GPU operator will be deployed
   - Then, the registry secret will be created
   - Finally, the SUSE AI applications (Ollama, Milvus, Open WebUI) will be deployed

## Deployment Order

The deployment is structured to respect these dependencies:

1. `gpu-operator` - Deploys the NVIDIA GPU Operator 
2. `cert-manager` - Deploys Cert-Manager for TLS certificate management
3. `registry-secret` - Creates the registry secret for pulling images
4. `suse-ai-apps` - Deploys all SUSE AI applications (depends on GPU operator, cert-manager, and registry secret)

This order ensures all prerequisites are met before deploying applications.

## Notes on Namespace Handling

This repository takes advantage of Fleet's automatic namespace creation feature. As described in the [Fleet documentation](https://fleet.rancher.io/namespaces):

> "When deploying a Fleet bundle, the specified namespace will automatically be created if it does not already exist."

Each chart specifies its target namespace in its own fleet.yaml file:
- GPU Operator deploys to the `gpu-operator` namespace
- All SUSE AI applications deploy to the `suse-ai` namespace

## Troubleshooting

- If charts fail to deploy due to authentication issues, verify that the `application-collection` secret is correctly configured
- For GPU-related issues with Ollama, ensure your nodes have proper NVIDIA drivers installed and the GPU is accessible to the container runtime
- For GPU Operator issues, check the logs in the gpu-operator namespace
- Check Fleet logs for any errors during chart deployment
- If custom values aren't being applied to Helm charts, ensure each chart's `fleet.yaml` includes the `valuesFiles` section pointing to the corresponding `values.yaml` file:
  ```yaml
  helm:
    # other configuration...
    valuesFiles:
      - values.yaml
  ```

## Notes about GPU Configuration

The GPU Operator is configured with the following settings for RKE2:

- Driver installation is disabled (assuming pre-installed drivers)
- Toolkit environment variables are set for RKE2 containerd configuration
- The nvidia runtime is set as the default for containerd
- Version is pinned to 24.6.0 which has been tested and found to be stable