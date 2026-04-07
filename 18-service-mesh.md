# Parte 18 — Service Mesh y Kubernetes Avanzado

← [Parte 17 — Migración Legacy](./17-migracion-legacy.md) | [Volver al índice](./README.md)

---

## 18.1 El problema que resuelve un Service Mesh

Spring Cloud resuelve los problemas de microservicios **desde el código**: el desarrollador añade dependencias, anotaciones y configuración en cada servicio. Un **service mesh** resuelve los mismos problemas **desde la infraestructura**, sin tocar el código.

```
SIN service mesh — lógica de red en cada servicio:
┌──────────────────────────────────────┐
│  productos-service                   │
│  ├── lógica de negocio               │
│  ├── Resilience4j (circuit breaker)  │
│  ├── Spring Cloud LoadBalancer       │
│  ├── Micrometer Tracing              │
│  └── mTLS manual                     │
└──────────────────────────────────────┘

CON service mesh — lógica de red en el sidecar:
┌─────────────────────────────┐  ┌────────────────────────┐
│  productos-service          │  │  Sidecar proxy (Envoy) │
│  └── solo lógica de negocio │  │  ├── circuit breaker   │
│                             │  │  ├── load balancing    │
└─────────────────────────────┘  │  ├── tracing           │
                                 │  └── mTLS automático   │
                                 └────────────────────────┘
```

---

## 18.2 Arquitectura de Istio

Istio es el service mesh más extendido. Se compone de dos planos:

```
PLANO DE CONTROL (control plane):
┌─────────────────────────────────────────┐
│  istiod                                 │
│  ├── Pilot    → distribuye configuración│
│  ├── Citadel  → gestiona certificados   │
│  └── Galley   → valida configuración    │
└─────────────────────────────────────────┘
              │ configuración
              ↓
PLANO DE DATOS (data plane):
┌─────────────────────────────────────────────────────┐
│  Pod A                     Pod B                    │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────┐ │
│  │  app A   │ │  Envoy   │ │  app B   │ │ Envoy  │ │
│  │ (puerto  │ │ (sidecar)│ │ (puerto  │ │(sidecar│ │
│  │  8080)   │ │          │ │  8080)   │ │        │ │
│  └──────────┘ └──────────┘ └──────────┘ └────────┘ │
│       todo el tráfico pasa por el sidecar           │
└─────────────────────────────────────────────────────┘
```

El sidecar Envoy se inyecta automáticamente en cada Pod al etiquetar el namespace:

```bash
kubectl label namespace default istio-injection=enabled
```

---

## 18.3 Spring Cloud vs Istio: tabla comparativa

| Capacidad | Spring Cloud | Istio |
|-----------|-------------|-------|
| Circuit Breaker | Resilience4j (código) | VirtualService + DestinationRule |
| Load Balancing | Spring Cloud LoadBalancer | Envoy automático |
| Tracing | Micrometer + Zipkin | Envoy → Jaeger/Zipkin automático |
| mTLS | Manual (configuración) | Automático entre todos los pods |
| Retries | `@Retry` Resilience4j | `VirtualService.retries` |
| Rate Limiting | Gateway + Redis | EnvoyFilter / RateLimitService |
| Service Discovery | Eureka / Consul | Kubernetes DNS nativo |
| Configuración | Config Server | ConfigMap + Secrets |
| Lenguaje | Solo JVM | Agnóstico (cualquier lenguaje) |
| Curva de aprendizaje | Media (Spring) | Alta (Kubernetes + CRDs) |

---

## 18.4 Cuándo usar Spring Cloud vs cuándo usar Istio

### Usar Spring Cloud cuando:
- El equipo conoce Spring y Java bien
- El entorno **no** es Kubernetes (VMs, bare metal)
- Necesitas lógica de resiliencia específica del negocio (fallbacks con datos reales)
- El sistema tiene menos de ~10 microservicios
- Quieres control fino sobre el comportamiento desde el código

### Usar Istio (o Linkerd) cuando:
- **Todo** el sistema ya corre en Kubernetes
- Tienes microservicios en **múltiples lenguajes** (no solo Java)
- El equipo de plataforma/SRE gestiona la infraestructura separado del equipo de desarrollo
- Necesitas mTLS automático entre todos los servicios
- La observabilidad centralizada (sin tocar código) es prioritaria

### El escenario más común en la práctica:
```
Migración gradual:
  Fase 1: Spring Cloud completo (sin Kubernetes)
  Fase 2: Spring Cloud + Kubernetes (Service Discovery via K8s, Config via ConfigMap)
  Fase 3: Istio reemplaza Ribbon/LoadBalancer y Sleuth/Tracing
  Fase 4: Istio gestiona resiliencia básica, Resilience4j solo para fallbacks de negocio
```

---

## 18.5 Spring Cloud en Kubernetes: configuración recomendada

### Reemplazar Eureka por Kubernetes Service Discovery

