# 🔁 Fase 3: Demo de Integración GitOps End-to-End

> **Objetivo:** Observar el ciclo completo de GitOps:  
> `git push` → Argo CD detecta el cambio → sincroniza → Crossplane reconcilia

---

## 3.1 Estado actual del entorno

Antes de iniciar la demo, verifica que todo esté en su lugar:

```bash
# 1. Verificar que Crossplane está running
kubectl get pods -n crossplane-system
echo "✅ Crossplane OK"

# 2. Verificar que ArgoCD está running
kubectl get pods -n argocd
echo "✅ Argo CD OK"

# 3. Verificar que PostgreSQL está running
kubectl get pods -n postgresql
echo "✅ PostgreSQL OK"

# 4. Verificar las Applications de Argo CD
argocd app list
echo "✅ Applications OK"

# 5. Ver la versión actual de la imagen de la app
kubectl get deployment gitops-demo-app -n default \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
echo ""
echo "✅ App version OK"
```

**Estado esperado:**
```
NAME              CLUSTER                        NAMESPACE  PROJECT  STATUS  HEALTH
gitops-demo       https://kubernetes.default.svc default    default  Synced  Healthy
crossplane-infra  https://kubernetes.default.svc default    default  Synced  Healthy
```

---

## 3.2 Demo 1: Actualización de Imagen (App GitOps)

Esta demo muestra el flujo de actualización de una aplicación sin tocar el clúster directamente.

### 🎬 Abrir terminales de observación

Antes de hacer el cambio, abre **dos terminales adicionales** para observar en tiempo real:

**Terminal 2 — Observar el estado de Argo CD:**
```bash
watch -n 2 argocd app get gitops-demo
```

**Terminal 3 — Observar el Deployment:**
```bash
kubectl rollout status deployment/gitops-demo-app -n default --watch
```

### 🚀 El cambio: actualizar la versión de nginx

En tu **Terminal 1**, edita el archivo `app/deployment.yaml`:

```bash
# Opción 1: Usar sed para cambiar la versión
sed -i '' 's/nginx:1.24.0/nginx:1.25.4/g' app/deployment.yaml

# Verificar el cambio
grep "image:" app/deployment.yaml
```

Deberías ver: `image: nginx:1.25.4`

### 📤 Push a Git

```bash
git add app/deployment.yaml
git commit -m "feat: upgrade nginx from 1.24.0 to 1.25.4

Actualizando la imagen de nginx a la última versión estable.
Este commit disparará la sincronización automática de Argo CD."

git push origin main
```

### 👀 Observar la sincronización

Observa en **Terminal 2** cómo el estado cambia:

```
1. Synced → OutOfSync    (Argo CD detectó diferencia con Git)
2. OutOfSync → Syncing   (Argo CD está aplicando el cambio)
3. Syncing → Synced      (El clúster ya refleja Git)
```

> ⏱️ **Tiempo esperado:** Argo CD hace polling cada 3 minutos por defecto. Si tienes webhooks configurados, el cambio se detecta en segundos.

### Forzar sincronización inmediata (para la demo)

```bash
# Si no quieres esperar el polling, fuerza la sincronización
argocd app sync gitops-demo
```

### Verificar el resultado

```bash
# Confirmar la nueva versión
kubectl get deployment gitops-demo-app -n default \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
# Debe mostrar: nginx:1.25.4

# Ver el historial de despliegues
argocd app history gitops-demo
```

```
ID  DATE                REVISION
0   2026-07-25 17:00    main (abc1234) - add gitops demo app manifests
1   2026-07-25 17:30    main (def5678) - upgrade nginx from 1.24.0 to 1.25.4
```

---

## 3.3 Demo 2: Self-Healing (Argo CD revierte cambios no autorizados)

Esta demo muestra una de las características más poderosas de GitOps: **la resistencia a cambios manuales**.

### El escenario

Simula que alguien "arregló" el clúster manualmente (lo que en GitOps se llama un **configuration drift**):

```bash
# Cambiar la imagen DIRECTAMENTE en el clúster (sin pasar por Git)
kubectl set image deployment/gitops-demo-app \
  gitops-demo=nginx:1.20.0 \
  -n default

# Verificar el cambio manual
kubectl get deployment gitops-demo-app -n default \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
# Muestra: nginx:1.20.0 ← imagen "no autorizada"
```

### Observar la auto-corrección

