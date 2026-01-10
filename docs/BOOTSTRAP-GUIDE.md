# Guía de Bootstrap - ArgoCD en k3s

## Pre-requisitos

- Cluster k3s funcionando
- kubectl configurado con acceso al cluster
- Git repository creado (público o privado)

## Paso 0: Preparar el Repositorio Git

```bash
# Clonar o crear el repositorio
git clone https://github.com/TU-ORG/gitops.git
cd gitops

# Copiar toda la estructura creada
# (ya debería estar si seguiste las instrucciones)

# Editar URLs del repositorio en todos los archivos
find . -name "*.yaml" -exec sed -i 's|https://github.com/TU-ORG/gitops.git|https://github.com/TU-USUARIO/TU-REPO.git|g' {} \;

# Commit y push
git add .
git commit -m "Initial GitOps structure"
git push origin main
```

## Paso 1: Instalar ArgoCD

```bash
# Crear namespace
kubectl create namespace argocd

# Instalar ArgoCD (manifest oficial estable)
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Esperar a que todos los pods estén ready
kubectl wait --for=condition=available deployment --all -n argocd --timeout=300s
```

## Paso 2: Configurar Acceso a Repositorio Privado (Opcional)

Si tu repositorio es privado:

```bash
# Crear secret con credenciales
kubectl create secret generic repo-gitops \
  --namespace argocd \
  --from-literal=type=git \
  --from-literal=url=https://github.com/TU-ORG/gitops.git \
  --from-literal=username=git \
  --from-literal=password=ghp_TU_PERSONAL_ACCESS_TOKEN

# Agregar label para que ArgoCD lo detecte
kubectl label secret repo-gitops -n argocd argocd.argoproj.io/secret-type=repository
```

## Paso 3: Aplicar Root Application (Bootstrap)

```bash
# Aplicar la configuración inicial y root app
kubectl apply -f bootstrap/argocd-install.yaml
```

## Paso 4: Verificar Instalación

```bash
# Ver estado de las applications
kubectl get applications -n argocd

# Debería mostrar:
# NAME            SYNC STATUS   HEALTH STATUS
# root            Synced        Healthy
# argocd          Synced        Healthy
# namespaces      Synced        Healthy
# ingress-config  Synced        Healthy
# example-app     Synced        Healthy
```

## Paso 5: Acceder a ArgoCD UI

### Opción A: Port-forward (desarrollo)

```bash
# Port-forward
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Obtener password admin
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo

# Acceder en: https://localhost:8080
# Usuario: admin
# Password: (el obtenido arriba)
```

### Opción B: Ingress (producción)

1. Configurar DNS para `argocd.example.com` → IP del cluster
2. Crear/configurar certificado TLS
3. ArgoCD estará disponible en `https://argocd.example.com`

## Paso 6: Cambiar Password Admin (Recomendado)

```bash
# Usando ArgoCD CLI
argocd login localhost:8080 --username admin --password <password-inicial>
argocd account update-password

# O desde la UI: Settings → User Info → Update Password
```

## Paso 7: (Opcional) Eliminar Secret de Password Inicial

```bash
kubectl delete secret argocd-initial-admin-secret -n argocd
```

---

## 🎉 ¡Bootstrap Completo!

A partir de ahora:

- **TODO** se gestiona desde Git
- **NUNCA** más ejecutar `kubectl apply` sobre recursos gestionados
- Para agregar apps: editar Git → push → ArgoCD sincroniza automáticamente

---

## Comandos Útiles Post-Bootstrap

```bash
# Ver todas las applications
kubectl get applications -n argocd

# Ver estado detallado de una app
kubectl describe application example-app -n argocd

# Ver logs del controller
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller -f

# Forzar sincronización (solo emergencias)
kubectl patch application example-app -n argocd --type merge -p '{"operation": {"sync": {}}}'

# Refresh desde Git (sin aplicar)
kubectl patch application example-app -n argocd --type merge -p '{"metadata": {"annotations": {"argocd.argoproj.io/refresh": "hard"}}}'
```

## Troubleshooting

### App stuck en "Progressing"

```bash
# Ver eventos
kubectl get events -n argocd --sort-by='.lastTimestamp'

# Ver logs del controller
kubectl logs -n argocd deployment/argocd-application-controller
```

### Sync failed

```bash
# Ver detalles del error
kubectl describe application <app-name> -n argocd | grep -A 20 "Status:"

# Ver conditions
kubectl get application <app-name> -n argocd -o jsonpath='{.status.conditions}' | jq
```

### OutOfSync constante (loop)

Verificar que `ignoreDifferences` está configurado correctamente para:
- `metadata.resourceVersion`
- `metadata.generation`
- `metadata.managedFields`
- `status`

```bash
# Ver qué campos están causando el diff
kubectl get application <app-name> -n argocd -o jsonpath='{.status.resources}' | jq
```
