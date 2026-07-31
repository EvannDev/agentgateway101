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
| `github-mcp-secret` | `Authorization` | `AgentgatewayBackend/mcp-tools`, target `github` (`static.policies.auth`) |

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

### 2.3 PAT GitHub pour le serveur MCP GitHub

Le target `github` du backend `mcp-tools` pointe sur le serveur MCP **hébergé par
GitHub** (`https://api.githubcopilot.com/mcp/`). Le serveur auto-hébergé
`ghcr.io/github/github-mcp-server` n'est pas utilisable ici : il ne parle que
stdio, alors qu'un target agentgateway exige `SSE` ou `StreamableHTTP`.

Créer un PAT sur https://github.com/settings/personal-access-tokens — de
préférence *fine-grained*, limité aux dépôts concernés, en lecture seule pour
commencer. Le jeu d'outils du serveur distant est large : donner le minimum.

```bash
export GITHUB_MCP_PAT='github_pat_...'
envsubst < base/agentgateway-configuration/github-mcp-apikey.yaml.example \
  > base/agentgateway-configuration/github-mcp-apikey.yaml
kubectl apply -f base/agentgateway-configuration/github-mcp-apikey.yaml
```

Le PAT est stocké **brut**, sans préfixe `Bearer ` : agentgateway lit la clé
`Authorization` du Secret et émet `Authorization: Bearer <valeur>`.

> **Créer le Secret AVANT d'appliquer le backend.** Un `secretRef` non résolu met
> l'`AgentgatewayBackend` en `Accepted=False (TranslationError)` — le backend
> **entier** est rejeté, pas seulement le target concerné. La route `/mcp` perd sa
> destination et répond `500 service not found`, Argo CD compris. `failureMode:
> FailOpen` ne protège pas de ça : il couvre les défaillances à l'exécution d'un
> target, pas une erreur de traduction de la configuration.
>
> Diagnostic : `kubectl get agentgatewaybackend mcp-tools -n agentgateway-system -o jsonpath='{.status.conditions[*].message}'`

Le pod du Gateway doit pouvoir sortir vers `api.githubcopilot.com:443`. Si tu
restreins l'egress plus tard (NetworkPolicy, ou networking `limited` sur
l'environnement), il faut autoriser ce domaine explicitement — sinon l'appel
échoue en silence.

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

### 3.2 SSO GitHub via Dex (Argo CD)

Entièrement manuel : `argocd-cm` n'est pas géré par ce dépôt.

**Trois valeurs doivent être identiques, protocole compris**, sinon Argo CD refuse
la redirection avec *« Invalid redirect URL: the protocol and host (including
port) must match »* :

1. `url` dans `argocd-cm`
2. `connectors[].config.redirectURI` dans `dex.config` (= `<url>/api/dex/callback`)
3. l'*Authorization callback URL* de l'OAuth App GitHub — **hors cluster**

```bash
kubectl patch cm argocd-cm -n argocd --type merge \
  -p '{"data":{"url":"https://argocd.evann-deb.fr"}}'

kubectl edit cm argocd-cm -n argocd     # redirectURI -> https://.../api/dex/callback

# Dex ne relit pas sa configuration à chaud
kubectl rollout restart deploy/argocd-dex-server -n argocd
```

**Le piège rencontré :** `url` était en `http://` alors que `argocd-server` sert en
TLS (`server.insecure` non activé) et renvoie un **307 vers `https://`**.
L'origine du navigateur devenait donc `https://` — protocole différent de `url`,
et la comparaison échoue. Vérification :

```bash
curl -sS -I http://argocd.evann-deb.fr | head -3   # doit-il rediriger ?
kubectl get cm argocd-cm -n argocd -o jsonpath='{.data.url}'
```

Le certificat servi est celui d'Argo CD (`issuer= /O=Argo CD`, auto-signé) : rien
ne termine TLS en amont, l'exposition Tailscale fait du passthrough TCP. D'où
l'avertissement du navigateur. Pour l'éliminer, terminer TLS côté Tailscale et
activer `server.insecure: "true"` dans `argocd-cmd-params-cm`.

Pour conserver un accès par port-forward en parallèle, ajouter les origines
supplémentaires plutôt que de changer `url` :

```yaml
data:
  additionalUrls: |
    - https://localhost:8080
```

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

### 5.1 En local (méthode recommandée)

```bash
kubectl port-forward -n agentgateway-system svc/agentgateway-proxy 8081:80
npx @modelcontextprotocol/inspector      # écoute sur localhost:6274
```

Dans l'UI : transport **Streamable HTTP**, puis l'une de ces cibles :

| Cible | URL (via le port-forward) |
|---|---|
| À travers la passerelle | `http://localhost:8081/mcp` |
| Le serveur MCP Argo CD en direct | port-forward séparé sur `svc/argocd-mcp 3000:3000`, puis `http://localhost:3000/mcp` |

Comparer les deux est le meilleur moyen de vérifier qu'une policy d'autorisation
filtre bien : la liste d'outils doit être plus courte via la passerelle.

**Les noms d'outils diffèrent entre les deux vues.** Le backend est en
`prefixMode: Always`, donc la passerelle expose `argocd_list_applications` et
`github_get_me` là où le serveur attaqué en direct annonce `list_applications`.
C'est voulu : le nommage reste stable quand on ajoute ou retire un target. Toute
règle CEL `mcp.tool.name` doit utiliser la forme préfixée.

### 5.2 Pourquoi il n'est pas déployé en cluster (bug amont)

Un déploiement en cluster a été tenté puis retiré du dépôt. Le pod démarrait et
écrivait bien `/home/node/.mcp-inspector/mcp.json`, mais **l'UI n'affichait aucun
serveur** : `GET /api/servers` renvoyait 500 avec
`Couldn't access platform storage: PermissionDenied`.

En 2.0.0 l'Inspector lit sa liste via un trousseau système, absent d'un conteneur,
et la dégradation prévue ne s'engage pas. Reproduit dans un conteneur **sans aucun
durcissement** — ce n'était donc pas lié au `readOnlyRootFilesystem` ni au
`fsGroup` du manifeste. Suivi en amont :

- [#1845](https://github.com/modelcontextprotocol/inspector/issues/1845) — le symptôme
- [#1848](https://github.com/modelcontextprotocol/inspector/issues/1848) — la cause (`KeyringSecretStore`) et le correctif proposé
- [#1858](https://github.com/modelcontextprotocol/inspector/issues/1858) — v2 se bloque silencieusement en `streamable-http`

En attendant un correctif : rester sur le §5.1. Si la 2.x se bloque aussi en local,
épingler la dernière 1.x (`npx @modelcontextprotocol/inspector@1.0.1`) — elle ne
persiste pas de catalogue, on saisit l'URL à chaque session.

**Autre limite :** l'Inspector ouvre un second port dynamique pour le bac à sable
« MCP Apps » (35701 et 45857 selon les essais), non couvert par un port-forward.
Les outils classiques fonctionnent, les ressources d'UI MCP Apps non.

---

## 6. Structure du dépôt

```
appset.yaml                            2 ApplicationSets : bootstrap (base/*) et apps (apps/*)
project.yaml                           AppProject "platform"

base/agentgateway/                     Application CR des CRDs (wave -50) et du contrôleur
                                       (wave -40), plus le Gateway
base/agentgateway-configuration/       Backends LLM et MCP, HTTPRoutes, policies,
                                       serveur MCP Argo CD
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
