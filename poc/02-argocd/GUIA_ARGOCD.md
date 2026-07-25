# 🔄 Fase 2: Argo CD — Operador GitOps para Kubernetes

> **Modalidad de esta fase:** Completamente guiada, paso a paso.  
> Instalaremos, configuraremos y operaremos Argo CD desde cero.

---

## 2.1 ¿Qué es Argo CD?

Argo CD es un **operador GitOps declarativo para Kubernetes**. Monitoriza continuamente un repositorio Git y asegura que el estado del clúster refleja exactamente lo que está en Git.

### El loop de reconciliación de Argo CD

```
                    ┌─────────────────────────┐
                    │       GIT REPO          │
                    │  (estado deseado)       │
                    └────────────┬────────────┘
                                 │ polling cada 3min
                                 │ (o webhook inmediato)
                                 ▼
┌──────────────────────────────────────────────────────┐
│                    ARGO CD                           │
│                                                      │
│  1. Compara Git ↔ Clúster                           │
│  2. Si hay diferencia → estado: OutOfSync            │
│  3. Si sync policy = auto → aplica cambios           │
│  4. Si sync policy = manual → espera aprobación      │
└──────────────────────────────────────────────────────┘
                                 │
                                 ▼ kubectl apply
                    ┌─────────────────────────┐
                    │   CLÚSTER KUBERNETES    │
                    │  (estado real/actual)   │
                    └─────────────────────────┘
```

### Conceptos clave de Argo CD

| Concepto | Descripción |
|---|---|
| **Application** | Objeto de Argo CD que conecta un repositorio Git con un destino en Kubernetes |
| **Sync** | Proceso de aplicar el estado de Git al clúster |
| **OutOfSync** | Estado cuando Git y el clúster difieren |
| **Synced** | Estado cuando Git y el clúster son idénticos |
| **Self-healing** | Capacidad de revertir cambios manuales no autorizados en el clúster |
| **App of Apps** | Patrón para gestionar múltiples Applications desde una sola |

---

## 2.2 Instalación de Argo CD

### Paso 1: Crear el namespace

```bash
# Crear el namespace dedicado para Argo CD
kubectl create namespace argocd

# Verificar
kubectl get namespace argocd
```

### Paso 2: Instalar Argo CD

```bash
# Instalar los manifiestos oficiales de Argo CD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

> ⏳ La instalación tarda aproximadamente 2-3 minutos. Mientras tanto, puedes observar cómo se crean los pods:

```bash
# Observar la creación de pods en tiempo real
kubectl get pods -n argocd --watch
```

**Espera hasta que TODOS los pods estén `Running`:**

```
NAME                                                READY   STATUS    RESTARTS
argocd-application-controller-0                     1/1     Running   0
argocd-applicationset-controller-xxxx               1/1     Running   0
argocd-dex-server-xxxx                              1/1     Running   0
argocd-notifications-controller-xxxx                1/1     Running   0
argocd-redis-xxxx                                   1/1     Running   0
argocd-repo-server-xxxx                             1/1     Running   0
argocd-server-xxxx                                  1/1     Running   0
```

Presiona `Ctrl+C` para salir del watch.

### Paso 3: Verificar los CRDs de Argo CD

```bash
# Argo CD instala sus propios CRDs
kubectl get crds | grep argoproj
```

Deberías ver:
```
applications.argoproj.io
applicationsets.argoproj.io
appprojects.argoproj.io
```

---

## 2.3 Acceder a la UI de Argo CD

Argo CD incluye una interfaz web completa. Para acceder desde un clúster Kind necesitamos hacer port-forwarding.

### Paso 1: Exponer el servidor de Argo CD

```bash
# En una terminal separada (déjala corriendo)
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

> 💡 Deja esta terminal abierta durante toda la PoC. Puedes abrir una nueva terminal para los siguientes pasos.

### Paso 2: Obtener la contraseña inicial

