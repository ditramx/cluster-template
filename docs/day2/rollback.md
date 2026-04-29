# Rollback

Cómo volver a una versión anterior de un sistema.

Hay dos enfoques según la urgencia.

---

## Opción A — Git revert (recomendado)

Mantiene el historial limpio y ArgoCD re-sincroniza desde el commit revertido.

```bash
# Ver el historial de commits
git log --oneline apps/systems/<nombre-app>.yaml

# Revertir el commit que introdujo el problema
git revert <commit-hash>

git push origin main
```

ArgoCD detecta el nuevo commit y sincroniza automáticamente.

---

## Opción B — ArgoCD rollback (rollback rápido sin tocar Git)

Útil para restaurar inmediatamente mientras se investiga el problema. **No modifica el repo** — al próximo sync ArgoCD vuelve al estado del repo.

Desde la CLI:

```bash
# Ver el historial de deployments de la app
argocd app history <nombre-app>

# Hacer rollback a una revisión específica
argocd app rollback <nombre-app> <revision-id>
```

Desde la UI: **Applications** → app → **History and Rollback** → seleccionar la revisión → **Rollback**.

> Después del rollback por UI/CLI, corregir el problema en Git y pushear para que el repo quede consistente con el cluster.

---

## Verificar el rollback

```bash
argocd app get <nombre-app>
kubectl get pods -n <namespace> -w
```
