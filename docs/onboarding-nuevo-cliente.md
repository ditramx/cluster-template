# Onboarding — Nuevo Cliente

Playbook paso a paso para levantar un cluster Kubernetes gestionado con ArgoCD desde cero.

**Tiempo estimado**: 2–3 horas  
**Prerequisitos**: Kubernetes operativo, acceso al servidor, dominio con wildcard TLS configurado

---

## Requisitos

| Herramienta | Versión | Notas |
|-------------|---------|-------|
| Ubuntu | 22.04 LTS | Otras distros funcionan, comandos pueden variar |
| k0s | v1.35.3 | Single-node controller |
| kubectl | v1.35.4 | Se instala por separado (k0s incluye `sudo k0s kubectl` pero no el binario standalone) |
| helm | v3.20.2 | |
| argocd CLI | v3.3.7 | |
| kubeseal | v0.27.2 | |
| git | cualquiera | |

---

## Paso 0 — Preparar el servidor

Ejecutar en el servidor como usuario con `sudo`.

### Instalar k0s

```bash
curl -sSLf https://get.k0s.sh | sudo sh

sudo k0s install controller --single

sudo k0s start

# Verificar que está corriendo
sudo systemctl status k0scontroller
```

### Instalar kubectl

k0s incluye su propio kubectl accesible como `sudo k0s kubectl`, pero es más cómodo tener el binario standalone. Instalarlo desde el repositorio oficial:

```bash
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update && sudo apt-get install -y kubectl

kubectl version --client
```

### Configurar kubeconfig

```bash
mkdir -p ~/.kube
sudo k0s kubeconfig admin > ~/.kube/config
chmod 600 ~/.kube/config

# Verificar acceso al cluster
kubectl get nodes
```

### Instalar Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

helm version --short
```

### Instalar ArgoCD CLI

```bash
curl -sSL -o /tmp/argocd \
  https://github.com/argoproj/argo-cd/releases/download/v2.14.11/argocd-linux-amd64

sudo install -m 555 /tmp/argocd /usr/local/bin/argocd

argocd version --client --short
```

### Instalar kubeseal

```bash
curl -sSL -o /tmp/kubeseal \
  https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.27.2/kubeseal-0.27.2-linux-amd64.tar.gz

tar -xzf /tmp/kubeseal -C /tmp kubeseal
sudo install -m 755 /tmp/kubeseal /usr/local/bin/kubeseal

kubeseal --version
```

### Instalar git y configurar SSH para GitHub

```bash
sudo apt-get update && sudo apt-get install -y git

# Generar llave SSH para el servidor (si no existe)
ssh-keygen -t ed25519 -C "<email-o-descripcion>" -f ~/.ssh/id_ed25519 -N ""

# Agregar la clave pública a GitHub: Settings → SSH Keys
cat ~/.ssh/id_ed25519.pub
```

---

## Paso 1 — Crear el repositorio del cliente

1. Ir a `github.com/ditramx/cluster-template`
2. Clic en **"Use this template"** → **"Create a new repository"**
3. Nombre del repo: `cluster-<cliente>` (ej. `cluster-acme`)
4. Visibilidad: **Private**
5. Clonar en el servidor:

```bash
git clone git@github.com:ditramx/cluster-<cliente>.git
cd cluster-<cliente>
```

---

## Paso 2 — Personalizar valores del cliente

Editar los siguientes archivos antes de continuar.

### MetalLB — rango de IPs

`infra/metallb/resources/ipaddresspool.yaml`

```yaml
spec:
  addresses:
    - 10.X.X.X-10.X.X.X   # una sola IP o rango, ej: 10.100.1.141-10.100.1.141
```

### Traefik — IP del LoadBalancer

`infra/traefik/values.yaml`

```yaml
service:
  loadBalancerIP: 10.X.X.X
```

### Dominios — ArgoCD y sistemas

Reemplazar el dominio en:

- `infra/traefik/resources/ingressroute-argocd.yaml`
- `systems/grafana/resources/ingressroute-grafana.yaml`
- `systems/influxdb2/resources/ingressroute-influxdb2.yaml`
- `systems/awx/resources/ingressroute-awx.yaml`

### InfluxDB2 — organización y bucket

`systems/influxdb2/values.yaml` y `systems/grafana/values.yaml`:

```yaml
# influxdb2/values.yaml
adminUser:
  organization: "<cliente>"
  bucket: "<cliente>"

# grafana/values.yaml — datasource
jsonData:
  organization: "<cliente>"
  defaultBucket: "<cliente>"
