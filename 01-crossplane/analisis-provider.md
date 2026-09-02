# Análisis del Provider PostgreSQL

## Provider: `tages/provider-postgresql` v0.1.0

Paquete OCI: `xpkg.upbound.io/tages/provider-postgresql:v0.1.0`

Provider generado con **Upjet** a partir del `terraform-provider-postgresql`. Al instalarse
registra 17 CRDs en el clúster, todos de **scope `Cluster`** (dato relevante: por eso el XRD
de esta PoC usa `scope: LegacyCluster`, ya que un XR namespaced de Crossplane v2 no puede
componer Managed Resources cluster-scoped).

> Verificación en el clúster: `kubectl get crds | grep postgresql.upbound.io`

---

### 1. Managed Resources disponibles

#### Recursos de datos (`postgresql.postgresql.upbound.io/v1alpha1`)

| Kind | Propósito |
|---|---|
| **Database** | Crea y gestiona una base de datos en el servidor PostgreSQL (`CREATE DATABASE`). |
| **Role** | Crea y gestiona un rol/usuario en el servidor (`CREATE ROLE`), con sus atributos (login, superuser, password, validez…). |
| **Schema** | Crea y gestiona un esquema dentro de una base de datos (`CREATE SCHEMA`). |
| **Extension** | Instala y gestiona una extensión en una base de datos (`CREATE EXTENSION`, ej. `postgis`, `uuid-ossp`). |
| **Function** | Crea y gestiona una función almacenada en la base de datos. |
| **Grant** | Otorga privilegios a un rol sobre objetos de una base de datos/esquema (`GRANT`). |
| **Publication** | Crea y gestiona una publicación para replicación lógica. |
| **Subscription** | Crea y gestiona una suscripción a una publicación (replicación lógica). |
| **Server** | Crea y gestiona un *foreign server* (Foreign Data Wrapper). |

#### Recursos con grupo propio

| Kind | apiVersion | Propósito |
|---|---|---|
| **Role** (grant) | `grant.postgresql.upbound.io/v1alpha1` | Gestiona la **pertenencia** de un rol a otros roles (`GRANT rol_a TO rol_b`). Ojo: es un kind distinto del `Role` de datos. |
| **Privileges** | `default.postgresql.upbound.io/v1alpha1` | Gestiona los **privilegios por defecto** (`ALTER DEFAULT PRIVILEGES`) que heredarán los objetos futuros. |
| **Mapping** | `user.postgresql.upbound.io/v1alpha1` | Gestiona un *user mapping* entre un rol local y un foreign server. |
| **ReplicationSlot** | `physical.postgresql.upbound.io/v1alpha1` | Crea y gestiona un slot de replicación **física**. |
| **Slot** | `replication.postgresql.upbound.io/v1alpha1` | Crea y gestiona un slot de replicación (lógica). |

#### Recursos de infraestructura del provider (no son Managed Resources de negocio)

| Kind | apiVersion | Propósito |
|---|---|---|
| **ProviderConfig** | `postgresql.upbound.io/v1beta1` | Configura *cómo* se conecta el provider a PostgreSQL. |
| **ProviderConfigUsage** | `postgresql.upbound.io/v1beta1` | Objeto interno; registra qué MR está usando un ProviderConfig (evita borrarlo mientras esté en uso). |
| **StoreConfig** | `postgresql.upbound.io/v1alpha1` | Configura dónde se almacenan los *connection details* (secretos de conexión). |

**Para esta PoC** solo se usa `Database`, más `ProviderConfig` como configuración de conexión.

---

### 2. Campos requeridos del recurso `Database`

`apiVersion: postgresql.postgresql.upbound.io/v1alpha1` · `kind: Database` · scope `Cluster`

#### Requeridos por el schema

A nivel de `spec`, el único campo obligatorio es **`forProvider`**. Dentro de `spec.forProvider`
el CRD **no marca ningún campo como `required`**, porque Upjet permite que el identificador real
del recurso venga de la anotación `crossplane.io/external-name`.

