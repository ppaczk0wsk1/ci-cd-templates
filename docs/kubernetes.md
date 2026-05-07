# Deploying to Kubernetes

[< Back to README](../README.md) | [SSH/VPS deploys](deploying.md)

---

## Overview

Deploying to Kubernetes from GitHub Actions follows a pattern: build a container image, push it to a registry, authenticate to the cluster, and apply manifests. This guide covers each step for managed clusters (EKS, GKE, AKS), self-hosted clusters, and Helm deployments.

## The flow

```
push to main
  |
  +- build Docker image
  +- push to container registry (GHCR, ECR, GCR, ACR)
  +- authenticate to Kubernetes cluster
  +- apply manifests (kubectl apply or helm upgrade)
  +- verify rollout succeeded
  +- rollback automatically if it didn't
```

## Prerequisites

Your workflow templates already have a Docker build+push job. This guide assumes you've uncommented it and have images landing in a registry. If not, start with the Docker section in your workflow template first.

## 1. Self-hosted cluster (kubeconfig)

This is the simplest approach. You export your kubeconfig, store it as a secret, and use it in CI.

### Get your kubeconfig

```bash
# On your local machine (where you already have kubectl access)
# Base64 encode the kubeconfig so it survives being stored as a secret
cat ~/.kube/config | base64 | tr -d '\n'
```

Copy the output and add it as a GitHub secret named `KUBECONFIG`.

**Important:** if your kubeconfig references `127.0.0.1` or `localhost`, that won't work from the GitHub runner. You need the cluster's public IP or DNS name in the kubeconfig's `server` field.

### Add the deploy job to your workflow

```yaml
deploy:
  runs-on: ubuntu-latest
  needs: [ci, docker]
  if: github.event_name == 'push' && github.ref == 'refs/heads/main'
  environment: production

  steps:
    - uses: actions/checkout@v6

    # Install a specific kubectl version for reproducibility.
    - name: Set up kubectl
      uses: azure/setup-kubectl@v4
      with:
        version: "v1.31.0"

    # Write the kubeconfig file from the secret.
    # base64 -d decodes the base64-encoded secret back to YAML.
    # chmod 600 restricts access to the current user only.
    - name: Configure kubectl
      run: |
        mkdir -p $HOME/.kube
        echo "${{ secrets.KUBECONFIG }}" | base64 -d > $HOME/.kube/config
        chmod 600 $HOME/.kube/config

    # Apply manifests and wait for the rollout to finish.
    # kubectl rollout status blocks until all pods are ready
    # or the timeout is reached.
    - name: Deploy
      run: |
        kubectl apply -f k8s/
        kubectl rollout status deployment/myapp -n production --timeout=300s

    # If the rollout failed, undo it.
    - name: Rollback on failure
      if: failure()
      run: kubectl rollout undo deployment/myapp -n production
```

### Setting up the k8s/ manifests directory

Your repo should have a `k8s/` directory with at least a Deployment and a Service:

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          # Use a placeholder tag here. We'll update it in CI
          # using kustomize or sed before applying.
          image: ghcr.io/youruser/yourrepo:latest
          ports:
            - containerPort: 3000
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "500m"
```

### Updating the image tag per deploy

You don't want to deploy `latest` every time. Use `sed` or `kustomize` to inject the commit SHA:

```yaml
- name: Set image tag
  run: |
    sed -i "s|ghcr.io/youruser/yourrepo:latest|ghcr.io/youruser/yourrepo:${{ github.sha }}|g" k8s/deployment.yaml

- name: Deploy
  run: kubectl apply -f k8s/
```

Or with `kubectl set image` (no file editing needed):

```yaml
- name: Deploy
  run: |
    kubectl set image deployment/myapp \
      myapp=ghcr.io/youruser/yourrepo:${{ github.sha }} \
      -n production
    kubectl rollout status deployment/myapp -n production --timeout=300s
