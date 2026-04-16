# 9.5 Namespace awareness y RBAC

← [9.4 Secrets PropertySource](sc-kubernetes-secrets.md) | [Índice (README.md)](README.md) | [9.6 KubernetesDiscoveryClient — descubrimiento de servicios](sc-kubernetes-discovery.md) →

---

## Introducción

Spring Cloud Kubernetes necesita llamar a la API de Kubernetes para leer ConfigMaps, Secrets, Services y Endpoints. Estas llamadas se realizan con la identidad del `ServiceAccount` asignado al pod, y Kubernetes verifica cada una contra las reglas RBAC definidas para ese ServiceAccount. Si el RBAC no está configurado correctamente, el resultado más habitual es un error `403 Forbidden` en runtime —a menudo sin un mensaje de error claro en los logs de la aplicación— que impide que SCK lea la configuración o descubra servicios. Este nodo explica cómo determinar el namespace de operación (que puede ser multi-namespace), cómo construir el `Role`/`ClusterRole` mínimo para cada funcionalidad de SCK, y cómo diagnosticar los errores 403. La comprensión de estos conceptos es prerequisito para los nodos de Discovery (9.6–9.8) y Leader Election (9.11), que requieren permisos adicionales.

> **[ADVERTENCIA]** El verbo `update` sobre `configmaps` es necesario para Leader Election (9.11) pero NO para ConfigMap PropertySource (9.2) ni para DiscoveryClient (9.6–9.7). Añadir `update` al Role de una aplicación que no usa Leader Election amplía innecesariamente la superficie de ataque.

## Representación visual

El diagrama muestra cómo SCK determina el namespace de operación y qué permisos RBAC mínimos necesita cada funcionalidad.

```
Kubernetes cluster
│
├── Namespace: produccion
│   │
│   ├── Pod: mi-app
│   │   └── ServiceAccount: mi-app-sa
│   │       │
│   │       ├── RoleBinding → Role "mi-app-config-reader"
│   │       │   Permisos: get/list/watch configmaps, secrets
│   │       │
│   │       └── RoleBinding → Role "mi-app-discovery-reader"
│   │           Permisos: get/list/watch services, endpoints, pods
│   │
│   └── /var/run/secrets/kubernetes.io/serviceaccount/namespace
│       └── "produccion"  ← SCK autodetecta namespace del pod
│
└── Namespace: otro-namespace
    └── (SCK necesita ClusterRoleBinding para acceso multi-namespace)

Detección de namespace (orden de precedencia):
  1. spring.cloud.kubernetes.client.namespace (override explícito)
  2. Fichero /var/run/secrets/kubernetes.io/serviceaccount/namespace
  3. "default" (fallback)
```

> **[CONCEPTO]** Kubernetes inyecta automáticamente el namespace del pod en el fichero `/var/run/secrets/kubernetes.io/serviceaccount/namespace` dentro del contenedor. SCK lo lee para determinar en qué namespace buscar ConfigMaps y Services, sin necesidad de configurarlo explícitamente en `application.yml`.

## Ejemplo central

El siguiente ejemplo muestra la configuración RBAC completa para una aplicación que usa ConfigMap PropertySource, Secrets PropertySource y DiscoveryClient en el namespace `produccion`, con comentarios sobre qué permiso necesita cada funcionalidad.

**ServiceAccount**:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: mi-app-sa
  namespace: produccion
```

**Role con permisos mínimos** (namespace-scoped, preferido sobre ClusterRole):

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: mi-app-role
  namespace: produccion
rules:
  # ConfigMap PropertySource (9.2) y recarga dinámica en modo polling
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "watch"]
  # Secrets PropertySource (9.4) — solo si secrets.enabled=true
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list", "watch"]
  # DiscoveryClient (9.6) y LoadBalancer (9.8)
  - apiGroups: [""]
    resources: ["services", "endpoints", "pods"]
    verbs: ["get", "list", "watch"]
  # Leader Election (9.11) — solo si se usa el módulo de leader election
  # - apiGroups: [""]
  #   resources: ["configmaps"]
  #   verbs: ["get", "list", "watch", "create", "update", "patch"]
```

**RoleBinding** (vincula el Role al ServiceAccount):

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: mi-app-rolebinding
  namespace: produccion
subjects:
  - kind: ServiceAccount
    name: mi-app-sa
    namespace: produccion
roleRef:
  kind: Role
  name: mi-app-role
  apiGroup: rbac.authorization.k8s.io
```

**Deployment referenciando el ServiceAccount**:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mi-app
  namespace: produccion
spec:
  replicas: 2
  selector:
    matchLabels:
      app: mi-app
  template:
    metadata:
      labels:
        app: mi-app
    spec:
      serviceAccountName: mi-app-sa   # OBLIGATORIO — sin esto usa default SA
      containers:
        - name: mi-app
          image: mi-empresa/mi-app:latest
```

**Configuración multi-namespace en application.yml**:

```yaml
spring:
  cloud:
    kubernetes:
      client:
        namespace: produccion          # Override explícito del namespace
      config:
        namespace: produccion
        sources:
          - name: mi-app-config
            namespace: produccion
          - name: shared-config
            namespace: configuracion-global  # Segundo namespace
      discovery:
        all-namespaces: false
        namespaces:
          - produccion
          - integracion
```

