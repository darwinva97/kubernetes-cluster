# ⚠️ Antipatrones GitOps - LO QUE NO DEBES HACER

## Antipatrón 1: Modificar recursos directamente en Kubernetes

### ❌ INCORRECTO
```bash
# Escalar deployment directamente
kubectl scale deployment example-app --replicas=5 -n apps

# Editar recursos en vivo
kubectl edit deployment example-app -n apps

# Aplicar manifiestos manualmente
kubectl apply -f mi-deployment.yaml
```

### Por qué es malo
- ArgoCD detecta **drift** entre Git y cluster
- Con `selfHeal: true`, ArgoCD **revierte** el cambio
- El cambio se pierde
- Genera confusión sobre el estado real

### ✅ CORRECTO
```bash
# Editar en Git y hacer push
vim apps/example-app/deployment.yaml
git commit -am "scale replicas" && git push
```

---

## Antipatrón 2: Kubernetes escribe a Git (Image Updater mal configurado)

### ❌ INCORRECTO - Loop infinito

```yaml
# Configuración que causa loops:
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  annotations:
    # ArgoCD Image Updater escribe a Git
    argocd-image-updater.argoproj.io/write-back-method: git
spec:
  syncPolicy:
    automated:
      selfHeal: true  # ArgoCD sincroniza desde Git
```

### Ciclo del loop:
```
1. Image Updater detecta nueva imagen
2. Image Updater hace commit a Git (v1.0.1)
3. ArgoCD detecta cambio en Git
4. ArgoCD aplica al cluster
5. Kubernetes actualiza pod
6. Image Updater detecta "cambio" (mismo tag diferente digest)
7. Image Updater hace commit a Git (v1.0.1 again)
8. LOOP INFINITO
```

### ✅ CORRECTO
```yaml
# Opción A: No usar write-back automático
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  annotations:
    # NO escribir a Git, solo actualizar en memoria
    argocd-image-updater.argoproj.io/write-back-method: argocd
```

```yaml
# Opción B: Si usas write-back, deshabilitar selfHeal para esa app
spec:
  syncPolicy:
    automated:
      prune: true
      selfHeal: false  # ⚠️ Deshabilitar para evitar loop
```

---

## Antipatrón 3: No ignorar campos autogenerados

### ❌ INCORRECTO
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
spec:
  # Sin ignoreDifferences
  syncPolicy:
    automated:
      selfHeal: true
```

### Problema:
```
1. ArgoCD despliega Application
2. Kubernetes agrega campos automáticos:
   - metadata.resourceVersion
   - metadata.generation
   - metadata.uid
   - metadata.managedFields
   - status.*
3. ArgoCD compara con Git (que no tiene estos campos)
4. ArgoCD ve "diferencia"
5. ArgoCD intenta sincronizar
6. Kubernetes regenera campos
7. LOOP de sincronización infinito
```

### ✅ CORRECTO
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
spec:
  syncPolicy:
    automated:
      selfHeal: true
  # CRÍTICO: Ignorar campos autogenerados
  ignoreDifferences:
    - group: argoproj.io
      kind: Application
      jsonPointers:
        - /status
        - /metadata/resourceVersion
        - /metadata/generation
        - /metadata/uid
        - /metadata/managedFields
    - group: "*"
      kind: "*"
      managedFieldsManagers:
        - argocd-controller
        - argocd-application-controller
        - kube-controller-manager
```

---

## Antipatrón 4: Usar webhooks bidireccionales

### ❌ INCORRECTO
```
┌─────────┐     webhook      ┌─────────┐
│   Git   │ ──────────────▶  │ ArgoCD  │
└─────────┘                  └─────────┘
     ▲                            │
     │                            │
     └────── webhook ─────────────┘
       (Kubernetes → Git)
```

### Por qué es malo:
- Rompe el principio de fuente única de verdad
- Crea dependencias cíclicas
- Dificulta el rollback
- Puede causar loops de sincronización

### ✅ CORRECTO
```
┌─────────┐     webhook      ┌─────────┐     apply       ┌────────────┐
│   Git   │ ──────────────▶  │ ArgoCD  │ ─────────────▶  │ Kubernetes │
└─────────┘                  └─────────┘                 └────────────┘
     │
     │  Solo humanos/CI
     │  escriben aquí
     ▼
```

---

## Antipatrón 5: Sincronizar el mismo recurso desde múltiples Applications

### ❌ INCORRECTO
```yaml
# app-1.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
spec:
  source:
    path: apps/shared-config
  destination:
    namespace: apps

# app-2.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
spec:
  source:
    path: apps/shared-config  # ⚠️ MISMO PATH
  destination:
    namespace: apps
```

### Problema:
- Ambas apps intentan gestionar los mismos recursos
- Conflictos de ownership
- Sincronizaciones que se pisan mutuamente
- Estado inconsistente

### ✅ CORRECTO
```yaml
# Usar un único Application por conjunto de recursos
# O usar ApplicationSet para generar apps dinámicamente
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: shared-apps
spec:
  generators:
    - list:
        elements:
          - name: app-1
            namespace: apps-1
          - name: app-2
            namespace: apps-2
  template:
    spec:
      source:
        path: apps/{{name}}
      destination:
        namespace: '{{namespace}}'
```

---

## Antipatrón 6: Almacenar secrets en Git sin encriptar

### ❌ INCORRECTO
```yaml
# secrets.yaml (en Git)
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  password: bXlwYXNzd29yZDEyMw==  # ⚠️ Base64 NO es encriptación
```

### ✅ CORRECTO - Opciones

#### Opción A: Sealed Secrets
```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: db-credentials
spec:
  encryptedData:
    password: AgBy8hCi...  # Encriptado con clave pública
```

#### Opción B: External Secrets
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  secretStoreRef:
    name: vault
  target:
    name: db-credentials
  data:
    - secretKey: password
      remoteRef:
        key: database/creds
```

#### Opción C: SOPS
```yaml
# Encriptado con SOPS
apiVersion: v1
kind: Secret
data:
    password: ENC[AES256_GCM,data:...,type:str]
sops:
    kms: []
    age:
        - recipient: age1...
```

---

## Resumen de Reglas de Oro

| ✅ Hacer | ❌ No hacer |
|----------|-------------|
| Editar manifiestos solo en Git | kubectl edit/apply/patch |
| Un Application por conjunto de recursos | Múltiples apps para mismos recursos |
| Configurar ignoreDifferences | Ignorar loops de sync |
| Flujo Git → ArgoCD → K8s | Flujo bidireccional |
| Encriptar secrets | Secrets en plaintext en Git |
| selfHeal + prune habilitados | Sincronización manual |