```xml
<!-- En lugar de Eureka -->
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
      config:
        enabled: true   # leer ConfigMaps como Config Server
      secrets:
        enabled: true   # leer Secrets de Kubernetes
```

```yaml
# Kubernetes Service — el DNS interno actúa como discovery
# productos-service en namespace "default" es accesible como:
# http://productos-service.default.svc.cluster.local:8080
```

### Reemplazar Config Server por ConfigMap

```yaml
# ConfigMap en Kubernetes
apiVersion: v1
kind: ConfigMap
metadata:
  name: productos-service   # debe coincidir con spring.application.name
  namespace: default
data:
  application.yml: |
    spring:
      datasource:
        url: jdbc:postgresql://postgres:5432/productos
    server:
      port: 8081
```

```java
// Spring Cloud Kubernetes lo inyecta automáticamente — sin Config Server
// Los cambios en el ConfigMap pueden recargar el contexto con:
@RefreshScope  // misma anotación que con Config Server
```

### RBAC — permisos para leer ConfigMaps y Secrets

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: spring-cloud-k8s-role
rules:
  - apiGroups: [""]
    resources: ["configmaps", "secrets", "pods", "services", "endpoints"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: spring-cloud-k8s-binding
subjects:
  - kind: ServiceAccount
    name: default
roleRef:
  kind: Role
  name: spring-cloud-k8s-role
  apiGroup: rbac.authorization.k8s.io
```

---

## 18.6 Circuit Breaker con Istio (sin código)

```yaml
# DestinationRule — configuración de resiliencia en Istio
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: productos-service-dr
spec:
  host: productos-service
  trafficPolicy:
    outlierDetection:
      # Circuit breaker: expulsar instancias problemáticas
      consecutiveGatewayErrors: 5    # tras 5 errores 5xx consecutivos
      interval: 10s                  # ventana de evaluación
      baseEjectionTime: 30s          # tiempo de expulsión
      maxEjectionPercent: 50         # máximo % de instancias expulsadas
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 50
        maxRequestsPerConnection: 10
```

```yaml
# VirtualService — retries y timeouts
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: productos-service-vs
spec:
  hosts:
    - productos-service
  http:
    - timeout: 3s
      retries:
        attempts: 3
        perTryTimeout: 1s
        retryOn: "gateway-error,connect-failure,retriable-4xx"
      route:
        - destination:
            host: productos-service
```

---

## 18.7 mTLS automático con Istio

Sin Istio, configurar mTLS entre microservicios requiere gestionar certificados manualmente. Con Istio, es una línea:

```yaml
# Activar mTLS estricto en todo el namespace
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: default
spec:
  mtls:
    mode: STRICT   # rechaza tráfico no cifrado entre pods
```

```yaml
# Autorización: solo pedidos-service puede llamar a productos-service
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: productos-policy
  namespace: default
spec:
  selector:
    matchLabels:
      app: productos-service
  rules:
    - from:
        - source:
            principals: ["cluster.local/ns/default/sa/pedidos-service"]
      to:
        - operation:
            methods: ["GET"]
            paths: ["/productos/*"]
```

---

## 18.8 Observabilidad con Istio: sin tocar el código

Istio inyecta headers de tracing automáticamente (`x-b3-traceid`, `x-b3-spanid`). Solo se necesita **propagarlos** en el código cuando se hacen llamadas HTTP manuales (con Feign o RestTemplate, la propagación es automática).

```yaml
# Integración con Prometheus — Istio expone métricas automáticamente
# No necesitas micrometer-registry-prometheus en cada servicio

# Métricas disponibles sin código:
# istio_requests_total
# istio_request_duration_milliseconds
# istio_tcp_connections_opened_total
```

```yaml
# Grafana — dashboards preconfigurados para Istio
# kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/grafana.yaml
# kubectl apply -f .../prometheus.yaml
# kubectl apply -f .../jaeger.yaml
# kubectl apply -f .../kiali.yaml  ← visualización del grafo de servicios
```

---

## 18.9 Resumen de decisión

```
¿Tu infraestructura es Kubernetes?
        │
       NO → Usa Spring Cloud completo
        │
       SÍ
        │
¿Tienes servicios en múltiples lenguajes?
        │
       SÍ → Istio (o Linkerd) desde el principio
        │
       NO (todo Java/Spring)
        │
¿Tienes equipo de plataforma dedicado?
        │
       NO → Spring Cloud + spring-cloud-kubernetes
            (sin Eureka, sin Config Server)
        │
       SÍ → Migración gradual a Istio
            Spring Cloud solo para fallbacks de negocio
            Istio para resiliencia, tracing y seguridad
```

> `[EXAMEN]` Spring Cloud y un service mesh no son mutuamente excluyentes. Lo habitual en empresas es usar **ambos**: Istio para resiliencia de red e infraestructura, Resilience4j para fallbacks con lógica de negocio (datos cacheados, respuestas degradadas).

---

← [Parte 17 — Migración Legacy](./17-migracion-legacy.md) | [Volver al índice](./README.md)
