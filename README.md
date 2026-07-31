# agentgateway101

Démo GitOps : Argo CD déploie agentgateway, qui sert de passerelle LLM (Anthropic,
Gemini) et MCP pour Open WebUI.

> **Ce README est le registre des actions manuelles.** Tout ce qui est décrit dans
> `base/` et `apps/` est réconcilié automatiquement par Argo CD. Tout ce qui figure
> ci-dessous ne l'est **pas** : c'est l'écart entre le contenu de Git et un cluster
> qui fonctionne. Toute nouvelle étape manuelle doit être ajoutée ici.

---

## 1. Bootstrap

À exécuter dans l'ordre, sur un cluster vide.

```bash
# 1.1 Argo CD
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 1.2 Gateway API (canal standard)
#     Hors GitOps : c'est un prérequis du contrôleur agentgateway, qui doit
#     exister avant qu'Argo CD ne tente de créer Gateway et HTTPRoute.
kubectl apply --server-side \
  -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.6.1/standard-install.yaml

# 1.3 AppProject — DOIT exister avant les ApplicationSets, sinon les Applications
#     générées restent en erreur "project platform does not exist".
kubectl apply -f project.yaml

# 1.4 Les deux ApplicationSets (bootstrap sur base/*, apps sur apps/*).
#     À partir d'ici, tout le reste est géré en GitOps.
kubectl apply -f appset.yaml
```

---

## 2. Credentials (hors GitOps par conception)

Les clés d'API ne sont **jamais** committées. Le `.gitignore` exclut
`base/agentgateway-configuration/*-apikey.yaml` ; seuls les fichiers `.example`
sont suivis. Il faut donc créer les Secrets à la main après le bootstrap.

| Secret | Clé | Consommé par |
|---|---|---|
| `anthropic-secret` | `Authorization` | `AgentgatewayBackend/anthropic` (`policies.auth.secretRef`) |
| `google-secret` | `Authorization` | `AgentgatewayBackend/google` |
| `argocd-mcp-secret` | `token` | `Deployment/argocd-mcp` (env `ARGOCD_API_TOKEN`) |
| `mcp-inspector-secret` | `token` | `Deployment/mcp-inspector` (env `MCP_INSPECTOR_API_TOKEN`) |

### 2.1 Clés des fournisseurs LLM

```bash
cd base/agentgateway-configuration

export ANTHROPIC_API_KEY='sk-ant-...'
envsubst < anthropic-apikey.yaml.example > anthropic-apikey.yaml
kubectl apply -f anthropic-apikey.yaml

export GOOGLE_KEY='...'
envsubst < gemini-apikey.yaml.example > gemini-apikey.yaml
kubectl apply -f gemini-apikey.yaml
```

`envsubst` vient avec `gettext` (`brew install gettext`). Sans lui, copier le
`.example` et remplacer la variable à la main.

### 2.2 Token Argo CD pour le serveur MCP

Le serveur `argocd-mcp` s'authentifie auprès de l'API Argo CD avec un token de
compte local. Sa génération n'est pas automatisable en GitOps.

```bash
# a) Déclarer un compte local avec capacité apiKey
kubectl patch cm argocd-cm -n argocd --type merge \
  -p '{"data":{"accounts.mcp":"apiKey"}}'

# b) Lui donner des droits en lecture seule.
#    ATTENTION : ce patch écrase policy.csv. Récupérer la valeur existante avant
#    si le fichier n'est pas vide :
#      kubectl get cm argocd-rbac-cm -n argocd -o jsonpath='{.data.policy\.csv}'
kubectl patch cm argocd-rbac-cm -n argocd --type merge \
  -p '{"data":{"policy.csv":"g, mcp, role:readonly"}}'

# c) Générer le token (nécessite le CLI argocd connecté)
argocd account generate-token --account mcp

# d) Créer le Secret
export ARGOCD_API_TOKEN='<le-token-généré>'
envsubst < base/agentgateway-configuration/argocd-mcp-apikey.yaml.example \
  > base/agentgateway-configuration/argocd-mcp-apikey.yaml
kubectl apply -f base/agentgateway-configuration/argocd-mcp-apikey.yaml
```

`role:readonly` est cohérent avec `MCP_READ_ONLY: "true"` dans le Deployment : les
deux couches expriment la même politique. Pour autoriser `sync_application` plus
tard, il faudra élargir **les deux** — le RBAC du compte Argo CD et la variable
d'environnement du conteneur.

### 2.3 Token de MCP Inspector

L'Inspector génère un token au démarrage et l'écrit dans ses logs. On lui en
impose un fixe pour qu'il survive aux redémarrages du pod.

```bash
export MCP_INSPECTOR_API_TOKEN=$(openssl rand -hex 32)
echo "Token : $MCP_INSPECTOR_API_TOKEN"    # à conserver, il sert dans l'URL

envsubst < base/agentgateway-configuration/mcp-inspector-apikey.yaml.example \
  > base/agentgateway-configuration/mcp-inspector-apikey.yaml
kubectl apply -f base/agentgateway-configuration/mcp-inspector-apikey.yaml
```

---

## 3. Actions manuelles optionnelles

### 3.1 Ordonnancement réel des sync-waves (recommandé)

