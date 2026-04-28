# Agregar un nuevo cliente

Para levantar el cluster de un nuevo cliente seguir el playbook completo de onboarding:

→ [`docs/onboarding-nuevo-cliente.md`](../onboarding-nuevo-cliente.md)

---

## Resumen rápido

1. Crear repo desde el template: `cluster-<nuevo-cliente>`
2. Personalizar valores (IPs, dominios, org/bucket)
3. Generar deploy key SSH y registrarla en GitHub
4. Instalar k0s + tooling en el servidor del cliente
5. Instalar ArgoCD vía Helm
6. Registrar repo y crear AppProjects
7. Aplicar root-app y sincronizar infra
8. Respaldar llave de Sealed Secrets
9. Sellar secrets de todos los sistemas
10. Sincronizar sistemas

Cada cliente tiene su propio repo Git aislado — los cambios en un cluster no afectan a otros.
