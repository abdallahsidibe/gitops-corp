# GitOps Todo API — Déploiement avec Argo CD et Kustomize

Ce dépôt décrit le déploiement d’une API Todo basée sur l’image publique `docker.io/shatri/todo-api-node` en suivant les bonnes pratiques GitOps avec Argo CD et Kustomize (base + overlays `dev` et `staging`).

---
## Principe (GitOps)
Le GitOps repose sur une règle simple :

✅ Git est la source de vérité

❌ On ne déploie jamais directement sur le cluster

🔄 Workflow

Dev → git push → ArgoCD détecte → Sync → Kubernetes

## Prérequis
- Accès à un cluster Kubernetes et `kubectl` configuré
- Argo CD installé (namespace `argocd`)
- Optionnel: `argocd` CLI, `kustomize` CLI

## Lancer le déploiement
 Pousser vos changements sur la branche suivie par Argo CD (ex: `main`).

### 1) Installer ArgoCD (si nécessaire)
```bash
    kubectl create namespace argocd
    kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 2) Déployer l’application via GitOps
```bash
    kubectl apply -f application.yaml -n argocd
```

### 3) Créer/mettre à jour l’Application Argo CD:
```bash
    kubectl apply -n argocd -f application.yaml
```
### 4) Synchroniser dans Argo CD (UI) ou en CLI:
```bash
    argocd app sync todo-api
```
   - Sans CLI Argo CD (alternative):
```bash
    kubectl -n argocd annotate application todo-api argocd.argoproj.io/refresh=hard --overwrite
```
### 5) Vérifier (namespace dev par défaut: todo-api-dev):
```bash
    kubectl get ns todo-api-dev
    kubectl -n todo-api-dev get deploy,po,svc -l app=todo-api
```
### 6) Tester depuis le cluster:
```bash
    kubectl -n todo-api-dev run tmp --rm -it --image=curlimages/curl --restart=Never -- \
      curl -sS http://todo-api.todo-api-dev.svc.cluster.local/
```

## Structure Kustomize
```
apps/todo-api/
├── base/
│  ├── kustomization.yaml
│  ├── deployment.yaml          # image: docker.io/shatri/todo-api-node, port 3000
│  └── service.yaml             # Service 80 -> targetPort: http (3000)
└── overlays/
   ├── dev/
   │  ├── kustomization.yaml    # namespace: todo-api-dev, labels, patch replicas
   │  ├── namespace.yaml        # Namespace todo-api-dev (optionnel si CreateNamespace=true)
   │  └── patch-replicas.yaml   # replicas: 1
   └── staging/
      ├── kustomization.yaml
      └── patch-replicas.yaml   # replicas: 2
```
Application Argo CD: `application.yaml` pointe sur `apps/todo-api/overlays/dev`. 

## Détails techniques
- Image: `docker.io/shatri/todo-api-node`
  - Base: l'image du Deployment référence `:latest`
  - Override via Kustomize: `apps/todo-api/base/kustomization.yaml` et `overlays/dev/kustomization.yaml` fixent `newTag: v1`
  - Recommandé: pinner par digest (`@sha256:<digest>`) pour l'immutabilité
- Conteneur écoute sur 3000 (`name: http`), Service expose 80 -> `targetPort: http`
- Probes HTTP sur `/`
- Sécurité durcie:
  - `runAsNonRoot`, `seccompProfile: RuntimeDefault`, `readOnlyRootFilesystem: true`
  - `allowPrivilegeEscalation: false`, `capabilities: drop [ALL]`
  - Volume `emptyDir` monté sur `/tmp`

## Important — Namespace dev (todo-api-dev)
- L’Application Argo CD du dépôt cible le namespace `todo-api-dev`:
  - Fichier: `application.yaml` → `spec.destination.namespace: todo-api-dev`
  - Option active: `syncOptions: [CreateNamespace=true]`
- Si vos manifests d’overlay `dev` définissent encore un objet `Namespace` nommé `todo-api`, alignez l’overlay pour éviter tout conflit:
  - Option A (recommandée): retirez `overlays/dev/namespace.yaml` et laissez Argo CD créer `todo-api-dev`.
  - Option B: ajoutez `namespace: todo-api-dev` dans `overlays/dev/kustomization.yaml` et gardez `CreateNamespace=true`.
- Forcer une resynchronisation sans CLI Argo CD:
```bash
    kubectl -n argocd annotate application todo-api argocd.argoproj.io/refresh=hard --overwrite
```
- Vérifier ensuite:
```bash
    kubectl get ns todo-api-dev
    kubectl -n todo-api-dev get deploy,po,svc -l app=todo-api