```bash
# La contraseña inicial del usuario 'admin' está en un Secret
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

Guarda esta contraseña — la necesitarás para el login.

### Paso 3: Acceder a la UI

Abre tu navegador en: **https://localhost:8080**

> ⚠️ El navegador mostrará una advertencia de certificado SSL. Haz clic en "Avanzado" → "Continuar a localhost". Esto es normal en entornos locales.

**Credenciales:**
- Usuario: `admin`
- Contraseña: (la que obtuviste en el paso anterior)

### Paso 4: Instalar el CLI de Argo CD (opcional pero recomendado)

```bash
# macOS con Homebrew
brew install argocd

# Verificar instalación
argocd version --client
```

### Paso 5: Login desde el CLI

```bash
# Login al servidor (acepta el certificado con --insecure)
argocd login localhost:8080 \
  --username admin \
  --password $(kubectl -n argocd get secret argocd-initial-admin-secret \
    -o jsonpath="{.data.password}" | base64 -d) \
  --insecure

# Verificar el login
argocd account list
```

---

## 2.4 Conectar Argo CD a tu Repositorio Git

### Paso 1: Preparar el repositorio

En **tu repositorio de GitHub**, asegúrate de que existe la siguiente estructura de directorios:

```
tu-repo/
├── README.md
├── poc/
│   ├── 01-crossplane/
│   └── 02-argocd/
│       └── apps/
│           └── app-gitops-demo.yaml  ← Argo CD leerá esto
└── app/
    ├── deployment.yaml               ← La app que Argo CD desplegará
    └── service.yaml
```

### Paso 2: Crear los manifiestos de la aplicación de demo

Crea en tu repositorio el archivo `app/deployment.yaml`:

```yaml
# app/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gitops-demo-app
  namespace: default
  labels:
    app: gitops-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: gitops-demo
  template:
    metadata:
      labels:
        app: gitops-demo
    spec:
      containers:
        - name: gitops-demo
          image: nginx:1.24.0      # ← Esta versión la cambiaremos en la demo
          ports:
            - containerPort: 80
          resources:
            requests:
              memory: "64Mi"
              cpu: "50m"
            limits:
              memory: "128Mi"
              cpu: "100m"
```

Crea el archivo `app/service.yaml`:

```yaml
# app/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: gitops-demo-svc
  namespace: default
spec:
  selector:
    app: gitops-demo
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```

```bash
# Commitea y pushea estos archivos
git add app/
git commit -m "feat: add gitops demo app manifests"
git push origin main
```

### Paso 3: Registrar el repositorio en Argo CD

**Opción A — Repositorio Público (más sencillo):**

```bash
# Para repositorios públicos, no necesitas credenciales
argocd repo add https://github.com/TU-USUARIO/TU-REPOSITORIO.git

# Verificar
argocd repo list
```

**Opción B — Repositorio Privado con Token:**

```bash
# Primero crea un Personal Access Token en GitHub:
# GitHub → Settings → Developer Settings → Personal Access Tokens

argocd repo add https://github.com/TU-USUARIO/TU-REPOSITORIO.git \
  --username TU-USUARIO-GITHUB \
  --password TU-PERSONAL-ACCESS-TOKEN

# Verificar
argocd repo list
```

**Verificar en la UI:**  
Ve a **Settings → Repositories** en la UI de Argo CD. Deberías ver tu repositorio con estado `✅ Successful`.

---

## 2.5 Crear la Application de Argo CD

Una `Application` en Argo CD es el objeto que conecta:
- **Fuente**: qué repositorio Git, qué rama, qué directorio
- **Destino**: qué clúster Kubernetes, qué namespace

### Paso 1: Crear la Application via CLI

```bash
argocd app create gitops-demo \
  --repo https://github.com/TU-USUARIO/TU-REPOSITORIO.git \
  --path app \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default \
  --sync-policy automated \
  --auto-prune \
  --self-heal \
  --revision main

