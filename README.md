# TP #2 — Premier déploiement avec ArgoCD

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
