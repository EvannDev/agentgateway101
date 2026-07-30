# agentgateway101

## Bootstrap

```bash
# 1. Argo CD
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 2. Gateway API (CRDs standard, hors GitOps car pré-requis d'Argo CD lui-même)
kubectl apply --server-side \
  -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.6.1/standard-install.yaml

# 3. AppProject — doit exister AVANT l'ApplicationSet, sinon les Applications
#    générées restent en erreur "project platform does not exist".
kubectl apply -f project.yaml

# 4. ApplicationSet de bootstrap : découvre base/* et génère une Application
#    par composant. Tout le reste est géré en GitOps.
kubectl apply -f appset.yaml
```

## Structure

```
appset.yaml                        ApplicationSet (git directory generator sur base/*)
project.yaml                       AppProject "platform"
base/<composant>/*.yaml            Application CR du composant (app-of-apps)
```

Ajouter un composant = créer `base/<nom>/` avec une ou plusieurs `Application`.
L'ApplicationSet le détecte au prochain scan du repo (3 min par défaut, ou
immédiatement via webhook GitHub).

## Vérifications

```bash
kubectl get applicationset -n argocd
kubectl get applications -n argocd
argocd app list                    # si le CLI est configuré
```
