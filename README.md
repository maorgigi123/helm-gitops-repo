# helm-gitops-repo

A GitOps repository for deploying **myapp** (a simple web microservice) to Kubernetes using Helm and ArgoCD.

## Project Structure

```
helm-gitops-repo/
├── argocd/
│   ├── dev-app.yaml          # ArgoCD Application for dev environment
│   └── prod-app.yaml         # ArgoCD Application for prod environment
└── charts/myapp/
    ├── Chart.yaml             # Helm chart metadata
    ├── values.yaml            # Default values
    ├── values-dev.yaml        # Dev environment overrides
    ├── values-prod.yaml       # Prod environment overrides
    └── templates/
        ├── deployment.yaml    # App Deployment
        ├── service.yaml       # ClusterIP Service
        ├── ingress.yaml       # Ingress (enabled per environment)
        ├── configmap.yaml     # Environment-specific config
        └── daemonset.yaml     # DaemonSet for node-level workloads
```

## How It Works

1. You push changes to the `main` branch on GitHub
2. ArgoCD watches this repo and automatically syncs changes to the cluster
3. Each environment (dev / prod) has its own ArgoCD Application, namespace, and values file

| Environment | Namespace | Values Files                    | Replicas | Ingress Host       |
|-------------|-----------|----------------------------------|----------|--------------------|
| dev         | `dev`     | `values.yaml` + `values-dev.yaml`  | 1        | `myapp.local`      |
| prod        | `prod`    | `values.yaml` + `values-prod.yaml` | 3        | `myapp.prod.local` |

## Prerequisites

- [minikube](https://minikube.sigs.k8s.io/docs/start/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [Helm](https://helm.sh/docs/intro/install/)
- [ArgoCD CLI](https://argo-cd.readthedocs.io/en/stable/cli_installation/) (optional)

## Setup

### 1. Start minikube

```bash
minikube start
```

### 2. Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait for ArgoCD to be ready:

```bash
kubectl wait --for=condition=available deployment/argocd-server -n argocd --timeout=120s
```

### 3. Register the ArgoCD Applications

```bash
kubectl apply -f argocd/dev-app.yaml
kubectl apply -f argocd/prod-app.yaml
```

ArgoCD will now pull the Helm chart from this repo and deploy to the `dev` and `prod` namespaces.

### 4. Enable Ingress (minikube)

```bash
minikube addons enable ingress
```

Add the following to `/etc/hosts` (replace the IP with your `minikube ip` output):

```bash
# Get your minikube IP
minikube ip

# Add to /etc/hosts (requires sudo)
echo '<MINIKUBE_IP>  myapp.local myapp.prod.local' | sudo tee -a /etc/hosts
```

### 5. Access the App

Once ArgoCD syncs and the ingress controller is running:

```bash
# Dev
curl http://myapp.local

# Prod
curl http://myapp.prod.local
```

Expected response:

```json
{"status": "ok", "service": "myapp"}
```

## Useful Commands

```bash
# Check ArgoCD app status
kubectl get applications -n argocd

# Check deployments
kubectl get all -n dev
kubectl get all -n prod

# Check ingress
kubectl get ingress -n dev
kubectl get ingress -n prod

# ArgoCD UI (port-forward)
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Then open https://localhost:8080

# Get ArgoCD admin password
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d
```

## Deploying Changes

This repo follows a GitOps workflow. To deploy changes:

1. Edit the Helm values or templates
2. Commit and push to `main`
3. ArgoCD automatically detects the change and syncs the cluster

Image tags are updated automatically by CI when new images are built.
