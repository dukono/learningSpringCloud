# Parte 11 — Spring Cloud Kubernetes

← [Parte 10 — Seguridad](./10-seguridad.md) | [Volver al índice](./README.md) | Siguiente: [Parte 12 — Proyecto Práctico](./12-proyecto-practico.md) →

---

## 11.1 Spring Cloud en entornos Kubernetes

Kubernetes ya proporciona muchas de las capacidades que Spring Cloud implementa:

| Capacidad | Spring Cloud (standalone) | Kubernetes |
|---|---|---|
| Service Discovery | Eureka Server | DNS interno + Services |
| Configuración | Config Server + Git | ConfigMaps + Secrets |
| Load Balancing | Spring Cloud LoadBalancer | kube-proxy + Service |
| Health checks | Actuator + Eureka heartbeat | Liveness/Readiness probes |
| Escalado | Manual / externo | HPA (Horizontal Pod Autoscaler) |

**Spring Cloud Kubernetes** es el puente que permite usar las APIs de Kubernetes como backend para los componentes de Spring Cloud, eliminando la necesidad de Eureka y Config Server en entornos Kubernetes.

### Cuándo usar Spring Cloud Kubernetes

- La aplicación se desplegará exclusivamente en Kubernetes
- Se quiere aprovechar los mecanismos nativos de K8s (ConfigMaps, Service DNS)
- Se quiere simplificar la arquitectura eliminando Eureka y Config Server

### Cuándo mantener Eureka + Config Server en Kubernetes

- La aplicación se despliega en múltiples entornos (K8s, VMs, on-premise)
- El equipo tiene experiencia con Eureka y no quiere migrar
- Se necesitan características avanzadas de Eureka (self-preservation, metadata)

---

## 11.2 Service Discovery nativo en Kubernetes

### Cómo funciona el discovery en Kubernetes

Kubernetes asigna un **nombre DNS interno** a cada Service. Los pods pueden resolver otros servicios por su nombre:

```
Servicio "pedidos-service" en namespace "miapp":
  → DNS: pedidos-service.miapp.svc.cluster.local
  → o simplemente: pedidos-service (dentro del mismo namespace)
```

### Spring Cloud Kubernetes Discovery

**Spring Cloud Kubernetes** lee el registro de pods de Kubernetes (usando la API de K8s) y lo expone como un `DiscoveryClient` estándar.

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-kubernetes-client-all</artifactId>
</dependency>
```

```yaml
spring:
  cloud:
    kubernetes:
      discovery:
        enabled: true
        # Descubrir servicios de todos los namespaces
        all-namespaces: false
        # Namespace específico
        namespaces:
          - miapp
```

Con esto, `@FeignClient(name = "pedidos-service")` resuelve el service de Kubernetes llamado `pedidos-service`, sin necesidad de Eureka.

### Manifest de Kubernetes para un microservicio

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pedidos-service
  namespace: miapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: pedidos-service
  template:
    metadata:
      labels:
        app: pedidos-service
    spec:
      containers:
        - name: pedidos-service
          image: miempresa/pedidos-service:1.0.0
          ports:
            - containerPort: 8083
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: "prod"
          # Probes de salud — Spring Actuator
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8083
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8083
            initialDelaySeconds: 20
            periodSeconds: 5
---
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: pedidos-service
  namespace: miapp
spec:
  selector:
    app: pedidos-service
  ports:
    - port: 8083
      targetPort: 8083
```

---

## 11.3 ConfigMaps y Secrets como fuente de configuración

**Spring Cloud Kubernetes Config** permite a los microservicios leer su configuración directamente desde **ConfigMaps** y **Secrets** de Kubernetes, sin necesidad de un Config Server externo.

### ConfigMap como application.yml

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: pedidos-service   # debe coincidir con spring.application.name
  namespace: miapp
data:
  application.yaml: |
    pedidos:
      max-items: 1000
      timeout-ms: 2000
    spring:
      datasource:
        url: jdbc:postgresql://db-service:5432/pedidos
```

### Secret para credenciales sensibles

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: pedidos-service-secrets
  namespace: miapp
type: Opaque
data:
  # Los valores deben estar en Base64
  db-password: cGFzc3dvcmQxMjM=   # echo -n "password123" | base64
  jwt-secret: bWktc2VjcmV0by1qd3Q=
```

