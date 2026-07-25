# Reporte Bono: Exploración de Provider Crossplane

> Completa este reporte como entregable de la Actividad Bono
> Ver: poc/04-bonus/GUIA_BONUS.md

---

## Sección 1: Provider elegido

**Nombre del provider:** <!-- ej: provider-helm -->

**Enlace en Upbound Marketplace:** <!-- URL -->

**¿Por qué lo elegiste?**

<!-- Tu justificación aquí -->

**Managed Resources que ofrece:**

| Managed Resource | Propósito |
|---|---|
| ... | ... |

---

## Sección 2: Arquitectura de la Composición

**Diagrama Claim → XR → Managed Resources:**

```
Claim: <nombre>
    │
    ▼
XR: <nombre>
    │
    ├── MR: <recurso 1>
    └── MR: <recurso 2>
```

**Parámetros expuestos en el Claim:**

| Parámetro | Tipo | Descripción | Requerido |
|---|---|---|---|
| ... | ... | ... | Si/No |

---

## Sección 3: Resultados observados

**Output de `kubectl get managed`:**

```
<!-- pega el output aquí -->
```

**Estado del Claim:**

```
<!-- kubectl get <claim> -n default -->
```

---

## Sección 4: Reflexión

<!-- ¿Cómo cambiaría la forma de trabajar de un equipo de plataforma? -->
<!-- ¿Qué resistencias u obstáculos anticipas en una adopción real? -->
<!-- Máximo 300 palabras -->
