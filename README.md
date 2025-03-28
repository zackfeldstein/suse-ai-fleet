# SUSE AI Fleet Repository

This repository contains Kubernetes manifests for deploying the SUSE AI stack using Rancher Fleet. The deployment includes:

- Ollama
- Milvus
- Open WebUI

## Repository Structure

```
fleet-resources/
├── fleet.yaml                # Main Fleet configuration
├── suse-ai/
│   ├── namespace.yaml        # Namespace definition
│   ├── registry-secret.yaml  # Registry authentication template
│   └── charts/
│       ├── ollama/           # Ollama chart configuration
│       │   └── fleet.yaml
│       ├── milvus/           # Milvus chart configuration
│       │   └── fleet.yaml
│       └── open-webui/       # Open WebUI chart configuration
│           └── fleet.yaml
```

## Prerequisites

1. Rancher with Fleet enabled
2. Kubernetes cluster registered with Rancher
3. OCI registry token for dp.apps.rancher.io

## Setup Instructions

### 1. Create Registry Secret

Before deploying via Fleet, you need to set up the registry authentication token:

1. Obtain your registry token for dp.apps.rancher.io
2. In the Rancher UI, go to your cluster → Resources → Secrets
3. Create a new secret in the `suse-ai` namespace (create the namespace first if needed):
   - Name: `application-collection`
   - Type: `docker-registry`
   - Registry: `dp.apps.rancher.io`
   - Username: (leave empty if using token authentication)
   - Password: Your registry token

Alternatively, you can apply the registry-secret.yaml after setting the `REGISTRY_TOKEN` placeholder:

```bash
# Replace the placeholder with your base64-encoded token
REGISTRY_TOKEN="your-token-here"
sed -i "s/\${REGISTRY_TOKEN}/$REGISTRY_TOKEN/" fleet-resources/suse-ai/registry-secret.yaml
kubectl apply -f fleet-resources/suse-ai/registry-secret.yaml
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

## Troubleshooting

- If charts fail to deploy due to authentication issues, verify that the `application-collection` secret is correctly configured
- For GPU-related issues with Ollama, ensure your nodes have proper NVIDIA drivers installed and the GPU is accessible to the container runtime
- Check Fleet logs for any errors during chart deployment

## Customization

To customize any of the Helm chart values, modify the corresponding `fleet.yaml` file in the appropriate chart directory. 