### Configurar Spring Boot para leer ConfigMap y Secrets

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-kubernetes-client-config</artifactId>
</dependency>
```

```yaml
# application.yml del microservicio
spring:
  config:
    import: "kubernetes:"   # le dice a Spring Boot que lea de K8s
  cloud:
    kubernetes:
      config:
        enabled: true
        name: pedidos-service        # nombre del ConfigMap
        namespace: miapp
        sources:
          - name: pedidos-service    # ConfigMap principal
          - name: comun-config       # ConfigMap compartido
      secrets:
        enabled: true
        name: pedidos-service-secrets
        namespace: miapp
```

### Permisos RBAC necesarios

Los pods necesitan permisos para leer ConfigMaps y Secrets de la API de Kubernetes:

```yaml
# rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pedidos-service-sa
  namespace: miapp
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pedidos-service-role
  namespace: miapp
rules:
  - apiGroups: [""]
    resources: ["configmaps", "secrets", "pods", "services", "endpoints"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pedidos-service-rolebinding
  namespace: miapp
subjects:
  - kind: ServiceAccount
    name: pedidos-service-sa
roleRef:
  kind: Role
  name: pedidos-service-role
  apiGroup: rbac.authorization.k8s.io
```

```yaml
# Asignar la ServiceAccount al Deployment
spec:
  template:
    spec:
      serviceAccountName: pedidos-service-sa
```

### Refresco automático de ConfigMap

Spring Cloud Kubernetes puede detectar cambios en el ConfigMap y refrescar automáticamente la configuración sin reiniciar el pod:

```yaml
spring:
  cloud:
    kubernetes:
      reload:
        enabled: true
        mode: event          # event (webhook) o polling
        period: 15000        # intervalo en modo polling (ms)
        strategy: refresh    # refresh (solo @RefreshScope) o restart_context
```

---

## 11.4 Eureka vs Kubernetes Service Discovery

### Comparativa detallada

| Aspecto | Eureka | Kubernetes Service DNS |
|---|---|---|
| **Registro** | El servicio se registra activamente (heartbeat) | Kubernetes registra el pod automáticamente |
| **Descubrimiento** | El cliente consulta el registry via HTTP | DNS nativo del cluster |
| **Infraestructura extra** | Requiere Eureka Server desplegado | Nativo en K8s, sin componentes extra |
| **Health checks** | Heartbeat cada 30s | Liveness/Readiness probes por kubelet |
| **Metadatos** | Soporta metadatos personalizados ricos | Labels y annotations de K8s |
| **Multi-cloud** | Portátil (funciona fuera de K8s) | Solo en K8s |
| **Latencia descubrimiento** | Hasta 90s para reflejar cambios | Casi inmediato |
| **Curva de aprendizaje** | Baja (solo Spring) | Media (requiere K8s) |

### Decisión recomendada

```
¿Solo vas a desplegar en Kubernetes?
  SÍ → Usa Spring Cloud Kubernetes Discovery (sin Eureka)
  NO → Usa Eureka (funciona en cualquier entorno)

¿Tienes K8s y quieres migrar gradualmente?
  → Usa spring-cloud-kubernetes con Eureka como fallback durante la transición
```

### Configuración de probes en Spring Boot 3

Spring Boot 3 expone endpoints específicos para las probes de Kubernetes:

```yaml
management:
  endpoint:
    health:
      probes:
        enabled: true
  health:
    livenessstate:
      enabled: true
    readinessstate:
      enabled: true
```

```
GET /actuator/health/liveness   → UP si el proceso está vivo
GET /actuator/health/readiness  → UP si el servicio está listo para recibir tráfico
```

El pod se considera **listo** (readiness=UP) solo cuando:
1. El contexto de Spring está completamente cargado
2. La configuración del Config Server se ha cargado (si se usa)
3. La conexión a la base de datos está disponible

---

← [Parte 10 — Seguridad](./10-seguridad.md) | [Volver al índice](./README.md) | Siguiente: [Parte 12 — Proyecto Práctico](./12-proyecto-practico.md) →