# Verificar la creación
argocd app list
```

### Paso 2: Inspeccionar el estado inicial

```bash
# Ver el estado detallado de la Application
argocd app get gitops-demo
```

**Salida esperada:**
```
Name:               argocd/gitops-demo
Project:            default
Server:             https://kubernetes.default.svc
Namespace:          default
URL:                https://localhost:8080/applications/gitops-demo
Repo:               https://github.com/TU-USUARIO/TU-REPOSITORIO.git
Target:             main
Path:               app
SyncWindow:         Sync Allowed
Sync Policy:        Automated (Prune)
Sync Status:        Synced to main (abc1234)
Health Status:      Healthy
```

### Paso 3: Observar en la UI

En la UI de Argo CD (https://localhost:8080):

1. Haz clic en la aplicación **gitops-demo**
2. Observa el **grafo de recursos**: verás el Deployment y el Service
3. Verifica que el estado sea **Synced** y **Healthy**

---

## 2.6 Crear la Application para Crossplane (GitOps para Infraestructura)

Ahora haremos que Argo CD también gestione los recursos de Crossplane desde Git.

```bash
argocd app create crossplane-infra \
  --repo https://github.com/TU-USUARIO/TU-REPOSITORIO.git \
  --path poc/01-crossplane \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default \
  --sync-policy automated \
  --self-heal \
  --revision main

# Verificar
argocd app list
```

> 💡 **Esto es GitOps puro:** Argo CD gestionará los Claims de Crossplane desde Git. Cuando hagas push de tu `claim.yaml`, Argo CD lo aplicará automáticamente, y Crossplane creará la base de datos.

---

## 2.7 Verificar la instalación completa

```bash
# Ver todas las Applications
argocd app list

# Ver pods de la app desplegada por Argo CD
kubectl get pods -n default

# Ver el deployment
kubectl get deployment gitops-demo-app -n default

# Ver la imagen actual del deployment
kubectl get deployment gitops-demo-app -n default \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

**Resultado esperado:** `nginx:1.24.0`

---

## 2.8 Manifiesto de la Application como YAML (GitOps sobre GitOps)

El patrón más avanzado es guardar el objeto `Application` de Argo CD **también en Git**. Así, incluso la configuración de Argo CD está versionada.

Crea el archivo `poc/02-argocd/apps/app-gitops-demo.yaml`:

```yaml
# poc/02-argocd/apps/app-gitops-demo.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: gitops-demo
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/TU-USUARIO/TU-REPOSITORIO.git
    targetRevision: main
    path: app
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true      # Elimina recursos que ya no están en Git
      selfHeal: true   # Revierte cambios manuales en el clúster
    syncOptions:
      - CreateNamespace=true
```

```bash
# Commitear el manifiesto de la Application
git add poc/02-argocd/
git commit -m "feat: add ArgoCD Application manifests to Git"
git push origin main
```

> 💡 Este es el patrón **"App of Apps"**: una Application que gestiona otras Applications. Puedes ver más en la [documentación oficial de App of Apps](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/).

---

## 2.9 Comandos de referencia rápida

```bash
# === Gestión de Applications ===
argocd app list                          # Listar todas las apps
argocd app get <nombre>                  # Ver estado detallado
argocd app sync <nombre>                 # Sincronizar manualmente
argocd app diff <nombre>                 # Ver diferencias Git vs Clúster

# === Historial y rollback ===
argocd app history <nombre>              # Ver historial de deploys
argocd app rollback <nombre> <id>        # Revertir a una versión anterior

# === Gestión del clúster ===
argocd cluster list                      # Ver clústeres registrados
argocd repo list                         # Ver repositorios registrados

# === Diagnóstico ===
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-server
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller
```

---

*Continúa con → [Fase 3: Integración GitOps](../03-integracion/GUIA_INTEGRACION.md)*
