# 🌟 Actividad Bono: Explorando el Ecosistema de Crossplane

> **Modalidad:** Exploración autónoma — No hay guía paso a paso.  
> El objetivo es que investigues, decidas y construyas por tu cuenta.

---

## Contexto

Has visto cómo Crossplane puede gestionar una base de datos PostgreSQL de forma declarativa. Pero Crossplane tiene un **ecosistema enorme** de providers que cubren prácticamente cualquier tipo de infraestructura.

En esta actividad bono, **tú eliges** qué provider explorar y construyes una nueva Composición.

---

## 🗺️ Elige tu Provider

Explora el catálogo de providers disponibles en el **Upbound Marketplace**:

🔗 **[marketplace.upbound.io/providers](https://marketplace.upbound.io/providers)**

### Providers sugeridos para esta actividad

| Provider | Casos de uso | Dificultad |
|---|---|---|
| [`provider-helm`](https://marketplace.upbound.io/providers/crossplane-contrib/provider-helm) | Gestionar releases de Helm como recursos Crossplane | ⭐⭐ Media |
| [`provider-kubernetes`](https://marketplace.upbound.io/providers/crossplane-contrib/provider-kubernetes) | Gestionar objetos Kubernetes en otros clústeres | ⭐⭐ Media |
| [`provider-http`](https://marketplace.upbound.io/providers/crossplane-contrib/provider-http) | Llamar APIs HTTP como Managed Resources | ⭐⭐⭐ Alta |
| [`provider-nop`](https://marketplace.upbound.io/providers/crossplane-contrib/provider-nop) | Provider "vacío" para practicar Compositions sin efectos reales | ⭐ Baja |
| [`provider-aws-s3`](https://marketplace.upbound.io/providers/upbound/provider-aws-s3) | Crear buckets S3 *(requiere cuenta AWS)* | ⭐⭐⭐ Alta |
| [`provider-gcp-storage`](https://marketplace.upbound.io/providers/upbound/provider-gcp-storage) | Gestionar Google Cloud Storage *(requiere cuenta GCP)* | ⭐⭐⭐ Alta |

> 💡 **Recomendación para esta PoC local:** Comienza con `provider-helm` o `provider-kubernetes`, ya que no requieren cuentas en la nube y puedes probarlos en tu clúster Kind.

---

## 📋 Actividades Bono

### 🎯 Bono 1: Instalación del Nuevo Provider

**Objetivo:** Instalar el provider que elegiste y verificar sus Managed Resources.

```bash
# Plantilla — reemplaza con el provider que elegiste
kubectl apply -f - <<EOF
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: <nombre-de-tu-provider>
spec:
  package: <package-del-marketplace>
EOF

# Verificar
kubectl get providers
kubectl get crds | grep <grupo-del-provider>
```

Documenta en `poc/04-bonus/instalacion-provider.md`:
1. ¿Qué provider elegiste y por qué?
2. ¿Qué Managed Resources ofrece?
3. ¿Qué campos son requeridos en el `ProviderConfig`?

---

### 🎯 Bono 2: Diseñar una XRD y Composition para el Nuevo Provider

**Objetivo:** Sin guía paso a paso, diseña y construye una abstracción de infraestructura usando tu nuevo provider.

Crea los archivos:
- `poc/04-bonus/xrd-bonus.yaml` — El `CompositeResourceDefinition`
- `poc/04-bonus/composition-bonus.yaml` — La `Composition`
- `poc/04-bonus/claim-bonus.yaml` — Un `Claim` de ejemplo

**Restricciones del diseño:**
- El Claim debe exponer **al menos 2 parámetros** configurables
- Debe usar **al menos un patch** de tipo `FromCompositeFieldPath`
- El XRD debe estar en el grupo `bonus.poc.io/v1alpha1`

**Criterio de éxito:**
```bash
# Este comando debe mostrar Ready: True
kubectl get <nombre-de-tu-claim> -n default
```

---

### 🎯 Bono 3: GitOps para la Actividad Bono

**Objetivo:** Integrar tu nuevo provider con Argo CD.

```bash
# Crea una Application de Argo CD para el directorio bonus
argocd app create bonus-infra \
  --repo https://github.com/TU-USUARIO/TU-REPOSITORIO.git \
  --path poc/04-bonus \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default \
  --sync-policy automated \
  --self-heal \
  --revision main
```

Luego haz un cambio en tu `claim-bonus.yaml` (ej: cambia un parámetro) y observa cómo Argo CD detecta el cambio y lo aplica automáticamente.

Documenta:
1. ¿Cuánto tiempo tardó Argo CD en detectar el cambio?
2. ¿El recurso de tu provider respondió al cambio? ¿Cómo?

---

### 🎯 Bono 4 (Desafío): Composición Multi-Recurso

**Objetivo:** Crear una Composition que materialice **múltiples Managed Resources** relacionados.

Por ejemplo, si usas el `provider-helm`:
- Un `Release` de Helm para la aplicación
- Un `Release` de Helm para su dependencia (Redis, por ejemplo)
- Un `Object` de Kubernetes para el ConfigMap de configuración

O si usas `provider-postgresql`:
- Una `Database`
- Un `Role` con permisos específicos
- Un `Grant` que asigna el Role a la Database

```bash
# Ver los recursos creados por una Composition multi-recurso
kubectl get managed

# Ver el grafo completo de un Claim
kubectl describe <tu-claim> -n default
```

---

## 📝 Entregable Final del Bono

Crea el archivo `poc/04-bonus/REPORTE_BONO.md` con:

### Sección 1: Provider elegido
- Nombre y enlace al Marketplace
- Justificación de la elección
- Lista de Managed Resources explorados

### Sección 2: Arquitectura de la Composición
- Diagrama (puede ser ASCII art) mostrando el Claim → XR → Managed Resources
- Descripción de los parámetros expuestos y por qué los elegiste

### Sección 3: Resultados observados
- Capturas de pantalla o output de comandos que demuestren el funcionamiento
- Estado final: `kubectl get managed`

### Sección 4: Reflexión
Responde (máx. 300 palabras):
> *"¿Cómo cambiaría la forma de trabajar de un equipo de plataforma si toda la infraestructura se gestiona como código en Git con Crossplane y Argo CD? ¿Qué resistencias u obstáculos anticipas en una adopción real?"*

---

## 🏆 Criterios de evaluación del Bono

| Criterio | Puntos |
|---|---|
| Provider instalado y funcional | 20% |
| XRD y Composition correctamente definidos | 30% |
| Claim desplegado con estado `Ready: True` | 20% |
| Integración con Argo CD funcional | 20% |
| Reporte de reflexión con profundidad conceptual | 10% |

---

## 🔗 Recursos adicionales

- 📖 [Upbound Marketplace — Todos los providers](https://marketplace.upbound.io/providers)
- 📖 [Crossplane Composition Functions (feature avanzado)](https://docs.crossplane.io/latest/concepts/composition-functions/)
- 📖 [Crossplane Community Examples](https://github.com/crossplane-contrib/provider-helm)
- 📺 [Crossplane Deep Dive (KubeCon)](https://www.youtube.com/watch?v=yrj4lmScaus)
- 📖 [Argo CD + Crossplane: GitOps para infraestructura](https://blog.upbound.io/argo-cd-crossplane-gitops)
- 📖 [Platform Engineering con Crossplane](https://www.giantswarm.io/blog/what-is-crossplane/)

---

*¡Felicitaciones por completar la PoC de la Sesión 4! 🎉*

*Maestría en Arquitectura de TI — ARTI-4219 | Sesión 4*
