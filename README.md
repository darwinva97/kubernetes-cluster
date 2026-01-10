# GitOps Bootstrap con ArgoCD para k3s

## Estructura del Repositorio

```
gitops/
├── bootstrap/              # Bootstrap inicial (solo se aplica UNA vez con kubectl)
│   └── argocd-install.yaml # Instalación de ArgoCD + App of Apps root
├── clusters/
│   └── k3s/
│       └── root-app.yaml   # App of Apps - punto de entrada autogestionado
├── infra/                  # Infraestructura base del cluster
│   ├── argocd/            # ArgoCD autogestionado
│   ├── namespaces/        # Namespaces del cluster
│   └── ingress/           # Ingress controller (traefik ya incluido en k3s)
└── apps/                   # Aplicaciones de usuario
    ├── _templates/        # Templates reutilizables
    └── example-app/       # Aplicación de ejemplo
```

## Descripción de Carpetas

| Carpeta | Propósito |
|---------|-----------|
| `bootstrap/` | Contiene lo mínimo para iniciar ArgoCD. Se aplica UNA SOLA VEZ con kubectl. |
| `clusters/` | Configuración por cluster. El `root-app.yaml` es el App of Apps que gestiona todo. |
| `infra/` | Componentes de infraestructura: ArgoCD (autogestionado), namespaces, ingress, etc. |
| `apps/` | Aplicaciones de negocio desplegadas por ArgoCD. |

## Principios GitOps Aplicados

1. **Git es la única fuente de verdad** - Todo cambio pasa por Git
2. **ArgoCD se autogestiona** - Después del bootstrap, ArgoCD se actualiza desde Git
3. **Sin loops de sincronización** - Configuración correcta de `ignoreDifferences`
4. **Kubernetes NUNCA escribe a Git** - Flujo unidireccional

## Comandos de Bootstrap (SOLO UNA VEZ)

```bash
# 1. Crear namespace argocd
kubectl create namespace argocd

# 2. Aplicar instalación de ArgoCD + Root App
kubectl apply -f bootstrap/argocd-install.yaml

# 3. Esperar a que ArgoCD esté listo
kubectl wait --for=condition=available deployment/argocd-server -n argocd --timeout=300s

# 4. Obtener password inicial de admin
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo

# 5. (Opcional) Port-forward para acceder a la UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

## ⚠️ LO QUE NUNCA DEBES HACER DESPUÉS DEL BOOTSTRAP

1. **NUNCA** volver a ejecutar `kubectl apply` sobre manifiestos gestionados por ArgoCD
2. **NUNCA** editar recursos directamente con `kubectl edit`
3. **NUNCA** usar `kubectl delete` en recursos gestionados
4. **NUNCA** configurar webhooks que escriban de Kubernetes a Git

## Agregar Nuevas Aplicaciones (Solo Git)

1. Crear carpeta en `apps/mi-nueva-app/`
2. Agregar manifiestos Kubernetes
3. Crear Application YAML en `apps/mi-nueva-app/application.yaml`
4. Agregar referencia en `clusters/k3s/root-app.yaml` o crear ApplicationSet
5. `git commit && git push`
6. ArgoCD detecta y aplica automáticamente