```

---

## Paso 3 — Generar Deploy Key para ArgoCD

```bash
ssh-keygen -t ed25519 -C "argocd-cluster-<cliente>" \
  -f ~/.ssh/argocd-cluster-<cliente> -N ""
```

Agregar la clave pública en GitHub:

1. `github.com/ditramx/cluster-<cliente>` → **Settings** → **Deploy keys** → **Add deploy key**
2. Título: `argocd-cluster-<cliente>`
3. Key: contenido de `~/.ssh/argocd-cluster-<cliente>.pub`
4. **Allow write access: NO**

---

## Paso 4 — Instalar ArgoCD

```bash
kubectl create namespace argocd

helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

helm upgrade --install argocd argo/argo-cd \
  --version 9.5.2 \
  --namespace argocd \
  --values argocd/values.yaml \
  --wait

kubectl rollout status deployment/argocd-server -n argocd --timeout=120s
```

Obtener la contraseña inicial:

```bash
kubectl get secret argocd-initial-admin-secret \
  -n argocd \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

Hacer login con la CLI:

```bash
kubectl port-forward svc/argocd-server -n argocd 8888:80 --address 127.0.0.1 &

argocd login localhost:8888 \
  --username admin \
  --password <password-del-paso-anterior> \
  --insecure
```

---

## Paso 5 — Registrar el repo y crear AppProjects

```bash
argocd repo add git@github.com:ditramx/cluster-<cliente>.git \
  --ssh-private-key-path ~/.ssh/argocd-cluster-<cliente> \
  --server localhost:8888 \
  --insecure

kubectl apply -f argocd/projects.yaml
```

---

## Paso 6 — Aplicar root-app y sincronizar infra

Aplicar la App-of-Apps raíz:

```bash
kubectl apply -f apps/root-app.yaml

argocd app sync root-app --server localhost:8888
```

Sincronizar infra en orden (cada uno espera a que el anterior esté healthy):

```bash
# MetalLB
argocd app sync metallb --server localhost:8888
argocd app wait metallb --health --server localhost:8888 --timeout 120
argocd app sync metallb-config --server localhost:8888

# OpenEBS
argocd app sync openebs --server localhost:8888
argocd app wait openebs --health --server localhost:8888 --timeout 180

# Traefik — primero crear el secret TLS manualmente
kubectl create namespace traefik
kubectl create secret tls ditra-tls \
  --cert=<ruta/certFile.crt> \
  --key=<ruta/star_cliente_com.key> \
  --namespace traefik

argocd app sync traefik --server localhost:8888
argocd app wait traefik --health --server localhost:8888 --timeout 120
argocd app sync traefik-config --server localhost:8888

# Sealed Secrets
argocd app sync sealed-secrets --server localhost:8888
argocd app wait sealed-secrets --health --server localhost:8888 --timeout 120
```

Migrar `ditra-tls` a SealedSecret (el SealedSecret ya está en el repo del template):

```bash
kubectl delete secret ditra-tls -n traefik
kubectl rollout restart deployment/sealed-secrets-controller -n sealed-secrets
kubectl rollout status deployment/sealed-secrets-controller -n sealed-secrets --timeout=60s

# Verificar que Sealed Secrets recreó el secret
kubectl get secret ditra-tls -n traefik
```

---

## Paso 7 — Respaldar la llave privada de Sealed Secrets

> **Crítico**: sin este backup, todos los SealedSecrets son irrecuperables si se pierde el controller.

```bash
kubectl get secret -n sealed-secrets \
  -l sealedsecrets.bitnami.com/sealed-secrets-key=active \
  -o yaml > sealed-secrets-master-key-<cliente>.yaml
```

Guardar el archivo en un lugar seguro **fuera del repo** (gestor de contraseñas, vault, etc.).

Para restaurar en caso de desastre:

```bash
kubectl apply -f sealed-secrets-master-key-<cliente>.yaml
kubectl rollout restart deployment/sealed-secrets-controller -n sealed-secrets
```

---

## Paso 8 — Sellar los secrets de los sistemas

### Grafana

```bash
kubectl create secret generic grafana-admin \
  --from-literal=admin-user=admin \
  --from-literal=admin-password=<password> \
  --namespace grafana \
  --dry-run=client -o yaml \
| kubeseal --format yaml \
  > systems/grafana/resources/sealed-secret-grafana-admin.yaml
```

### InfluxDB2