```

## Lancer en staging (nouvelle Application)
Deux options:
- Créer une deuxième Application Argo CD pointant sur `apps/todo-api/overlays/staging`
- Ou cloner/adapter `application.yaml` avec `path: apps/todo-api/overlays/staging`

Exemple rapide (nouvelle app):
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: todo-api-staging
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/abdallahsidibe/gitops-corp
    targetRevision: HEAD
    path: apps/todo-api/overlays/staging
  destination:
    server: https://kubernetes.default.svc
    namespace: todo-api
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

## Inspection locale (sans déployer)
- Générer les manifests:
```bash
    kustomize build apps/todo-api/overlays/dev | less
    kustomize build apps/todo-api/overlays/staging | less
```
- Conformément à GitOps, évitez `kubectl apply -f -` directement; validez via PR + Argo CD.

## Configuration
- Changer le nombre de réplicas: éditez `overlays/*/patch-replicas.yaml`
- Figer l’image par digest (recommandé):
```yaml
images:
- name: docker.io/shatri/todo-api-node
  newName: docker.io/shatri/todo-api-node@sha256:<digest>
```
- Variables/Secrets: utilisez `SealedSecrets` ou `External Secrets` (pas de secret en clair)
- Réseau/Scalabilité: ajoutez `NetworkPolicy` et `HPA` selon vos besoins

## Dépannage
- App en `OutOfSync`: `argocd app sync todo-api`
- Pods en CrashLoop avec FS en lecture seule: vérifiez le montage `/tmp` et les probes
- Service non joignable: vérifiez `selector`/labels, port nommé `http`, et endpoints
- Logs:
```bash
    kubectl -n todo-api-dev logs deploy/todo-api -c todo-api --tail=200
```

## Nettoyage
- Supprimer l’application Argo CD (prune des ressources):
```bash
    argocd app delete todo-api --yes
```
- Si le namespace a été créé par Argo CD (CreateNamespace=true), il sera supprimé automatiquement s’il est géré. Sinon, supprimez-le manuellement (dev):
```bash
    kubectl delete ns todo-api-dev
```

---

## Commandes utiles
```bash
# Liste des applications ArgoCD
    argocd app list

# Détails d'une application
    argocd app get todo-api

# Forcer une synchronisation
    argocd app sync todo-api

# Voir les logs de l'app
    argocd app logs todo-api

# Lister les pods
    kubectl get pods -n todo-api

# Surveiller en temps réel
    kubectl get pods -n todo-api --watch
```

## Audit de Déploiement : De la Synchronisation GitOps au Rendu Applicatif

### État Souhaité (Objectif)
Le but est d'obtenir une application parfaitement saine et synchronisée automatiquement depuis Git.

![Vue d'ensemble de la santé du cluster](./images/node1.png)|![Vue d'ensemble de la santé du cluster](./images/node2.png)|
![Vue d'ensemble de la santé du cluster](./images/node3.png)

*Figure 1 : Tableau de bord Argo CD confirmant que 100% des applications sont 'Healthy' et 'Synced'.*

###  1. Authentification et État Argo CD (CLI)
L'étape initiale consiste à s'authentifier auprès du serveur Argo CD pour vérifier l'état des ressources en ligne de commande. Cette capture confirme que l'application `todo-api` est synchronisée avec succès.

![Analyse CLI ArgoCD](./images/node4.png)
*Figure : Extraction des détails de l'application via `argocd app get`.*

---

###  2. Gestion du Flux Réseau (Port-Forward)
Pour accéder aux services internes du cluster depuis une machine locale, nous utilisons le tunnel de redirection de port via `kubectl`.

![Flux Port-Forward](./images/node5.png)

*Figure : Terminal illustrant les connexions actives vers le service `argocd-server`.*

---

###  3. Validation de l'API (Rendu Navigateur)

Une fois les ressources synchronisées et le tunnel établi, nous validons la disponibilité de l'application sur `localhost:3000`.

####  Message de Bienvenue (Root)
Vérification de l'endpoint principal de l'API Express.
![API Welcome](./images/node6.png)

####  Récupération des Données (/todos)
Validation de la persistance et de l'affichage des tâches provenant de la base de données.
![API Data](./images/node7.png)

---

##  Points importants (expert)
- ❌ Ne jamais modifier le cluster à la main
- ✅ Toujours passer par Git
- ❌ Pas de tag `latest` en prod
- ✅ Utiliser des versions immuables (ex: `v1.0.0`, digest SHA)
- ✅ ArgoCD = contrôle total de l’état du cluster

---

## ✅ Résultat final
- ✔ Déploiement GitOps fonctionnel
- ✔ Sync automatique Git → Kubernetes
- ✔ Self-healing actif
- ✔ Scaling via Git
- ✔ Rollback disponible

---