```

## 2. AWS EKS

EKS uses IAM for authentication. The best approach is OIDC (no long-lived keys).

### One-time setup

1. Create an IAM OIDC identity provider for GitHub Actions in your AWS account
2. Create an IAM role with EKS access, trusted by the GitHub OIDC provider
3. See [AWS docs on GitHub OIDC](https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services)

### Workflow

```yaml
deploy:
  runs-on: ubuntu-latest
  needs: [ci, docker]
  if: github.event_name == 'push' && github.ref == 'refs/heads/main'
  environment: production

  # OIDC requires these permissions.
  permissions:
    id-token: write
    contents: read

  steps:
    - uses: actions/checkout@v6

    # Authenticate to AWS using OIDC (no access keys stored in secrets).
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsEKS
        aws-region: us-east-1

    # Generate a kubeconfig pointing at your EKS cluster.
    - name: Update kubeconfig
      run: |
        aws eks update-kubeconfig \
          --region us-east-1 \
          --name my-cluster

    - name: Deploy
      run: |
        kubectl set image deployment/myapp \
          myapp=123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:${{ github.sha }} \
          -n production
        kubectl rollout status deployment/myapp -n production --timeout=300s

    - name: Rollback on failure
      if: failure()
      run: kubectl rollout undo deployment/myapp -n production
```

## 3. Google GKE

GKE also supports OIDC via Workload Identity Federation.

### One-time setup

1. Create a Workload Identity Pool and Provider in your GCP project
2. Create a service account with GKE access
3. See [GCP Workload Identity Federation docs](https://cloud.google.com/iam/docs/workload-identity-federation-with-deployment-pipelines)

### Workflow

```yaml
deploy:
  runs-on: ubuntu-latest
  needs: [ci, docker]
  if: github.event_name == 'push' && github.ref == 'refs/heads/main'
  environment: production

  permissions:
    id-token: write
    contents: read

  steps:
    - uses: actions/checkout@v6

    # Authenticate to GCP using Workload Identity Federation.
    - name: Authenticate to Google Cloud
      uses: google-github-actions/auth@v2
      with:
        workload_identity_provider: projects/123456/locations/global/workloadIdentityPools/github/providers/github
        service_account: deploy@my-project.iam.gserviceaccount.com

    - name: Set up gcloud
      uses: google-github-actions/setup-gcloud@v2

    # Fetch cluster credentials into kubeconfig.
    - name: Get GKE credentials
      uses: google-github-actions/get-gke-credentials@v2
      with:
        cluster_name: my-cluster
        location: us-central1

    - name: Deploy
      run: |
        kubectl set image deployment/myapp \
          myapp=gcr.io/my-project/myapp:${{ github.sha }} \
          -n production
        kubectl rollout status deployment/myapp -n production --timeout=300s

    - name: Rollback on failure
      if: failure()
      run: kubectl rollout undo deployment/myapp -n production
```

## 4. Azure AKS

```yaml
deploy:
  runs-on: ubuntu-latest
  needs: [ci, docker]
  if: github.event_name == 'push' && github.ref == 'refs/heads/main'
  environment: production

  permissions:
    id-token: write
    contents: read

  steps:
    - uses: actions/checkout@v6

    # Authenticate to Azure using OIDC.
    - name: Azure login
      uses: azure/login@v2
      with:
        client-id: ${{ secrets.AZURE_CLIENT_ID }}
        tenant-id: ${{ secrets.AZURE_TENANT_ID }}
        subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

    # Set the AKS cluster context.
    - name: Set AKS context
      uses: azure/aks-set-context@v4
      with:
        resource-group: my-resource-group
        cluster-name: my-cluster

    - name: Deploy
      run: |
        kubectl set image deployment/myapp \
          myapp=myregistry.azurecr.io/myapp:${{ github.sha }} \
          -n production
        kubectl rollout status deployment/myapp -n production --timeout=300s

    - name: Rollback on failure
      if: failure()
      run: kubectl rollout undo deployment/myapp -n production
