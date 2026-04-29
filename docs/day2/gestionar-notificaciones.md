# Gestionar notificaciones

ArgoCD Notifications envía alertas automáticas a Webex (y opcionalmente por email) cuando una app cambia de estado.

---

## Cómo funciona

El sistema tiene 3 triggers configurados en `infra/argocd-notifications/resources/argocd-notifications-cm.yaml`:

| Trigger | Cuándo se dispara |
|---------|------------------|
| `on-sync-failed` | El sync falló o tuvo un error |
| `on-health-degraded` | La app pasó a estado Degraded |
| `on-deployed` | El sync fue exitoso y la app está Healthy |

Cada app se suscribe a los triggers mediante anotaciones en su Application yaml:

```yaml
annotations:
  notifications.argoproj.io/subscribe.on-sync-failed.webex: ""
  notifications.argoproj.io/subscribe.on-health-degraded.webex: ""
  notifications.argoproj.io/subscribe.on-deployed.webex: ""
```

---

## Agregar una app a las notificaciones

Agregar las anotaciones al Application yaml de la app en `apps/systems/<nombre>.yaml` o `apps/infra/<nombre>.yaml`:

```yaml
metadata:
  annotations:
    notifications.argoproj.io/subscribe.on-sync-failed.webex: ""
    notifications.argoproj.io/subscribe.on-health-degraded.webex: ""
    notifications.argoproj.io/subscribe.on-deployed.webex: ""
```

Commitear y pushear — ArgoCD aplica el cambio automáticamente.

---

## Quitar una app de las notificaciones

Eliminar las anotaciones del Application yaml y pushear.

---

## Cambiar el webhook de Webex

El webhook URL está sellado en `infra/argocd-notifications/resources/sealed-secret-notifications.yaml`. Para cambiarlo, re-sellar con el nuevo URL:

```bash
kubectl create secret generic argocd-notifications-secret \
  --from-literal=webex-webhook-url=<nuevo-url> \
  --namespace argocd \
  --dry-run=client -o yaml \
| kubeseal --format yaml \
  > infra/argocd-notifications/resources/sealed-secret-notifications.yaml

git add infra/argocd-notifications/resources/sealed-secret-notifications.yaml
git commit -m "chore(notifications): update webex webhook url"
git push origin main
```

---

## Activar notificaciones por email SMTP

El ConfigMap ya tiene el servicio SMTP configurado. Solo falta sellar las credenciales en el secret.

Re-sellar incluyendo los campos SMTP:

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

git add infra/argocd-notifications/resources/sealed-secret-notifications.yaml
git commit -m "chore(notifications): enable SMTP email alerts"
git push origin main
```

Para suscribir una app también a email, agregar las anotaciones SMTP en su Application yaml:

```yaml
annotations:
  notifications.argoproj.io/subscribe.on-sync-failed.smtp: "<email-destinatario>"
  notifications.argoproj.io/subscribe.on-health-degraded.smtp: "<email-destinatario>"
  notifications.argoproj.io/subscribe.on-deployed.smtp: "<email-destinatario>"
```

---

## Verificar que las notificaciones funcionan

```bash
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-notifications-controller --tail=50
```
