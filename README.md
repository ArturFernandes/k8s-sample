# Demo Project: GitOps with Kind, Argo CD, and Sealed Secrets

This project demonstrates deploying a web application on a local Kubernetes cluster (kind) using a GitOps approach. Argo CD is used to synchronize Kubernetes manifests from a Git repository, and Sealed Secrets is used to manage secrets securely.

## 🏛️ Architecture
The environment consists of:

- **Local Kubernetes Cluster**: A kind cluster used to simulate a production Kubernetes environment.
- **Web Application**: A simple Python application called "it-depends".
- **Argo CD**: Acts as the GitOps operator, monitoring the Git repository for manifest changes and applying them automatically to the cluster.
- **NGINX Ingress Controller**: Manages external traffic, routing requests to the correct services within the cluster.
- **Sealed Secrets**: Enables storing secrets (like TLS certificates) in an encrypted form safely in a public Git repository.

## 📹 Video Example:

https://github.com/user-attachments/assets/b8379721-2c56-41de-acd2-f5c084794a70

> **Note**: The video will automatically play when viewing this README on GitHub. If you're viewing this documentation elsewhere, you can find the video in the `docs` folder.

This video covers:
- Step-by-step cluster setup
- Argo CD installation and configuration
- Deploying the application
- Managing secrets with Sealed Secrets

## ✅ Prerequisites
Before starting, ensure you have the following tools installed on your machine:

- Docker: To run kind containers
- Kind: To create the local Kubernetes cluster
- kubectl: The command-line tool for interacting with Kubernetes clusters
- kubeseal: The CLI for encrypting secrets and creating SealedSecrets

## 🚀 Local Installation Guide
Follow these steps to deploy the environment on your machine.

### 1. Create Kubernetes Cluster with Kind
Use the provided configuration file to create the cluster. This file maps ports 80 and 443 from your host to the kind container, enabling Ingress Controller access.

```bash
kind create cluster --config=kind-config.yaml
```

### 2. Install NGINX Ingress Controller
The Ingress Controller is required for the Ingress resource to work and direct traffic to our application.

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

# Wait for the Ingress Controller pods to be ready
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s
```

### 3. Install Argo CD
Create the Argo CD namespace and install it using the official manifest.

```bash
# Create namespace
kubectl create namespace argocd

# Install Argo CD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 4. Install Sealed Secrets Controller
This controller is responsible for decrypting SealedSecrets within the cluster and converting them into regular Kubernetes Secrets.

```bash
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.20.0/controller.yaml
```

### 5. Apply the Argo CD Root Application (App of Apps)
This step initiates the GitOps magic by telling Argo CD to start monitoring your Git repository.

```bash
kubectl apply -f argocd/root-app.yaml
```

At this point, Argo CD will:
- Detect the root application
- Read the specified repository and path (apps/it-depends)
- Find the it-depends application definition
- Start synchronizing all manifests located in apps/manifests

### 6. Configure Local DNS
To access the application using the it-depends.local domain, add an entry to your hosts file:

- Linux/macOS: Edit `/etc/hosts`
- Windows: Edit `C:\Windows\System32\drivers\etc\hosts` (as administrator)

Add the following line:
```
127.0.0.1 it-depends.local
```

### 7. Access the Application! 🎉
Now that everything is configured, you can access the application. Open your browser and visit:

https://it-depends.local

Since we're using a self-signed certificate for TLS, your browser will display a security warning. You can safely proceed to access the application.

## 🔧 Managing Secrets
The TLS secret for it-depends.local is stored in encrypted form in `apps/manifests/sealed-tls-secret.yaml`. To generate a new certificate or modify this secret:

1. Generate new tls.crt and tls.key files (if needed)

2. Create a temporary Kubernetes Secret in YAML format:
```bash
kubectl create secret tls it-depends-tls-secret \
    --cert=tls.crt \
    --key=tls.key \
    --namespace=it-depends \
    --dry-run=client -o yaml > it-depends-tls-secret.yaml
```

3. Seal the secret using kubeseal:
```bash
kubeseal --format=yaml < it-depends-tls-secret.yaml > apps/manifests/sealed-tls-secret.yaml
```

4. Commit and push the change to your Git repository. Argo CD will detect the change and automatically update the secret in the cluster.
