# Kubernetes GitOps Platform
Ce repository contient l'infrastructure Kubernetes gérée en **GitOps via Argo CD (IAC)**.

## Install Kubernetes
```sh
curl -sfL https://get.k3s.io | sh -
```

## Install ArgoCD
```sh
kubectl create namespace argocd
kubectl apply --server-side -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl get pods -n argocd -w
# Tous les pods status en "1/1 Running"

# Activer le mode insecure (ArgoCD derrière un reverse proxy TLS)
kubectl -n argocd patch cm argocd-cmd-params-cm -p '{"data":{"server.insecure":"true"}}'
kubectl -n argocd rollout restart deployment argocd-server

# Recuperation du MDP compte "admin" (il sera supprimé par la suite)
#printf "%s\n" "$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d)"
```

### Ajouter la clef privee SOPS (decode secret)
```sh
kubectl -n argocd create secret generic sops-age --from-literal=keys.txt="AGE-SECRET-KEY-1XXXX..."
```

### Bootstrap sync ArgosCD
```sh
kubectl apply -n argocd -f bootstrap/root-application.yaml
```
