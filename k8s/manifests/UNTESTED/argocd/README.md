# ArgoCD Installation Guide

This guide provides instructions for installing ArgoCD in a Kubernetes cluster using Helm.

## Prerequisites

- A running Kubernetes cluster
- kubectl command-line tool installed and configured
- Helm installed (version 3+)

## Installation Steps

### 1. Add the ArgoCD Helm repository

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
```

### 2. Create a namespace for ArgoCD

```bash
kubectl create namespace argocd
```

### 3. Install ArgoCD using Helm

```bash
helm install argocd argo/argo-cd --namespace argocd
```

### 4. Access the ArgoCD UI

By default, the ArgoCD API server is not exposed with an external IP. To access the UI, you can use port-forwarding:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Then visit: https://localhost:8080

### 5. Get the initial admin password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Use these credentials to log in:
- Username: admin
- Password: ---

**Important**: After logging in, you should change the default password and delete the initial secret as recommended by ArgoCD's documentation.

To delete the initial admin secret after changing the password:
```bash
kubectl -n argocd delete secret argocd-initial-admin-secret
```

## Uninstallation

To uninstall ArgoCD:

```bash
helm uninstall argocd -n argocd
kubectl delete namespace argocd
``` 