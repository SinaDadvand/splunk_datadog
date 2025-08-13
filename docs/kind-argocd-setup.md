# Local Kubernetes + ArgoCD on Windows (Docker Desktop + Kind)

This guide sets up a local Kubernetes cluster with Kind, then installs ArgoCD for GitOps. Commands are for Windows PowerShell 5.1+.

## Prerequisites
- Windows 10/11 with admin rights
- Docker Desktop (Linux containers)
- PowerShell 5.1+ (or PowerShell 7)

## Install tooling (with winget)
Run PowerShell as Administrator:

```
winget install -e --id Docker.DockerDesktop
winget install -e --id Kubernetes.kubectl
winget install -e --id Kubernetes.Kind
winget install -e --id Helm.Helm
winget install -e --id Git.Git
```

Notes:
- After installing Docker Desktop, start it and keep it running.
- Ensure Docker is set to “Use the WSL 2 based engine” and Linux containers.

## Create a Kind cluster
```
kind create cluster --name dev
kubectl cluster-info
kubectl get nodes -o wide
```

Delete later with:
```
kind delete cluster --name dev
```

## Install ArgoCD
```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd rollout status deploy/argocd-server
```

Port-forward the UI:
```
kubectl -n argocd port-forward svc/argocd-server 8080:443
```
Open http://localhost:8080.

Get initial admin password (PowerShell):
```
$pwdB64 = kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}'
[Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($pwdB64))
```
Username: admin

Secure ArgoCD after first login:
- Change the admin password (User Info → Change Password).
- Add your Git repository via Settings → Repositories.

You’re ready to deploy apps with ArgoCD. Continue with `splunk-lab.md` and `datadog-lab.md`.