```bash
# Observar en tiempo real
watch -n 2 "kubectl get deployment gitops-demo-app -n default -o jsonpath='{.spec.template.spec.containers[0].image}'"
```

Argo CD detectará el drift (la imagen `nginx:1.20.0` no está en Git) y **automáticamente revertirá** al estado de Git (`nginx:1.25.4`).

> ⏱️ Esto ocurre en segundos cuando `selfHeal: true` está activado.

### Verificar la restauración

```bash
kubectl get deployment gitops-demo-app -n default \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
# Debe volver a: nginx:1.25.4
```

---

## 3.4 Demo 3: GitOps para Infraestructura (Crossplane + Argo CD)

Esta demo muestra cómo Argo CD gestiona los recursos de Crossplane desde Git.

### Prerequisito

Asegúrate de haber completado las actividades TODO de Crossplane y tener los archivos:
- `poc/01-crossplane/xrd.yaml`
- `poc/01-crossplane/composition.yaml`
- `poc/01-crossplane/claim.yaml`

### Despliegue via GitOps

```bash
# 1. Agregar los manifiestos de Crossplane a Git
git add poc/01-crossplane/xrd.yaml
git add poc/01-crossplane/composition.yaml
git add poc/01-crossplane/claim.yaml
git commit -m "feat: add crossplane postgresql composition and claim

- Define XRD PostgreSQLInstance
- Crea Composition que provisiona DB en el clúster
- Agrega Claim de ejemplo para el equipo de desarrollo"

git push origin main
```

```bash
# 2. Forzar sincronización de la Application de infraestructura
argocd app sync crossplane-infra

# 3. Verificar que Argo CD aplicó los recursos
argocd app get crossplane-infra
```

### Observar la reconciliación de Crossplane

```bash
# Ver el estado del Claim
kubectl get postgresqlinstances -n default

# Observar los eventos de Crossplane
kubectl get events -n crossplane-system \
  --sort-by='.lastTimestamp' | tail -20

# Ver los Managed Resources creados
kubectl get managed
```

### El flujo completo

```
git push claim.yaml
        │
        ▼
  GitHub Repo
        │  Argo CD polling (3min) o webhook
        ▼
  Argo CD detecta: OutOfSync
        │  sync automático
        ▼
  kubectl apply claim.yaml
        │
        ▼
  Kubernetes API Server
        │  Crossplane controller loop
        ▼
  Crossplane crea Managed Resources
        │  provider reconcile
        ▼
  PostgreSQL DB creada ✅
```

---

## 3.5 Demo 4: Rollback via Git

GitOps permite revertir cambios simplemente revirtiendo el commit de Git.

```bash
# Ver el historial de Argo CD
argocd app history gitops-demo

# Opción 1: Rollback via Argo CD (revertir al despliegue anterior)
argocd app rollback gitops-demo 0

# Opción 2: Rollback via Git (la forma "GitOps pura")
git revert HEAD
git push origin main
# Argo CD detectará el revert y aplicará el estado anterior
```

---

## 3.6 Resumen de la Demo: ¿Qué observamos?

Completa esta tabla en tu repositorio (`poc/03-integracion/observaciones.md`):

| Evento | Tiempo de detección | Tiempo de aplicación | ¿Manual o automático? |
|---|---|---|---|
| Push de nueva imagen | ~3 min (polling) | ~30 seg | Automático |
| Configuration drift detectado | < 10 seg (self-heal) | < 10 seg | Automático |
| Nuevo Claim de Crossplane | ~3 min (polling) | Variable (DB creation) | Automático |
| Rollback | Inmediato (git revert) | ~30 seg | Semi-automático |

---

## 3.7 Configuración de Webhook (Opcional — Detección Instantánea)

Para eliminar el delay de 3 minutos del polling, configura un webhook en GitHub:

```bash
# Exponer Argo CD externamente (para que GitHub pueda contactarlo)
# En entornos locales, usa ngrok para tunelizar
# brew install ngrok
# ngrok http 8080
```

En GitHub → tu repositorio → **Settings → Webhooks → Add webhook:**

```
Payload URL: https://TU-NGROK-URL/api/webhook
Content type: application/json
Secret: (dejar vacío para desarrollo local)
Events: Just the push event
```

Con el webhook activo, Argo CD detectará cambios **en menos de 1 segundo**.

---

*Continúa con → [Actividad Bono](../04-bonus/GUIA_BONUS.md)*
