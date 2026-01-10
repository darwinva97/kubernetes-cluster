# Flujo GitOps Correcto

## Diagrama de Flujo

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ Desarrollador│────▶│    Git      │────▶│   ArgoCD    │────▶│ Kubernetes  │
│   (push)    │     │ (source of  │     │  (detecta   │     │  (aplica)   │
│             │     │   truth)    │     │   cambios)  │     │             │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
                           │                   │
                           │                   │
                           ▼                   ▼
                    ┌─────────────────────────────────┐
                    │  FLUJO UNIDIRECCIONAL           │
                    │  Git ──▶ ArgoCD ──▶ Kubernetes  │
                    │                                 │
                    │  ❌ Kubernetes NUNCA escribe    │
                    │     de vuelta a Git             │
                    └─────────────────────────────────┘
```

## Ejemplo Paso a Paso

### 1. Cambiar la imagen de una aplicación

```bash
# 1. Editar el archivo localmente
vim apps/example-app/deployment.yaml

# Cambiar la línea:
#   image: nginx:1.25-alpine
# Por:
#   image: nginx:1.26-alpine

# 2. Commit y push
git add apps/example-app/deployment.yaml
git commit -m "feat(example-app): upgrade nginx to 1.26"
git push origin main
```

### 2. ArgoCD detecta el cambio

ArgoCD realiza polling cada 3 minutos (configurable) o recibe un webhook.

```
┌──────────────────────────────────────────────────────────┐
│ ArgoCD UI                                                │
├──────────────────────────────────────────────────────────┤
│ App: example-app                                         │
│ Status: OutOfSync ⚠️                                     │
│ Health: Healthy ✓                                        │
│                                                          │
│ Diff:                                                    │
│ - image: nginx:1.25-alpine                               │
│ + image: nginx:1.26-alpine                               │
└──────────────────────────────────────────────────────────┘
```

### 3. ArgoCD aplica el cambio (automático)

Con `automated.selfHeal: true` y `automated.prune: true`:

1. ArgoCD detecta diferencia entre Git y cluster
2. ArgoCD aplica el nuevo manifiesto
3. Kubernetes hace rolling update del Deployment
4. Pods nuevos se crean con la nueva imagen
5. Pods antiguos se terminan
6. Estado final: **Synced ✓**

```
┌──────────────────────────────────────────────────────────┐
│ ArgoCD UI                                                │
├──────────────────────────────────────────────────────────┤
│ App: example-app                                         │
│ Status: Synced ✓                                         │
│ Health: Healthy ✓                                        │
│                                                          │
│ Last Sync: 2 minutes ago                                 │
│ Revision: abc123 (feat: upgrade nginx to 1.26)           │
└──────────────────────────────────────────────────────────┘
```

## Escalar Replicas Correctamente

### ❌ INCORRECTO
```bash
# NO hacer esto:
kubectl scale deployment example-app --replicas=5 -n apps
```

### ✅ CORRECTO
```bash
# Editar en Git:
vim apps/example-app/deployment.yaml
# Cambiar replicas: 2 → replicas: 5

git add . && git commit -m "scale: increase replicas to 5" && git push
```

## Agregar Nueva Aplicación

### Paso 1: Crear estructura

```bash
mkdir -p apps/nueva-app
```

### Paso 2: Crear manifiestos

```yaml
# apps/nueva-app/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nueva-app
  namespace: apps
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nueva-app
  template:
    metadata:
      labels:
        app: nueva-app
    spec:
      containers:
        - name: app
          image: mi-imagen:v1.0.0
          ports:
            - containerPort: 8080
```

### Paso 3: Agregar Application en root-app.yaml

```yaml
# clusters/k3s/root-app.yaml (agregar al final)
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nueva-app
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/TU-ORG/gitops.git
    targetRevision: main
    path: apps/nueva-app
  destination:
    server: https://kubernetes.default.svc
    namespace: apps
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### Paso 4: Commit y push

```bash
git add apps/nueva-app clusters/k3s/root-app.yaml
git commit -m "feat: add nueva-app"
git push origin main
```

### Paso 5: Verificar

```bash
# ArgoCD detectará y desplegará automáticamente
kubectl get applications -n argocd
kubectl get pods -n apps
```
