# 🧩 Fase 1: Crossplane — Infraestructura como Código en Kubernetes

> **Modalidad de esta fase:** Conceptual + Actividades TODO  
> El objetivo no es solo ejecutar comandos, sino **entender** por qué Crossplane representa un cambio de paradigma en la gestión de infraestructura.

---

## 1.1 ¿Qué es Crossplane?

Crossplane es un **operador de Kubernetes** de código abierto (CNCF) que extiende el API de Kubernetes para gestionar recursos de infraestructura externos (bases de datos, colas de mensajes, redes en la nube, etc.) de la misma manera que gestionamos Pods o Deployments.

La idea central es poderosa: **el plano de control de Kubernetes se convierte en el plano de control de toda tu infraestructura**.

```
┌──────────────────────────────────────────────────────────────────┐
│                    Kubernetes API Server                         │
│                                                                  │
│  Recursos nativos:          Recursos Crossplane (CRDs):          │
│  ┌─────────────┐            ┌────────────────────────────┐       │
│  │ Pod         │            │ PostgreSQLInstance (Claim)  │       │
│  │ Deployment  │            │ XPostgreSQLInstance (XR)    │       │
│  │ Service     │            │ Composition                 │       │
│  │ ConfigMap   │            │ Provider                    │       │
│  └─────────────┘            └────────────────────────────┘       │
└──────────────────────────────────────────────────────────────────┘
```

### Conceptos clave que debes dominar:

| Concepto | Descripción | Analogía |
|---|---|---|
| **Provider** | Plugin que sabe cómo hablar con un sistema externo (AWS, GCP, PostgreSQL) | Controlador de dispositivo (driver) |
| **Managed Resource (MR)** | Representación 1:1 de un recurso externo en Kubernetes | Un Pod que "vive" en AWS |
| **Composite Resource (XR)** | Agrupación de Managed Resources que forman una unidad lógica | Una "clase" que une DB + red + permisos |
| **Composition** | Plantilla que define cómo se ensamblan los MRs para crear un XR | Un "molde" de infraestructura |
| **Claim** | Lo que pide un equipo de desarrollo para obtener infraestructura | Un ticket de solicitud de recursos |
| **CompositeResourceDefinition (XRD)** | Define el esquema del XR y los Claims | Un CRD para tus propios CRDs |

### 📐 Arquitectura: del Claim al recurso real

```
Equipo Dev          Equipo Platform          Crossplane Operator
    │                    │                         │
    │ kubectl apply      │                         │
    │ claim.yaml ──────► │ XRD (valida)            │
    │                    │       │                  │
    │                    │  Composition ──────────► │ Managed Resources
    │                    │  (plantilla)             │ (PostgreSQL DB)
    │                    │                         │
    │◄──────────────────────────────────────────── │ Estado: Ready
    │  Credenciales en Secret
```

### 🔗 Referencias esenciales