Les `Application` enfants n'ont **aucun health check** depuis Argo CD 1.8, donc
elles sont considérées `Healthy` dès leur création. Conséquence concrète : les
ressources de wave 0 de `base/agentgateway-configuration` (dont les
`AgentgatewayBackend`, qui sont des CR) s'appliquent avant que les CRDs de wave
-50 soient réellement présentes → `no matches for kind "AgentgatewayBackend"` à la
première sync, rattrapé au retry par `selfHeal`.

Pour rendre les waves réellement bloquantes, restaurer le health check :

```bash
kubectl patch cm argocd-cm -n argocd --type merge -p '{"data":{"resource.customizations":"argoproj.io/Application:\n  health.lua: |\n    hs = {}\n    hs.status = \"Progressing\"\n    hs.message = \"\"\n    if obj.status ~= nil then\n      if obj.status.health ~= nil then\n        hs.status = obj.status.health.status\n        if obj.status.health.message ~= nil then\n          hs.message = obj.status.health.message\n        end\n      end\n    end\n    return hs\n"}}'
```

Sans ce patch, tout finit par converger, mais la première sync affiche une app en
erreur — gênant pendant une démo.

---

## 4. Problèmes connus et contournements manuels

### 4.1 Open WebUI n'affiche aucun modèle

`GET /v1/models` à travers la passerelle renvoie :

```
{"error":"Route 'Models' not implemented"}
```

Le type de route `Models` existe dans l'API agentgateway mais n'est **pas
implémenté pour le provider Anthropic** en v1.4.1. Ajouter
`policies.ai.routes: {"/v1/models": Models}` au backend ne change rien.

**Contournement manuel** (stocké en base Open WebUI, donc hors GitOps) :
*Admin Settings → Connections →* sur la connexion OpenAI, renseigner le champ
*Model IDs* avec les identifiants à exposer.

**Solution GitOps** (pas encore appliquée) : une `HTTPRoute` en `Exact: /v1/models`
plus une `AgentgatewayPolicy` avec `traffic.directResponse` renvoyant une liste
statique au format OpenAI.

---

## 5. MCP Inspector (debug)

> **Outil de développement, volontairement non exposé.** Son backend peut lancer
> des processus — [CVE-2025-49596](https://github.com/advisories/GHSA-7f8r-222p-6f5g)
> (CVSS 9.4) permettait une RCE non authentifiée avant la 0.14.1. Le Service est
> donc en `ClusterIP`, sans `HTTPRoute` ni `LoadBalancer`, et l'authentification
> par token reste active. Ne jamais ajouter `DANGEROUSLY_OMIT_AUTH`.

### 5.1 En cluster (déployé par Argo CD)

```bash
kubectl port-forward -n agentgateway-system svc/mcp-inspector 6274:6274
```

Puis ouvrir `http://localhost:6274?MCP_INSPECTOR_API_TOKEN=<le-token-du-§2.3>`.

Dans l'UI : transport **Streamable HTTP**, URL du serveur à tester. L'Inspector
tournant dans le cluster, il joint les Services par leur DNS interne :

| Cible | URL |
|---|---|
| Le serveur MCP Argo CD en direct | `http://argocd-mcp.agentgateway-system.svc:3000/mcp` |
| À travers la passerelle | `http://agentgateway-proxy.agentgateway-system.svc/mcp` |

Comparer les deux est le meilleur moyen de vérifier qu'une policy d'autorisation
filtre bien : la liste d'outils doit être plus courte via la passerelle.

### 5.2 En local, sans rien déployer

Alternative plus sûre pour une démo ponctuelle :

```bash
kubectl port-forward -n agentgateway-system svc/agentgateway-proxy 8081:80
npx @modelcontextprotocol/inspector      # écoute sur localhost:6274
# Dans l'UI : Streamable HTTP -> http://localhost:8081/mcp
```

**Limite connue :** l'Inspector ouvre un second port dynamique pour le bac à
sable « MCP Apps » (35701 lors de mes tests). Il n'est pas couvert par le
port-forward du §5.1 — les outils classiques fonctionnent, les ressources d'UI
MCP Apps non.

---

## 6. Structure du dépôt

```
appset.yaml                            2 ApplicationSets : bootstrap (base/*) et apps (apps/*)
project.yaml                           AppProject "platform"

base/agentgateway/                     Application CR des CRDs (wave -50) et du contrôleur
                                       (wave -40), plus le Gateway
base/agentgateway-configuration/       Backends LLM, HTTPRoute, policies, serveur MCP Argo CD,
                                       MCP Inspector (outil de debug, non exposé)
apps/openwebui/                        Application CR d'Open WebUI
```

Ajouter un composant = créer un dossier sous `base/` ou `apps/`. L'ApplicationSet
le détecte au prochain scan du repo (3 min par défaut, ou immédiatement via
webhook GitHub).

---

## 7. Vérifications

```bash
# GitOps
kubectl get applicationset -n argocd
kubectl get applications -n argocd

# Plan de données
kubectl get gateway,httproute -n agentgateway-system
kubectl get agentgatewaybackend,agentgatewaypolicy -n agentgateway-system
kubectl get pods,svc -n agentgateway-system

# Chemin de complétion de bout en bout (adapter le port du port-forward)
kubectl port-forward -n agentgateway-system svc/agentgateway-proxy 8081:80 &
curl -sS -w '\n[%{http_code}]\n' localhost:8081/v1/chat/completions \
  -H 'content-type: application/json' \
  -d '{"model":"claude-opus-4-6","messages":[{"role":"user","content":"dis juste ok"}],"max_tokens":16}'
```