> Verificado con:
> `kubectl get crd databases.postgresql.postgresql.upbound.io -o jsonpath='{.spec.versions[0].schema.openAPIV3Schema.properties.spec.required}'` → `["forProvider"]`

**En la práctica, para que el recurso funcione hay que aportar:**

| Campo | Por qué es necesario |
|---|---|
| `spec.forProvider.name` (o la anotación `crossplane.io/external-name`) | Es el nombre de la base de datos en PostgreSQL. Sin él, Crossplane usa el `metadata.name` del MR como external-name. |
| `spec.providerConfigRef.name` | Indica qué `ProviderConfig` usar. Tiene `default: {name: default}`, así que si tu ProviderConfig se llama distinto (aquí `postgresql-config`) es obligatorio declararlo. |

#### Opcionales de `spec.forProvider`

| Campo | Tipo | Descripción | Default |
|---|---|---|---|
| `name` | string | Nombre de la base de datos. Debe ser único en la instancia. | external-name |
| `owner` | string | Rol propietario de la base de datos. | usuario que ejecuta el comando |
| `encoding` | string | Codificación de caracteres (`UTF8`, `SQL_ASCII`…). | la del template |
| `lcCollate` | string | Orden de colación (`LC_COLLATE`). | la del template |
| `lcCtype` | string | Clasificación de caracteres (`LC_CTYPE`). | la del template |
| `template` | string | Base de datos plantilla desde la que clonar. Cambiarlo **recrea** la BD. | `template0` |
| `tablespaceName` | string | Tablespace asociado a la base de datos. | el del template |
| `connectionLimit` | number | Conexiones concurrentes máximas. `-1` = sin límite. | `-1` |
| `allowConnections` | boolean | Si es `false`, nadie puede conectarse. | `true` |
| `isTemplate` | boolean | Marca la BD como clonable por cualquier usuario con `CREATEDB`. | `false` |

#### Otros campos de `spec` (comunes a todo MR de Crossplane)

`deletionPolicy` (`Delete`/`Orphan`), `managementPolicies`, `initProvider`,
`providerConfigRef`, `writeConnectionSecretToRef`, `publishConnectionDetailsTo`.

---

### 3. Información requerida por el `ProviderConfig`

`apiVersion: postgresql.upbound.io/v1beta1` · `kind: ProviderConfig`

El schema exige **`spec.credentials`**, y dentro de él **`source`** es el único campo obligatorio.
`source` es un enum **case-sensitive**: `None | Secret | InjectedIdentity | Environment | Filesystem`.

Según la fuente elegida se acompaña de:

| `source` | Campo adicional | Campos requeridos |
|---|---|---|
| `Secret` | `secretRef` | `name`, `namespace`, `key` (los tres obligatorios) |
| `Environment` | `env` | `name` |
| `Filesystem` | `fs` | `path` |
| `None` / `InjectedIdentity` | — | — |

En esta PoC se usa `source: Secret`, apuntando al Secret `postgresql-credentials` del namespace
`crossplane-system`, clave `connection`.

**Contenido de la credencial** (JSON dentro de la clave del Secret) — es lo que el provider
necesita realmente para abrir la conexión:

```json
{
  "host": "postgresql.postgresql.svc.cluster.local",
  "port": "5432",
  "username": "postgres",
  "password": "platform123",
  "database": "postgres",
  "sslmode": "disable"
}
```

| Dato | Para qué |
|---|---|
| `host` | DNS del servicio de PostgreSQL dentro del clúster. |
| `port` | Puerto de escucha (5432). |
| `username` / `password` | Credenciales de un rol con permisos de `CREATEDB` / superusuario. |
| `database` | Base de datos de *bootstrap* contra la que se conecta para ejecutar `CREATE DATABASE`. |
| `sslmode` | `disable` porque el PostgreSQL de la PoC no expone TLS. |

Manifiesto resultante (`provider-config.yaml`):

```yaml
apiVersion: postgresql.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: postgresql-config
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: postgresql-credentials
      key: connection
```

El nombre `postgresql-config` es el que cada Managed Resource referencia mediante
`spec.providerConfigRef.name` — en esta PoC, desde el `base` del `Database` en la Composition.
