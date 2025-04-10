# SUSE AI Fleet Repository

This repository contains Kubernetes manifests for deploying the SUSE AI stack using Rancher Fleet. The deployment includes:

- NVIDIA GPU Operator
- Cert-Manager (for TLS certificate management)
- SUSE AI Components (Ollama, Milvus, Open WebUI)
- NeuVector (for container security)

## Repository Structure

The repository follows the standard Fleet pattern for multiple Helm charts:

```
suse-ai-minimal/
├── fleet.yaml                # Main fleet configuration with ordering
├── charts/                   # All Helm charts and resources go here
│   ├── gpu-operator-namespace/ # Creates the gpu-operator namespace
│   │   └── ... 
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
│   ├── neuvector-namespace/  # Creates the cattle-neuvector-system namespace
│   │   └── ... 
│   ├── neuvector-crd/        # NeuVector CRDs
│   │   └── fleet.yaml        # NeuVector CRD fleet configuration
│   ├── neuvector/            # NeuVector security platform
│   │   ├── fleet.yaml        # NeuVector fleet configuration
│   │   └── values.yaml       # NeuVector values
```

## Prerequisites

1. Rancher with Fleet enabled
2. Kubernetes cluster registered with Rancher
3. OCI registry token for dp.apps.rancher.io
4. NVIDIA GPUs in your cluster for Ollama
5. **IMPORTANT**: You MUST manually create the `suse-ai` namespace and the `application-collection` secret BEFORE deploying this repository with Fleet:

   ```bash
   # Step 1: Create the suse-ai namespace
   kubectl create ns suse-ai
   
   # Step 2: Create the application-collection secret in the suse-ai namespace
   # This secret is required for pulling images from the SUSE Application Catalog
   kubectl create secret docker-registry application-collection \
     --docker-server=dp.apps.rancher.io \
     --docker-username=APPCO_USERNAME \
     --docker-password=APPCO_USER_TOKEN \
     -n suse-ai
   ```
   Replace `APPCO_USERNAME` and `APPCO_USER_TOKEN` with your actual SUSE Application Collection credentials.

   The deployment will fail if this namespace and secret don't exist, as the Fleet repository assumes they are already created.

## Setup Instructions

### 1. Add Repository to Fleet

1. In the Rancher UI, navigate to "Continuous Delivery" → "Git Repos"
2. Click "Add Repository"
3. Fill in the form:
   - Name: `suse-ai-fleet`
   - Repository URL: Your GitHub repository URL
   - Branch: `main` (or your default branch)
   - Path: `suse-ai-minimal`
   - Cluster Selector: Match labels that correspond to your target cluster
   - For authentication, select the appropriate method for your GitHub repo

### 2. Configure Cluster Labels

Ensure your target cluster has the label `environment: production` to match the Fleet configuration, or modify the `fleet.yaml` file to match your existing labels.

### 3. Monitor Deployment

1. Navigate to "Continuous Delivery" → "Git Repos"
2. Click on your newly added repository
3. Monitor the deployment status of each bundle
4. Note that the bundles will deploy in the proper order:
   - First, the GPU operator namespace and the GPU operator will be deployed
   - Then, cert-manager will be deployed
   - Next, the SUSE AI applications (Ollama, Milvus, Open WebUI) will be deployed
   - Then, the NeuVector namespace will be created
   - Next, the NeuVector CRDs will be installed
   - Finally, the NeuVector services will be deployed

## Deployment Order

The deployment is structured to respect these dependencies:

1. `gpu-operator-namespace` - Creates the namespace for GPU Operator
2. `gpu-operator` - Deploys the NVIDIA GPU Operator 
3. `cert-manager` - Deploys Cert-Manager for TLS certificate management
4. `suse-ai-apps` - Deploys all SUSE AI applications (depends on GPU operator and cert-manager)
5. `neuvector-namespace` - Creates the namespace for NeuVector
6. `neuvector-crd` - Deploys NeuVector Custom Resource Definitions
7. `neuvector` - Deploys NeuVector container security platform

This order ensures all prerequisites are met before deploying applications.

## Notes on Namespace Handling

Each chart specifies its target namespace in its own fleet.yaml file:
- The GPU Operator namespace chart creates the `gpu-operator` namespace
- GPU Operator deploys to the `gpu-operator` namespace
- All SUSE AI applications deploy to the `suse-ai` namespace (must be created manually before deployment)
- The NeuVector namespace chart creates the `cattle-neuvector-system` namespace
- NeuVector CRDs and services deploy to the `cattle-neuvector-system` namespace

## NeuVector Configuration

The NeuVector deployment includes:
- NeuVector namespace creation (cattle-neuvector-system)
- NeuVector CRD installation (deployed as a separate chart)
- NeuVector Controller, Enforcer, and Manager components
- Integration with Rancher authentication
- Scanner for vulnerability detection

The default configuration includes:
- 3 controller replicas for high availability
- Rancher SSO authentication enabled
- Automatic certificate generation
- Vulnerability scanning enabled with CVE updates scheduled daily

### Configuring NeuVector Rancher URL

The Rancher URL used for NeuVector SSO authentication is configured in the `suse-ai-minimal/charts/neuvector/fleet.yaml` file. To change the Rancher URL, modify the following section:

```yaml
helm:
  # other configuration...
  values:
    global:
      cattle:
        url: https://ranch.kcdemos.com
```

Update the URL to match your Rancher installation. This approach allows you to easily change the URL without modifying the values.yaml file directly.

## Troubleshooting

- If charts fail to deploy due to authentication issues, verify that the `application-collection` secret is correctly configured in the `suse-ai` namespace
- If you see errors like `namespaces "suse-ai" not found`, ensure you've manually created the namespace before deploying
- For GPU-related issues with Ollama, ensure your nodes have proper NVIDIA drivers installed and the GPU is accessible to the container runtime
- For GPU Operator issues, check the logs in the gpu-operator namespace
- For NeuVector issues, check the logs in the cattle-neuvector-system namespace
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