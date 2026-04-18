# cluster-template

Template de repositorio GitOps para gestión de clusters Kubernetes con ArgoCD usando el patrón App-of-Apps.

## Uso

1. Hacer clic en **"Use this template"** → Create new repository
2. Clonar el nuevo repo: `git clone git@github.com:<org>/<cluster-name>.git`
3. Completar los `values.yaml` de cada componente con los valores del cliente
4. Instalar ArgoCD: `helm install argocd argo/argo-cd -n argocd --create-namespace -f argocd/values.yaml`
5. Aplicar la app raíz: `kubectl apply -f apps/root-app.yaml`

## Estructura

```
.
├── apps/
│   ├── infra/          # Applications de infraestructura base (MetalLB, OpenEBS, Traefik, Sealed Secrets)
│   ├── systems/        # Applications de sistemas de negocio
│   └── root-app.yaml   # App-of-Apps raíz — punto de entrada de ArgoCD
├── argocd/
│   ├── values.yaml     # Configuración de ArgoCD (Helm)
│   └── projects.yaml   # AppProjects: infra y systems
├── infra/
│   ├── metallb/        # IPAddressPools, L2Advertisement, values
│   ├── openebs/        # StorageClass default, values
│   └── traefik/        # IngressClass, middlewares, values
└── systems/
    ├── awx/
    ├── grafana/
    ├── influxdb2/
    ├── ldap-apis/
    ├── techflow-db/
    └── znuny/
```

## Convenciones

- `apps/infra/` usa sync waves negativas para garantizar orden de despliegue: MetalLB (-3) → OpenEBS (-2) → Traefik/Sealed Secrets (-1)
- `apps/systems/` usa sync wave 0 (por defecto)
- Los secrets se gestionan con **Sealed Secrets** — nunca se commitean valores en texto plano
- `argocd/projects.yaml` define los namespaces permitidos por proyecto

## Requisitos

- Kubernetes 1.28+
- Helm 3.x
- ArgoCD 2.x
- kubeseal (para gestión de secrets)