```bash
kubectl create secret generic influxdb2-auth \
  --from-literal=admin-user=admin \
  --from-literal=admin-password=<password> \
  --from-literal=admin-token=<token> \
  --namespace influxdb2 \
  --dry-run=client -o yaml \
| kubeseal --format yaml \
  > systems/influxdb2/resources/sealed-secret-influxdb2-auth.yaml

# Token para Grafana
kubectl create secret generic influxdb2-token \
  --from-literal=admin-token=<token> \
  --namespace grafana \
  --dry-run=client -o yaml \
| kubeseal --format yaml \
  > systems/grafana/resources/sealed-secret-influxdb2-token.yaml
```

### AWX

```bash
kubectl create secret generic awx-admin-password \
  --from-literal=password=<password> \
  --namespace awx \
  --dry-run=client -o yaml \
| kubeseal --format yaml \
  > systems/awx/resources/sealed-secret-awx-admin.yaml
```

### Notificaciones (Webex + SMTP opcional)

```bash
kubectl create secret generic argocd-notifications-secret \
  --from-literal=webex-webhook-url=<url-webhook> \
  --from-literal=smtp-host=<smtp-host> \
  --from-literal=smtp-from=<from-address> \
  --from-literal=smtp-username=<usuario> \
  --from-literal=smtp-password=<password> \
  --from-literal=smtp-use-tls=true \
  --namespace argocd \
  --dry-run=client -o yaml \
| kubeseal --format yaml \
  > infra/argocd-notifications/resources/sealed-secret-notifications.yaml
```

### LDAP APIs

> Estos secrets van en el repo `ldap-apis-prod`, no en el template.

```bash
cd ~/ldap-apis-prod
```

Credenciales de la API:

```bash
kubectl create secret generic ldap-apis-creds \
  --from-literal=auth-token=<token> \
  --from-literal=ldap-password=<password> \
  --namespace ldap-fastapi \
  --dry-run=client -o yaml \
| kubeseal --format yaml \
  > sealed-secret-ldap-apis.yaml
```

Registry pull secret (para imagen privada):

```bash
kubectl create secret docker-registry regcred \
  --docker-server=<registry> \
  --docker-username=<usuario> \
  --docker-password=<password> \
  --namespace ldap-fastapi \
  --dry-run=client -o yaml \
| kubeseal --format yaml \
  > sealed-secret-regcred.yaml
```

Commitear en `ldap-apis-prod`:

```bash
git add sealed-secret-ldap-apis.yaml sealed-secret-regcred.yaml
git commit -m "feat: seal secrets for ldap-apis"
git push origin main
```

Volver al repo del cliente:

```bash
cd ~/cluster-<cliente>
```

---

Commitear y pushear todos los SealedSecrets del cluster:

```bash
git add systems/ infra/argocd-notifications/
git commit -m "feat: seal secrets for all systems"
git push origin main
```

---

## Paso 9 — Sincronizar los sistemas

El servidor necesita resolver su propio dominio. Obtener la IP de Traefik y agregarla al `/etc/hosts` del servidor:

```bash
TRAEFIK_IP=$(kubectl get svc traefik -n traefik -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "$TRAEFIK_IP argocd.<dominio-cliente>" | sudo tee -a /etc/hosts
```

Hacer login:

```bash
argocd login argocd.<dominio-cliente> \
  --username admin \
  --password $(kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d) \
  --insecure
```

Sincronizar:

```bash
argocd app sync argocd-notifications
argocd app sync grafana
argocd app sync influxdb2
argocd app sync awx-operator
argocd app sync awx
argocd app sync ldap-apis
```

---

## Paso 10 — Verificación final

```bash
# Pods corriendo
kubectl get pods -A | grep -v Running | grep -v Completed

# IPs asignadas
kubectl get svc -A | grep LoadBalancer

# IngressRoutes activos
kubectl get ingressroute -A

# Apps en ArgoCD
argocd app list
```

Accesos esperados:

| Sistema | URL |
|---------|-----|
| ArgoCD | `https://argocd.<dominio-cliente>` |
| Grafana | `https://grafana.<dominio-cliente>` |
| InfluxDB2 | `https://influxdb.<dominio-cliente>` |
| AWX | `https://awx.<dominio-cliente>` |

---

## Referencia rápida — archivos a editar por cliente

| Archivo | Qué cambiar |
|---------|------------|
| `infra/metallb/resources/ipaddresspool.yaml` | Rango de IPs |
| `infra/traefik/values.yaml` | IP del LoadBalancer |
| `infra/traefik/resources/ingressroute-argocd.yaml` | Dominio ArgoCD |
| `systems/grafana/values.yaml` | Org/bucket InfluxDB2 y dominio |
| `systems/influxdb2/values.yaml` | Org y bucket |
| `systems/*/resources/ingressroute-*.yaml` | Dominio de cada sistema |
