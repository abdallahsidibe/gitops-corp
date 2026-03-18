# GitOps Todo API — Déploiement avec Argo CD et Kustomize

Ce dépôt décrit le déploiement d’une API Todo basée sur l’image publique `docker.io/shatri/todo-api-node` en suivant les bonnes pratiques GitOps avec Argo CD et Kustomize (base + overlays `dev` et `staging`). Document concis, sans blabla.

## TL;DR — Lancer le déploiement
1) Pousser vos changements sur la branche suivie par Argo CD (ex: `main`).
2) Créer/mettre à jour l’Application Argo CD:
```bash
kubectl apply -n argocd -f application.yaml
```
3) Synchroniser dans Argo CD (UI) ou en CLI:
```bash
argocd app sync todo-api
```
4) Vérifier:
```bash
kubectl get ns todo-api
kubectl -n todo-api get deploy,po,svc -l app=todo-api
```
5) Tester depuis le cluster:
```bash
kubectl -n todo-api run tmp --rm -it --image=curlimages/curl --restart=Never -- \
  curl -sS http://todo-api.todo-api.svc.cluster.local/
```

## Prérequis
- Accès à un cluster Kubernetes et `kubectl` configuré
- Argo CD installé (namespace `argocd`)
- Optionnel: `argocd` CLI, `kustomize` CLI

## Structure Kustomize
```
apps/todo-api/
├── base/
│  ├── kustomization.yaml
│  ├── deployment.yaml          # image: docker.io/shatri/todo-api-node, port 3000
│  └── service.yaml             # Service 80 -> targetPort: http (3000)
└── overlays/
   ├── dev/
   │  ├── kustomization.yaml    # namespace: todo-api, labels, patch replicas
   │  └── patch-replicas.yaml   # replicas: 1
   └── staging/
      ├── kustomization.yaml
      └── patch-replicas.yaml   # replicas: 2
```
Application Argo CD: `application.yaml` pointe sur `apps/todo-api/overlays/dev`.

## Détails techniques
- Image: `docker.io/shatri/todo-api-node:latest` (pouvez pinner par digest)
- Conteneur écoute sur 3000 (`name: http`), Service expose 80 -> `targetPort: http`
- Probes HTTP sur `/`
- Sécurité durcie:
  - `runAsNonRoot`, `seccompProfile: RuntimeDefault`, `readOnlyRootFilesystem: true`
  - `allowPrivilegeEscalation: false`, `capabilities: drop [ALL]`
  - Volume `emptyDir` monté sur `/tmp`

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
kubectl -n todo-api logs deploy/todo-api -c todo-api --tail=200
```

## Nettoyage
- Supprimer l’application Argo CD (prune des ressources):
```bash
argocd app delete todo-api --yes
```
- Si namespace créé par Argo CD: il sera supprimé si géré comme ressource. Sinon:
```bash
kubectl delete ns todo-api
```

## (Historique — obsolète) Ancien TP — à des fins d'archive

## Principe (GitOps)
Le **GitOps** repose sur une règle simple :

- ✅ **Git est la source de vérité**
- ❌ On ne déploie jamais directement sur le cluster

### 🔄 Workflow
Dev → git push → ArgoCD détecte → Sync → Kubernetes

---

## Structure du projet
```
k8s-gitops-infra/
├── README.md
├── .gitignore
├── application.yaml
└── apps/
    └── todo-api/
        ├── namespace.yaml
        ├── deployment.yaml
        └── service.yaml
```

---

## Manifestes Kubernetes

### 1) Namespace
Chemin: `apps/todo-api/namespace.yaml`
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: todo-api
```

### 2) Deployment
Chemin: `apps/todo-api/deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: todo-api
  namespace: todo-api
  labels:
    app: todo-api
spec:
  replicas: 1
  selector:
    matchLabels:
      app: todo-api
  template:
    metadata:
      labels:
        app: todo-api
    spec:
      containers:
        - name: todo-api
          image: docker.io/library/nginx:alpine
          ports:
            - containerPort: 80
              name: http
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
```

### 3) Service
Chemin: `apps/todo-api/service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: todo-api
  namespace: todo-api
  labels:
    app: todo-api
spec:
  type: ClusterIP
  selector:
    app: todo-api
  ports:
    - name: http
      port: 80
      targetPort: http
      protocol: TCP
```

---

##  Application ArgoCD
Chemin: `application.yaml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: todo-api
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/abdallahsidibe/k8s-gitops-infra
    targetRevision: HEAD
    path: apps/todo-api
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

---

## 🚀 Déploiement

### 1) Installer ArgoCD (si nécessaire)
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 2) Déployer l’application via GitOps
```bash
kubectl apply -f application.yaml -n argocd
```

---

## Vérifications

### Application ArgoCD
```bash
kubectl get application -n argocd
```
Résultat attendu :
```
NAME       SYNC STATUS   HEALTH STATUS
todo-api   Synced        Healthy
```

### Pods
```bash
kubectl get pods -n todo-api
```

### Namespaces
```bash
kubectl get ns
```

---

## Accès à l’UI ArgoCD

### Port-forward
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
Ouvrir : https://localhost:8080

### Login (mot de passe initial admin)
```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

> Pense à changer le mot de passe après la première connexion.

---

## Test GitOps (important)

### 1) Modifier le nombre de replicas
Ouvrir `apps/todo-api/deployment.yaml` et changer :
```yaml
spec:
  replicas: 3
```

### 2) Push
```bash
git add apps/todo-api/deployment.yaml
git commit -m "scale: increase todo-api replicas to 3"
git push origin main
```

### 3) Résultat attendu
```bash
kubectl get pods -n todo-api
```
➡️ 3 pods automatiquement créés (sans `kubectl apply`).

---

## 🛡️ Test Self-Healing

### Sabotage manuel
```bash
kubectl scale deployment todo-api -n todo-api --replicas=1
```

### Résultat attendu
➡️ ArgoCD restaure automatiquement à 3 replicas (état désiré = Git).

---

## 🔁 Rollback

Lister l’historique et revenir à une révision précédente :
```bash
argocd app history todo-api
argocd app rollback todo-api 0
```
⚠️ Après rollback, la synchronisation automatique peut être désactivée — vérifier et réactiver si besoin.

---

## 🧰 Commandes utiles
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

---

## ⚠️ Points importants (expert)
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

## Aller plus loin (setup pro)
Envie d’aller au niveau supérieur ?
- Ingress + domaine
- HTTPS (cert-manager)
- CI/CD avec build Docker automatique
- Helm / Kustomize

👉 Avec ces briques, on passe du TP à un setup "entreprise".
