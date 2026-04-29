# Rotar un secret

Cómo actualizar las credenciales de un sistema cuando un password expira o se compromete.

---

## Proceso general

1. Re-sellar el secret con el nuevo valor
2. Commitear y pushear
3. ArgoCD aplica el nuevo SealedSecret automáticamente
4. Sealed Secrets controller recrea el secret en el cluster

---

## Ejemplos por sistema

### Grafana

```bash
kubectl create secret generic grafana-admin \
  --from-literal=admin-user=admin \
  --from-literal=admin-password=<nuevo-password> \
  --namespace grafana \
  --dry-run=client -o yaml \
| kubeseal --format yaml \
  > systems/grafana/resources/sealed-secret-grafana-admin.yaml

git add systems/grafana/resources/sealed-secret-grafana-admin.yaml
git commit -m "chore(grafana): rotate admin password"
git push origin main
```

### InfluxDB2

```bash
kubectl create secret generic influxdb2-auth \
  --from-literal=admin-user=admin \
  --from-literal=admin-password=<nuevo-password> \
  --from-literal=admin-token=<nuevo-token> \
  --namespace influxdb2 \
  --dry-run=client -o yaml \
| kubeseal --format yaml \
  > systems/influxdb2/resources/sealed-secret-influxdb2-auth.yaml
```

Si el token también cambia, actualizarlo en Grafana:

```bash
kubectl create secret generic influxdb2-token \
  --from-literal=admin-token=<nuevo-token> \
  --namespace grafana \
  --dry-run=client -o yaml \
| kubeseal --format yaml \
  > systems/grafana/resources/sealed-secret-influxdb2-token.yaml

git add systems/influxdb2/resources/ systems/grafana/resources/sealed-secret-influxdb2-token.yaml
git commit -m "chore(influxdb2): rotate admin credentials"
git push origin main
```

### AWX

```bash
kubectl create secret generic awx-admin-password \
  --from-literal=password=<nuevo-password> \
  --namespace awx \
  --dry-run=client -o yaml \
| kubeseal --format yaml \
  > systems/awx/resources/sealed-secret-awx-admin.yaml

git add systems/awx/resources/sealed-secret-awx-admin.yaml
git commit -m "chore(awx): rotate admin password"
git push origin main
```

### LDAP APIs

> Estos secrets van en el repo `ldap-apis-prod`.

```bash
cd ~/ldap-apis-prod

kubectl create secret generic ldap-apis-creds \
  --from-literal=auth-token=<nuevo-token> \
  --from-literal=ldap-password=<nuevo-password> \
  --namespace ldap-fastapi \
  --dry-run=client -o yaml \
| kubeseal --format yaml \
  > sealed-secret-ldap-apis.yaml

git add sealed-secret-ldap-apis.yaml
git commit -m "chore(ldap-apis): rotate credentials"
git push origin main
```

---

## Verificar que el secret fue rotado

```bash
# Verificar que el SealedSecret fue sincronizado
kubectl get sealedsecret -n <namespace> <nombre-secret>

# Verificar que el secret fue recreado con los nuevos valores
kubectl get secret -n <namespace> <nombre-secret> -o jsonpath='{.data.<key>}' | base64 -d
```
