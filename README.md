# 🚀 Sesión 4: Evolucionar la Plataforma con *Everything as Code* y GitOps

**Maestría en Arquitectura de TI — ARTI-4219**

---

## 📋 Descripción

En esta sesión aplicamos los principios de **GitOps** para gestionar infraestructura y aplicaciones de forma declarativa, usando Git como única fuente de verdad (*single source of truth*).

Construiremos una PoC completa que integra dos herramientas clave del ecosistema Cloud-Native:

| Herramienta | Rol en la PoC |
|---|---|
| **Argo CD** | Operador GitOps — sincroniza el estado del clúster con Git |
| **Crossplane** | Control Plane de infraestructura — gestiona recursos como PostgreSQL de forma declarativa |

---

## 🗂️ Estructura del Repositorio

```
.
├── README.md                          # Esta guía
├── poc/
│   ├── 01-crossplane/                 # Manifiestos y actividades Crossplane
│   │   ├── GUIA_CROSSPLANE.md
│   │   ├── composition.yaml           # TODO — lo construyes tú
│   │   ├── claim.yaml                 # TODO — lo construyes tú
│   │   └── examples/
│   ├── 02-argocd/                     # Manifiestos ArgoCD (guiados)
│   │   ├── GUIA_ARGOCD.md
│   │   ├── install/
│   │   │   └── argocd-namespace.yaml
│   │   └── apps/
│   │       └── app-gitops-demo.yaml
│   ├── 03-integracion/                # Integración completa
│   │   └── GUIA_INTEGRACION.md
│   └── 04-bonus/                      # Actividad de bono
│       └── GUIA_BONUS.md
└── app/                               # Aplicación de demostración
    ├── deployment.yaml
    └── service.yaml
```

---

## ✅ Prerequisitos

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

> **Repositorio necesario:** Un repositorio de GitHub **público** (o accesible con token) donde Argo CD pueda leer manifiestos.

---

## 🔁 Flujo General de la PoC

```
┌─────────────────────────────────────────────────────────────┐
│                        DESARROLLADOR                        │
│  git push → GitHub Repo (manifiestos YAML)                  │
└───────────────────────┬─────────────────────────────────────┘
                        │ webhook / polling
                        ▼
┌─────────────────────────────────────────────────────────────┐
│                        ARGO CD                              │
│  Detecta out-of-sync → aplica cambios en el clúster         │
└────────────┬────────────────────────────────────────────────┘
             │ kubectl apply (automático)
             ▼
┌─────────────────────────────────────────────────────────────┐
│                   CLÚSTER KUBERNETES (kind)                  │
│                                                             │
│  ┌──────────────┐        ┌──────────────────────────┐       │
│  │  App Service │        │  Crossplane Operator     │       │
│  │  (deployment)│        │  PostgreSQLInstance CR   │       │
│  └──────────────┘        └──────────────┬───────────┘       │
│                                         │ reconcile         │
│                                         ▼                   │
│                          ┌──────────────────────────┐       │
│                          │  PostgreSQL (en cluster)  │       │
│                          └──────────────────────────┘       │
└─────────────────────────────────────────────────────────────┘
```

---

## 📚 Guías de la PoC

Sigue las guías en orden:

1. **[Fase 1 — Crossplane](poc/01-crossplane/GUIA_CROSSPLANE.md)** *(Conceptual + Actividades TODO)*
2. **[Fase 2 — Argo CD](poc/02-argocd/GUIA_ARGOCD.md)** *(Guía completa paso a paso)*
3. **[Fase 3 — Integración GitOps](poc/03-integracion/GUIA_INTEGRACION.md)** *(Demo end-to-end)*
4. **[Actividad Bono](poc/04-bonus/GUIA_BONUS.md)** *(Exploración autónoma)*

---

## 🎯 Objetivos de Aprendizaje

Al finalizar esta PoC serás capaz de:

- [ ] Explicar el modelo de control de Crossplane y su relación con los Custom Resource Definitions (CRDs)
- [ ] Construir una **Composición** en Crossplane para abstraer la provisión de infraestructura
- [ ] Instalar y configurar Argo CD en un clúster local con Kind
- [ ] Conectar Argo CD a un repositorio Git y configurar la sincronización automática
- [ ] Observar el ciclo completo de GitOps: *commit → detect → sync → reconcile*
- [ ] Evaluar cuándo usar Crossplane vs. otras herramientas de IaC

---

*Maestría en Arquitectura de TI — ARTI-4219 | Sesión 4*
