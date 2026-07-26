# Sesión 4: Everything as Code y GitOps

En esta sesión aplicamos los principios de **GitOps** para gestionar infraestructura y aplicaciones de forma declarativa, usando Git como única fuente de verdad.

Construiremos una PoC completa que integra dos herramientas clave:

| Herramienta | Rol en la PoC |
|---|---|
| **Argo CD** | Operador GitOps — sincroniza el estado del clúster con Git |
| **Crossplane** | Control Plane de infraestructura — gestiona recursos como PostgreSQL de forma declarativa |

---

## Requisitos Previos

Antes de comenzar, asegúrate de tener instalado:

```bash
# Verificar kind
kind version        # >= 0.20.0

# Verificar kubectl
kubectl version --client

# Verificar helm
helm version        # >= 3.x

# Verificar git y acceso al repo
git remote -v
```

**Repositorio necesario:** Debes clonar este repositorio de GitHub **público** (o accesible con token) donde Argo CD pueda leer manifiestos.

---

## Guías de la PoC

Sigue las guías en orden:

1. **[Fase 1 — Crossplane](01-crossplane/README.md)**
2. **[Fase 2 — Argo CD](02-argocd/README.md)**
3. **[Fase 3 — Integración GitOps](03-integracion/README.md)**
