# Actualizar un sistema

Cómo actualizar la versión de un chart Helm o los valores de configuración de un sistema existente.

---

## Actualizar versión de chart

Editar el `targetRevision` en el Application yaml correspondiente en `apps/systems/` o `apps/infra/`.

Ejemplo — actualizar Grafana de `10.5.15` a `10.6.0`:

```yaml
# apps/systems/grafana.yaml
sources:
  - repoURL: https://grafana.github.io/helm-charts
    chart: grafana
    targetRevision: "10.6.0"   # <- cambiar aquí
```

Commitear y pushear:

```bash
git add apps/systems/grafana.yaml
git commit -m "chore(grafana): bump chart to 10.6.0"
git push origin main
```

ArgoCD detecta el cambio y sincroniza automáticamente (auto-sync activado).

---

## Actualizar valores de configuración

Editar el `values.yaml` del sistema en `systems/<nombre>/values.yaml`.

Commitear y pushear — ArgoCD aplica los cambios automáticamente.

---

## Verificar el despliegue

```bash
argocd app get <nombre-app>
```

O desde la UI: **Applications** → seleccionar la app → verificar que el estado sea `Synced` y `Healthy`.

---

## Si el sync falla

Ver los logs del sync:

```bash
argocd app sync <nombre-app>
argocd app logs <nombre-app>
```

O desde la UI: **Applications** → app → **Events** para ver el error.