- 📖 [Documentación oficial de Crossplane](https://docs.crossplane.io/latest/)
- 📖 [Conceptos: Composite Resources](https://docs.crossplane.io/latest/concepts/composite-resources/)
- 📖 [Conceptos: Compositions](https://docs.crossplane.io/latest/concepts/compositions/)
- 📖 [Provider PostgreSQL en Upbound Marketplace](https://marketplace.upbound.io/providers/tages/provider-postgresql/v0.1.0?tab=managedResources)
- 📖 [CNCF Crossplane Project](https://www.cncf.io/projects/crossplane/)
- 📺 [GitOps con Crossplane (CNCF Webinar)](https://www.youtube.com/watch?v=n8KjVmuHm7A)

---

## 1.2 Instalación de Crossplane

Esta parte **sí es guiada**. Sigue los pasos para tener Crossplane corriendo.

### Paso 1: Crear el clúster Kind

```bash
# Crear un clúster Kind con nombre gitops-poc
kind create cluster --name gitops-poc

# Verificar que el contexto apunta al nuevo clúster
kubectl config current-context
# Debe mostrar: kind-gitops-poc
```

### Paso 2: Instalar Crossplane con Helm

```bash
# Agregar el repositorio de Helm de Crossplane
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update

# Instalar Crossplane en el namespace crossplane-system
helm install crossplane \
  crossplane-stable/crossplane \
  --namespace crossplane-system \
  --create-namespace \
  --wait

# Verificar la instalación
kubectl get pods -n crossplane-system
```

**Salida esperada:**
```
NAME                                       READY   STATUS    RESTARTS
crossplane-xxxx-xxxxx                      1/1     Running   0
crossplane-rbac-manager-xxxx-xxxxx         1/1     Running   0
```

```bash
# Verificar los CRDs instalados por Crossplane
kubectl get crds | grep crossplane
```

### Paso 3: Instalar el Provider de PostgreSQL

El provider que usaremos es [`tages/provider-postgresql`](https://marketplace.upbound.io/providers/tages/provider-postgresql/v0.1.0).

```bash
# Aplicar el manifiesto del Provider
kubectl apply -f - <<EOF
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-postgresql
spec:
  package: xpkg.upbound.io/tages/provider-postgresql:v0.1.0
EOF

# Esperar a que el provider esté listo (puede tomar 1-2 min)
kubectl get providers --watch
```

**Salida esperada cuando esté listo:**
```
NAME                    INSTALLED   HEALTHY   PACKAGE                                           AGE
provider-postgresql     True        True      xpkg.upbound.io/tages/provider-postgresql:v0.1.0  2m
```

```bash
# Verificar los CRDs que instaló el provider
kubectl get crds | grep postgresql
```

> 💡 **¿Por qué aparecen CRDs nuevos?**  
> Cuando instalas un Provider, Crossplane descarga e instala los Custom Resource Definitions que ese provider soporta. Cada CRD representa un tipo de recurso que puedes gestionar (ej: `databases.postgresql.sql.crossplane.io`).

### Paso 4: Desplegar PostgreSQL en el clúster para el Provider

El provider-postgresql necesita conectarse a una instancia de PostgreSQL. Vamos a desplegarla directamente en el clúster Kind.

```bash
# Instalar PostgreSQL usando Helm
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

helm install postgresql bitnami/postgresql \
  --namespace postgresql \
  --create-namespace \
  --set auth.postgresPassword=postgres123 \
  --set auth.username=crossplane \
  --set auth.password=crossplane123 \
  --set auth.database=crossplanedb \
  --wait

# Verificar que PostgreSQL está corriendo
kubectl get pods -n postgresql
```

```bash
# Crear el Secret con las credenciales de conexión
kubectl create secret generic postgresql-credentials \
  --namespace crossplane-system \
  --from-literal=password=postgres123

# Verificar el secret
kubectl get secret postgresql-credentials -n crossplane-system
```

### Paso 5: Configurar el ProviderConfig

```bash
kubectl apply -f - <<EOF
apiVersion: postgresql.sql.crossplane.io/v1alpha1
kind: ProviderConfig
metadata:
  name: postgresql-config
spec:
  credentials:
    source: PostgreSQLConnectionString
    connectionStringSecretRef:
      namespace: crossplane-system
      name: postgresql-credentials
      key: connectionString
EOF
```

> ⚠️ **Nota:** La configuración exacta del ProviderConfig depende de la versión del provider. Consulta la [documentación del provider en Upbound](https://marketplace.upbound.io/providers/tages/provider-postgresql/v0.1.0?tab=managedResources) para ver la estructura exacta.

---

## 1.3 Conceptos Avanzados: Compositions y XRDs

Antes de las actividades, comprende bien estos dos conceptos.

### CompositeResourceDefinition (XRD)

Un XRD es como un `CRD` para tus propios recursos compuestos. Define:
- El **nombre** del Composite Resource y del Claim
- El **schema** (qué campos acepta, cuáles son requeridos)
- Los **grupos de conexión** que se expondrán a los Claims

```yaml
# Ejemplo conceptual de un XRD
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xpostgresqlinstances.database.empresa.io
spec:
  group: database.empresa.io
  names:
    kind: XPostgreSQLInstance         # Nombre del Composite Resource
    plural: xpostgresqlinstances
  claimNames:
    kind: PostgreSQLInstance          # Nombre del Claim (lo que usa Dev)
    plural: postgresqlinstances
  versions:
    - name: v1alpha1
      served: true
      referenceable: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                parameters:
                  type: object
                  properties:
                    storageGB:           # Campo que el Dev puede configurar
                      type: integer
                  required: ["storageGB"]
```

### Composition

Una Composition define cómo se **materializa** el XRD en recursos reales (Managed Resources):

```yaml
# Ejemplo conceptual de una Composition
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: postgresql-composition
spec:
  compositeTypeRef:
    apiVersion: database.empresa.io/v1alpha1
    kind: XPostgreSQLInstance
  resources:
    - name: postgresql-database
      base:
        apiVersion: postgresql.sql.crossplane.io/v1alpha1
        kind: Database
        spec:
          forProvider:
            # Configuración del recurso real
          providerConfigRef:
            name: postgresql-config
      patches:
        # Copiar valores del Claim al Managed Resource
        - type: FromCompositeFieldPath
          fromFieldPath: "spec.parameters.storageGB"
          toFieldPath: "spec.forProvider.storageGB"
```

### 🔗 Referencias para profundizar:

- 📖 [Crossplane Compositions — Documentación Oficial](https://docs.crossplane.io/latest/concepts/compositions/)
- 📖 [Crossplane XRDs — Documentación Oficial](https://docs.crossplane.io/latest/concepts/composite-resource-definitions/)
- 📖 [Patches y Transforms en Compositions](https://docs.crossplane.io/latest/concepts/patch-and-transform/)
- 📖 [Managed Resources del provider-postgresql](https://marketplace.upbound.io/providers/tages/provider-postgresql/v0.1.0?tab=managedResources)

---

## 1.4 🎯 Actividades TODO

> Estas actividades requieren que leas la documentación, analices los ejemplos y construyas los manifiestos. **No hay una solución única** — el proceso de razonamiento es el objetivo.

---

### ✏️ TODO 1: Análisis del Provider PostgreSQL

**Objetivo:** Entender qué Managed Resources ofrece el provider antes de usarlo.

Ve a la [página del provider en Upbound Marketplace](https://marketplace.upbound.io/providers/tages/provider-postgresql/v0.1.0?tab=managedResources).

Responde en un archivo `poc/01-crossplane/analisis-provider.md`:

1. ¿Qué **Managed Resources** ofrece este provider? Lista cada uno con una descripción breve de su propósito.
2. Para el recurso `Database`: ¿Qué campos son **requeridos** en `spec.forProvider`? ¿Cuáles son opcionales?
3. ¿Qué diferencia hay entre un recurso `Database` y un `Role` en este provider?
4. ¿Qué información necesita el `ProviderConfig` para conectarse a PostgreSQL?

---

### ✏️ TODO 2: Diseñar el CompositeResourceDefinition (XRD)

**Objetivo:** Definir la API que usarán los equipos de desarrollo para solicitar una base de datos PostgreSQL.

Crea el archivo `poc/01-crossplane/xrd.yaml` con un `CompositeResourceDefinition` que:

- Defina un Composite Resource llamado `XPostgreSQLInstance` en el grupo `database.poc.io/v1alpha1`
- Defina un Claim llamado `PostgreSQLInstance`
- Exponga **al menos 2 parámetros** configurables por el equipo de desarrollo (ej: `dbName`, `owner`)
- Use un schema OpenAPI v3 válido

**Guías de ayuda:**
- 📖 [Guía oficial de XRDs](https://docs.crossplane.io/latest/concepts/composite-resource-definitions/)
- 📖 [Ejemplo de XRD en la comunidad](https://github.com/crossplane/crossplane/tree/main/docs/guides)

> 💭 **Pregunta de reflexión:** ¿Por qué separamos el XRD de la Composition? ¿Qué ventaja arquitectónica tiene esta separación para un equipo de plataforma?

---

### ✏️ TODO 3: Construir la Composition

**Objetivo:** Implementar la plantilla que materializa el XRD en recursos PostgreSQL reales.

Crea el archivo `poc/01-crossplane/composition.yaml` con una `Composition` que:

- Referencie el XRD creado en el TODO 2
- Cree al menos un `Database` (Managed Resource del provider)
- Use **al menos un patch** para pasar un parámetro del Claim al Managed Resource
- Referencie el `ProviderConfig` llamado `postgresql-config`

**Guías de ayuda:**
- 📖 [Compositions — Documentación](https://docs.crossplane.io/latest/concepts/compositions/)
- 📖 [Patches disponibles](https://docs.crossplane.io/latest/concepts/patch-and-transform/)

> 💭 **Pregunta de reflexión:** ¿Qué ocurre si modificas la Composition después de haber creado Claims? ¿Se actualizan automáticamente los recursos existentes?

---

### ✏️ TODO 4: Crear un Claim y observar la reconciliación

**Objetivo:** Simular ser un equipo de desarrollo que solicita una base de datos.

Crea el archivo `poc/01-crossplane/claim.yaml` con un `PostgreSQLInstance` (Claim) que:

- Use la API definida en tu XRD
- Especifique valores para los parámetros que definiste
- Sea desplegado en el namespace `default`

Luego aplícalo:

```bash
kubectl apply -f poc/01-crossplane/claim.yaml

# Observar el estado de reconciliación
kubectl get postgresqlinstances -n default
kubectl describe postgresqlinstance <nombre> -n default

# Ver los eventos de Crossplane
kubectl get events -n crossplane-system --sort-by='.lastTimestamp'
```

Responde en un comentario al final del `claim.yaml`:

1. ¿Cuánto tiempo tardó en pasar al estado `Ready: True`?
2. ¿Qué Managed Resources se crearon como consecuencia de aplicar el Claim?
3. ¿Dónde están las credenciales de la base de datos creada?

---

### ✏️ TODO 5 (Reflexión): Crossplane vs. Terraform

Escribe en `poc/01-crossplane/reflexion-crossplane-vs-terraform.md` una comparación breve (máx. 400 palabras) respondiendo:

1. **Modelo de operación:** ¿Cómo difiere el ciclo de reconciliación de Crossplane del ciclo `plan/apply` de Terraform?
2. **Drift detection:** ¿Cómo detecta y corrige cada herramienta el drift (desviación del estado deseado)?
3. **Integración con Kubernetes:** ¿Por qué es ventajoso (o no) gestionar infraestructura desde dentro de Kubernetes?
4. **¿Cuándo usarías cada uno?** Da un ejemplo concreto para cada herramienta.

---

*Continúa con → [Fase 2: Argo CD](../02-argocd/GUIA_ARGOCD.md)*
