# AKS CI/CD setup

The workflow at `.github/workflows/deploy-aks.yml` builds the Docker image, pushes it
to Azure Container Registry (ACR), and deploys it to AKS on every push to `main`.
It authenticates via OIDC federated credentials — no long-lived secrets stored in GitHub.

## 1. Create an Azure AD app registration + federated credential

```bash
# Variables — fill in your values
GH_ORG="tanmaymishra0607"
GH_REPO="GenAI_AKS_CICD"
APP_NAME="gh-genai-aks-cicd"
RESOURCE_GROUP="<your-resource-group>"
ACR_NAME="<your-acr-name>"
AKS_CLUSTER_NAME="<your-aks-cluster-name>"

# Create the app registration + service principal
APP_ID=$(az ad app create --display-name "$APP_NAME" --query appId -o tsv)
az ad sp create --id "$APP_ID"

# Federated credential scoped to pushes on main
az ad app federated-credential create --id "$APP_ID" --parameters '{
  "name": "github-main",
  "issuer": "https://token.actions.githubusercontent.com",
  "subject": "repo:'"$GH_ORG"'/'"$GH_REPO"':ref:refs/heads/main",
  "audiences": ["api://AzureADTokenExchange"]
}'

# Optional: also allow manual workflow_dispatch runs from main (same subject, so nothing extra needed)
```

## 2. Grant the app the permissions it needs

```bash
SUBSCRIPTION_ID=$(az account show --query id -o tsv)
ACR_ID=$(az acr show -n "$ACR_NAME" -g "$RESOURCE_GROUP" --query id -o tsv)
AKS_ID=$(az aks show -n "$AKS_CLUSTER_NAME" -g "$RESOURCE_GROUP" --query id -o tsv)

# Push images to ACR
az role assignment create --assignee "$APP_ID" --role AcrPush --scope "$ACR_ID"

# Fetch AKS credentials
az role assignment create --assignee "$APP_ID" --role "Azure Kubernetes Service Cluster User Role" --scope "$AKS_ID"

# If the cluster uses Azure RBAC for Kubernetes authorization, also grant a data-plane role
# (skip this if the cluster uses local Kubernetes RBAC / kubeconfig admin creds instead):
az role assignment create --assignee "$APP_ID" --role "Azure Kubernetes Service RBAC Writer" --scope "$AKS_ID"
```

## 3. Configure GitHub

```bash
gh secret set AZURE_CLIENT_ID --body "$APP_ID"
gh secret set AZURE_TENANT_ID --body "$(az account show --query tenantId -o tsv)"
gh secret set AZURE_SUBSCRIPTION_ID --body "$SUBSCRIPTION_ID"

gh variable set ACR_NAME --body "$ACR_NAME"
gh variable set AKS_CLUSTER_NAME --body "$AKS_CLUSTER_NAME"
gh variable set AKS_RESOURCE_GROUP --body "$RESOURCE_GROUP"
```

## 4. Deploy

Push to `main`, or run manually:

```bash
gh workflow run deploy-aks.yml
```

The workflow builds the image via `az acr build` (tagged with the short commit SHA and
`latest`), substitutes the image into `k8s/deployment.yaml`, applies both manifests to
the `default` namespace, and waits for the rollout to finish. The Service is
`type: LoadBalancer` — once it's up, get the external IP with:

```bash
kubectl get svc genai-aks-cicd -n default
```
