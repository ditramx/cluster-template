# Troubleshooting

Diagnóstico de los problemas más comunes en el cluster.

---

## App en estado OutOfSync

ArgoCD detecta diferencias entre el repo y el cluster.

```bash
# Ver qué recursos están fuera de sync
argocd app diff <nombre-app>
```

**Causas comunes:**

- Cambio manual directo en el cluster con `kubectl` — ArgoCD lo detecta como drift. Solución: hacer sync para que ArgoCD restaure el estado del repo.
- Campo auto-generado por el cluster (ej. `caBundle` en CRDs) — ignorar o configurar `ignoreDifferences` en el Application yaml.

Forzar sync:

```bash
argocd app sync <nombre-app>
```

---

## App en estado Degraded

La app sincronizó pero algún recurso no está healthy (pod crasheando, deployment sin réplicas listas, etc.).

```bash
# Ver el estado detallado
argocd app get <nombre-app>

# Ver pods del namespace
kubectl get pods -n <namespace>

# Ver logs del pod con problemas
kubectl logs -n <namespace> <nombre-pod> --previous
```

**Causas comunes:**

- Secret no encontrado — verificar que el SealedSecret esté `SYNCED: True`:
  ```bash
  kubectl get sealedsecret -n <namespace>
  ```
- Imagen no encontrada — verificar el tag y las credenciales del registry (`regcred`).
- PVC sin provisionar — verificar que OpenEBS esté healthy:
  ```bash
  kubectl get pvc -n <namespace>
  ```

---

## App en estado Unknown

ArgoCD no puede determinar el estado de la app.

```bash
kubectl get pods -n argocd | grep argocd-application-controller
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller --tail=50
```

Generalmente indica un problema con el Application Controller. Reiniciarlo suele resolver:

```bash
kubectl rollout restart deployment/argocd-application-controller -n argocd
```

---

## Sync fallido — error de permisos

```
ComparisonError: Forbidden: ... is not permitted in project
```

El recurso o repo no está permitido en el AppProject. Verificar `argocd/projects.yaml`:

- `sourceRepos` — el repo del recurso debe estar listado
- `destinations` — el namespace de destino debe estar listado

Actualizar `projects.yaml`, commitear y pushear. Luego forzar sync del AppProject:

```bash
kubectl apply -f argocd/projects.yaml
```

---

## ArgoCD no puede acceder al repo

```
Failed to fetch default: ssh: handshake failed
```

El deploy key puede haber expirado o fue revocado en GitHub.

```bash
# Ver el error completo
argocd repo list

# Re-registrar el repo
argocd repo add git@github.com:ditramx/cluster-<cliente>.git \
  --ssh-private-key-path ~/.ssh/argocd-cluster-<cliente> \
  --upsert
```

---

## Sealed Secrets no descifra un secret

```bash
# Ver eventos del SealedSecret
kubectl describe sealedsecret -n <namespace> <nombre>
```

Si el error es `no key could decrypt secret` significa que el SealedSecret fue encriptado con una llave diferente a la que tiene el controller (por ejemplo, tras una reinstalación del cluster sin restaurar el backup).

Restaurar el backup de la llave:

```bash
kubectl apply -f sealed-secrets-master-key-<cliente>.yaml
kubectl rollout restart deployment/sealed-secrets-controller -n sealed-secrets
```

Re-sellar todos los secrets y pushear.