**ClusterRole para acceso multi-namespace** (alternativa para `all-namespaces: true`):

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: mi-app-cluster-reader
rules:
  - apiGroups: [""]
    resources: ["services", "endpoints", "pods", "configmaps"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: mi-app-cluster-rolebinding
subjects:
  - kind: ServiceAccount
    name: mi-app-sa
    namespace: produccion
roleRef:
  kind: ClusterRole
  name: mi-app-cluster-reader
  apiGroup: rbac.authorization.k8s.io
```

**Diagnóstico de error 403** — verificar permisos desde la CLI:

```bash
# Verificar si el ServiceAccount puede listar configmaps
kubectl auth can-i list configmaps \
  --as=system:serviceaccount:produccion:mi-app-sa \
  -n produccion

# Verificar permiso específico de watch
kubectl auth can-i watch services \
  --as=system:serviceaccount:produccion:mi-app-sa \
  -n produccion

# Ver todos los permisos del ServiceAccount
kubectl auth can-i --list \
  --as=system:serviceaccount:produccion:mi-app-sa \
  -n produccion
```

> **[EXAMEN]** Pregunta frecuente: "¿Cuándo usar `Role`/`RoleBinding` vs `ClusterRole`/`ClusterRoleBinding` para SCK?" Regla general: usar `Role` (namespace-scoped) siempre que la aplicación solo necesite acceso en su propio namespace. Usar `ClusterRole` solo cuando `discovery.all-namespaces=true` o cuando se leen ConfigMaps de múltiples namespaces. El scope mínimo es el scope correcto.

## Tabla de elementos clave

| Elemento | Scope | Descripción |
|---|---|---|
| `ServiceAccount` | Namespace | Identidad del pod para llamadas a la Kubernetes API |
| `Role` | Namespace | Conjunto de permisos limitados a un namespace |
| `ClusterRole` | Cluster | Conjunto de permisos válidos en todos los namespaces |
| `RoleBinding` | Namespace | Vincula un Role o ClusterRole a un ServiceAccount en un namespace |
| `ClusterRoleBinding` | Cluster | Vincula un ClusterRole a un ServiceAccount en todos los namespaces |
| `spring.cloud.kubernetes.client.namespace` | — | Override del namespace de operación (precede al autodetectado) |
| `spring.cloud.kubernetes.discovery.all-namespaces` | — | Descubrimiento en todos los namespaces (requiere ClusterRole) |
| `spring.cloud.kubernetes.discovery.namespaces` | — | Lista explícita de namespaces para el descubrimiento |

**Tabla de permisos mínimos por funcionalidad de SCK:**

| Funcionalidad | Recurso K8s | Verbos necesarios |
|---|---|---|
| ConfigMap PropertySource (lectura) | `configmaps` | `get`, `list`, `watch` |
| Secrets PropertySource | `secrets` | `get`, `list`, `watch` |
| DiscoveryClient | `services`, `endpoints`, `pods` | `get`, `list`, `watch` |
| Health Indicator | `configmaps` | `get` |
| Leader Election | `configmaps` | `get`, `list`, `watch`, `create`, `update`, `patch` |

## Buenas y malas prácticas

**Hacer:**

- Usar `Role` + `RoleBinding` namespace-scoped en lugar de `ClusterRole` + `ClusterRoleBinding` siempre que la aplicación opere en un único namespace: reduce el radio de impacto si el ServiceAccount es comprometido.
- Crear un `ServiceAccount` dedicado por cada microservicio: evita que un Service Account compartido entre varios servicios otorgue permisos excesivos si uno de ellos necesita accesos adicionales.
- Verificar permisos con `kubectl auth can-i` antes de desplegar: es mucho más rápido que desplegar y leer los logs de error 403.
- Documentar explícitamente en el `Role` (como comentarios YAML) qué funcionalidad de SCK justifica cada permiso: facilita las auditorías de seguridad y las revisiones de RBAC.

**Evitar:**

- Usar el ServiceAccount `default` del namespace para las aplicaciones SCK: el ServiceAccount `default` es compartido por todos los pods del namespace que no declaran uno explícito, y concederle permisos amplios afecta a todos esos pods.
- Añadir `update`/`create`/`patch` sobre `configmaps` al Role de aplicaciones que no usan Leader Election: estos verbos permiten modificar la configuración de otros servicios del namespace.
- Usar `ClusterRoleBinding` con `all-namespaces=true` en entornos multi-tenant: un bug en la aplicación podría listar Services de namespaces de otros equipos o entornos.
- Ignorar el error 403 en producción reiniciando el pod: el 403 persistirá hasta que se corrija el RBAC; el pod no obtendrá los datos de configuración correctos aunque reinicie.

---

← [9.4 Secrets PropertySource](sc-kubernetes-secrets.md) | [Índice (README.md)](README.md) | [9.6 KubernetesDiscoveryClient — descubrimiento de servicios](sc-kubernetes-discovery.md) →
