# Agregar un sistema nuevo

Cómo incorporar una nueva aplicación al cluster gestionada por ArgoCD.

---

## Paso 1 — Crear la estructura de archivos

```bash
mkdir -p systems/<nombre>/resources
```

Si usa Helm, crear el `values.yaml`:

```bash
touch systems/<nombre>/values.yaml
```

---

## Paso 2 — Crear el Application de ArgoCD

Crear `apps/systems/<nombre>.yaml`.

### Si usa Helm con values locales (multi-source)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <nombre>
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "1"
    github-repo-url: "https://github.com/ditramx/cluster-<cliente>"
    notifications.argoproj.io/subscribe.on-sync-failed.webex: ""
    notifications.argoproj.io/subscribe.on-health-degraded.webex: ""
    notifications.argoproj.io/subscribe.on-deployed.webex: ""
spec:
  project: systems
  sources:
    - repoURL: <helm-repo-url>
      chart: <chart-name>
      targetRevision: "<version>"
      helm:
        valueFiles:
          - $values/systems/<nombre>/values.yaml
    - repoURL: git@github.com:ditramx/cluster-<cliente>.git
      targetRevision: HEAD
      ref: values
    - repoURL: git@github.com:ditramx/cluster-<cliente>.git
      targetRevision: HEAD
      path: systems/<nombre>/resources
  destination:
    server: https://kubernetes.default.svc
    namespace: <namespace>
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### Si usa Kustomize desde repo externo

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <nombre>
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "1"
    github-repo-url: "https://github.com/ditramx/cluster-<cliente>"
    notifications.argoproj.io/subscribe.on-sync-failed.webex: ""
    notifications.argoproj.io/subscribe.on-health-degraded.webex: ""
    notifications.argoproj.io/subscribe.on-deployed.webex: ""
spec:
  project: systems
  source:
    repoURL: git@github.com:ditramx/<nombre>-prod.git
    targetRevision: HEAD
    path: .
  destination:
    server: https://kubernetes.default.svc
    namespace: <namespace>
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

---

## Paso 3 — Agregar el Helm repo al AppProject (si aplica)

Si el chart viene de un repo Helm nuevo, agregarlo a `sourceRepos` en `argocd/projects.yaml`:

```yaml
spec:
  sourceRepos:
    - 'git@github.com:ditramx/cluster-<cliente>.git'
    - '<nuevo-helm-repo-url>'   # <- agregar aquí
```

---

## Paso 4 — Sellar los secrets del sistema

```bash
kubectl create secret generic <nombre-secret> \
  --from-literal=<key>=<value> \
  --namespace <namespace> \
  --dry-run=client -o yaml \
| kubeseal --format yaml \
  > systems/<nombre>/resources/sealed-secret-<nombre>.yaml
```

---

## Paso 5 — Commitear y pushear

```bash
git add apps/systems/<nombre>.yaml systems/<nombre>/ argocd/projects.yaml
git commit -m "feat(<nombre>): add <nombre> to ArgoCD"
git push origin main
```

ArgoCD detecta el nuevo Application y lo despliega automáticamente.

---

## Verificar

```bash
argocd app get <nombre>
kubectl get pods -n <namespace>
```