```

## 5. Helm deployments

Helm is the standard package manager for Kubernetes. Use it when your app has lots of configurable values, multiple environments, or you want templated manifests.

```yaml
deploy:
  runs-on: ubuntu-latest
  needs: [ci, docker]
  if: github.event_name == 'push' && github.ref == 'refs/heads/main'
  environment: production

  steps:
    - uses: actions/checkout@v6

    # Install Helm.
    - name: Set up Helm
      uses: azure/setup-helm@v4
      with:
        version: "v3.16.0"

    - name: Set up kubectl
      uses: azure/setup-kubectl@v4

    - name: Configure kubectl
      run: |
        mkdir -p $HOME/.kube
        echo "${{ secrets.KUBECONFIG }}" | base64 -d > $HOME/.kube/config
        chmod 600 $HOME/.kube/config

    # helm upgrade --install: installs if the release doesn't exist,
    #   upgrades if it does.
    # --set image.tag: overrides the image tag in values.yaml with
    #   the commit SHA, so each deploy uses the exact image we just built.
    # --wait: blocks until all pods are ready.
    # --timeout: fails the step if pods aren't ready in time.
    # --atomic: automatically rolls back if the deploy fails.
    - name: Deploy with Helm
      run: |
        helm upgrade --install myapp ./helm/myapp \
          --namespace production \
          --set image.repository=ghcr.io/${{ github.repository }} \
          --set image.tag=${{ github.sha }} \
          --wait \
          --timeout 5m \
          --atomic

    # Verify what got deployed.
    - name: Verify
      run: |
        kubectl get pods -n production -l app.kubernetes.io/name=myapp
        kubectl rollout status deployment/myapp -n production
```

The `--atomic` flag is key. If anything goes wrong during the Helm upgrade, it automatically rolls back to the previous release. No need for a separate rollback step.

## Security tips

- **Prefer OIDC over long-lived credentials.** EKS, GKE, and AKS all support it. Tokens are short-lived and scoped to your repo.
- **Create a dedicated service account** in your cluster with only the permissions it needs (deploy to one namespace, not cluster-admin).
- **Use namespaces** to isolate environments (staging, production).
- **Use environment protection rules** in GitHub. Require manual approval for production deploys at Settings -> Environments.
- **Pin kubectl and Helm versions** for reproducibility. A surprise version bump shouldn't break your deploys.
- **Don't store kubeconfig with cluster-admin access.** Scope it to the deploy service account.
- **Set resource requests and limits** in your Deployment manifests so a bad deploy can't consume the whole cluster.

## Example RBAC for the deploy service account

Create a limited service account that can only manage deployments in one namespace:

```yaml
# k8s/rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: github-actions
  namespace: production
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployer
  namespace: production
rules:
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets"]
    verbs: ["get", "list", "watch", "update", "patch"]
  - apiGroups: [""]
    resources: ["pods", "services"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: github-actions-deployer
  namespace: production
subjects:
  - kind: ServiceAccount
    name: github-actions
    namespace: production
roleRef:
  kind: Role
  name: deployer
  apiGroup: rbac.authorization.k8s.io
```

Apply it: `kubectl apply -f k8s/rbac.yaml`, then create a kubeconfig for this service account instead of using your personal one.

## Secrets reference

| Secret | Used for | Where to get it |
|---|---|---|
| `KUBECONFIG` | Self-hosted clusters | `cat ~/.kube/config | base64 | tr -d '\n'` |
| AWS OIDC role ARN | EKS | IAM console |
| GCP Workload Identity | GKE | GCP console |
| `AZURE_CLIENT_ID` | AKS | Azure AD app registration |
| `AZURE_TENANT_ID` | AKS | Azure AD |
| `AZURE_SUBSCRIPTION_ID` | AKS | Azure portal |
