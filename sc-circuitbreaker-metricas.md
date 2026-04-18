# 4.12 Métricas avanzadas y Health Indicators

← [4.11 Integración con Spring Cloud Gateway](sc-circuitbreaker-gateway.md) | [Índice](README.md) | [4.13 Testing con Resilience4j](sc-circuitbreaker-testing.md) →

---

## Introducción

Las métricas básicas de Resilience4j cubiertas en [4.9 Eventos y Métricas](sc-circuitbreaker-eventos.md) son suficientes para la observabilidad del día a día. Este fichero profundiza en la integración avanzada con Prometheus, las convenciones de naming de métricas, las métricas de Bulkhead y RateLimiter, los Health Indicators para Kubernetes readiness/liveness probes, y la estructura de dashboards Grafana para Resilience4j. Este nivel de detalle es relevante para la configuración de producción y para algunas preguntas de examen sobre la API de Spring Boot 3.x.

> [PREREQUISITO] Requiere `resilience4j-spring-boot3` (o `resilience4j-micrometer`) y `spring-boot-starter-actuator` en el classpath. Las métricas solo aparecen si existe al menos una instancia de cada patrón configurada.

## Métricas completas por patrón

Resilience4j publica un conjunto específico de métricas para cada patrón. Todas siguen la convención de Micrometer con tags para distinguir instancias.

Las métricas del CircuitBreaker y sus tags disponibles:

| Métrica | Tipo | Tags | Descripción |
|---------|------|------|-------------|
| `resilience4j.circuitbreaker.calls` | Counter | `name`, `kind` | Llamadas por tipo: successful, failed, not_permitted, ignored |
| `resilience4j.circuitbreaker.state` | Gauge | `name`, `state` | 1 si el CB está en ese estado, 0 si no |
| `resilience4j.circuitbreaker.failure.rate` | Gauge | `name` | Tasa de fallos actual (%) en la sliding window |
| `resilience4j.circuitbreaker.slow.call.rate` | Gauge | `name` | Tasa de llamadas lentas actual (%) |
| `resilience4j.circuitbreaker.buffered.calls` | Gauge | `name`, `kind` | Llamadas en la sliding window |
| `resilience4j.circuitbreaker.max.buffered.calls` | Gauge | `name` | Tamaño de la sliding window |

Métricas del Retry:

| Métrica | Tags | Descripción |
|---------|------|-------------|
| `resilience4j.retry.calls` | `name`, `kind` | Kind: successful_without_retry, successful_with_retry, failed_with_retry, failed_without_retry |

Métricas del Bulkhead:

| Métrica | Tags | Descripción |
|---------|------|-------------|
| `resilience4j.bulkhead.available.concurrent.calls` | `name` | Permisos libres (SEMAPHORE) |
| `resilience4j.bulkhead.max.allowed.concurrent.calls` | `name` | Máximo de llamadas concurrentes (SEMAPHORE) |
| `resilience4j.thread.pool.bulkhead.available.queue.capacity` | `name` | Capacidad libre de la queue (THREADPOOL) |
| `resilience4j.thread.pool.bulkhead.active.thread.count` | `name` | Threads activos en el pool (THREADPOOL) |

Métricas del RateLimiter:

| Métrica | Tags | Descripción |
|---------|------|-------------|
| `resilience4j.ratelimiter.available.permissions` | `name` | Permisos disponibles en el período actual |
| `resilience4j.ratelimiter.waiting.threads` | `name` | Threads esperando permiso |

## Ejemplo central

El ejemplo muestra la configuración completa para Prometheus, el scraping de métricas y la configuración de Health Indicators:

```xml
<!-- pom.xml — dependencias necesarias -->
<!-- resilience4j-spring-boot3 incluye spring-boot-starter-actuator y micrometer -->
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot3</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

```yaml
# application.yml — configuración de métricas y health indicators
management:
  endpoints:
    web:
      exposure:
        include: health, metrics, prometheus, circuitbreakers, circuitbreakerevents
  endpoint:
    health:
      show-details: always
      show-components: always
    prometheus:
      enabled: true
  metrics:
    tags:
      application: ${spring.application.name}  # tag global en todas las métricas
  health:
    circuitbreakers:
      enabled: true   # activa CircuitBreakerHealthIndicator global
    retryevents:
      enabled: true

resilience4j:
  circuitbreaker:
    configs:
      default:
        register-health-indicator: true    # añade al /actuator/health
        sliding-window-size: 10
        failure-rate-threshold: 50
    instances:
      paymentService:
        base-config: default
      inventoryService:
        base-config: default
        register-health-indicator: false   # excluir del health check
```

Consultas PromQL útiles para Grafana:

```promql
# Tasa de fallos del CircuitBreaker "paymentService" en los últimos 5 minutos
rate(resilience4j_circuitbreaker_calls_total{name="paymentService",kind="failed"}[5m])
/
rate(resilience4j_circuitbreaker_calls_total{name="paymentService",kind=~"failed|successful"}[5m])
* 100

# Estado actual del CircuitBreaker (1 = OPEN)
resilience4j_circuitbreaker_state{name="paymentService", state="open"}

# Llamadas rechazadas por CB abierto (alertar si > 0 durante más de 30s)
increase(resilience4j_circuitbreaker_calls_total{kind="not_permitted"}[1m])

# Permisos disponibles en RateLimiter (alertar si cerca de 0)
resilience4j_ratelimiter_available_permissions{name="externalApi"}
```

## CircuitBreakerHealthIndicator y Spring Boot 3.x

`CircuitBreakerHealthIndicator` implementa `WritableHealthContributor`, que es la API de Spring Boot 3.x para contribuyentes de salud jerárquicos. Esta API permite que el estado de un CB se muestre como sub-componente del health endpoint.

```
GET /actuator/health

{
  "status": "DOWN",
  "components": {
    "circuitBreakers": {
      "status": "DOWN",
      "details": {
        "paymentService": {
          "status": "DOWN",
          "details": {
            "failureRate": "52.0%",
            "failureRateThreshold": "50.0%",
            "slowCallRate": "0.0%",
            "slowCallRateThreshold": "100.0%",
            "bufferedCalls": 10,
            "failedCalls": 6,
            "notPermittedCalls": 0,
            "state": "OPEN"
          }
        },
        "inventoryService": {
          "status": "UP",
          ...
        }
      }
    }
  }
}
```

La tabla de mapeo estado CB → estado Health:

| Estado Circuit Breaker | Estado Health | Código HTTP /actuator/health |
|----------------------|---------------|------------------------------|
| CLOSED | UP | 200 |
| HALF_OPEN | UNKNOWN | 200 |
| OPEN | DOWN | 503 |
| DISABLED | UP | 200 |
| FORCED_OPEN | DOWN | 503 |

> [EXAMEN] Si cualquier CircuitBreaker con `register-health-indicator: true` está en estado OPEN, el endpoint `/actuator/health` devolverá HTTP 503 (no 200). Esto puede afectar a los health checks de Kubernetes si no se configura correctamente qué CBs contribuyen al health.

## Configuración para Kubernetes liveness/readiness

En Kubernetes, es común configurar el health endpoint de forma diferente para liveness y readiness probes para evitar que un CB abierto cause reinicios innecesarios:

```yaml
management:
  endpoint:
    health:
      group:
        liveness:
          include: ping, diskSpace    # liveness: solo checks básicos, NO circuit breakers
        readiness:
          include: circuitBreakers    # readiness: si CB abierto, pod no recibe tráfico
```

Con esta configuración:
- `/actuator/health/liveness` → responde UP siempre que el proceso esté vivo.
- `/actuator/health/readiness` → responde DOWN si algún CB está OPEN, sacando el pod del load balancer.

## Buenas y malas prácticas

**Buenas prácticas:**
- Añadir el tag global `application: ${spring.application.name}` a todas las métricas para filtrar por servicio en Grafana.
- Separar liveness y readiness en Kubernetes para que un CB abierto no cause reinicios.
- Crear alertas sobre `resilience4j_circuitbreaker_state{state="open"} > 0` con un umbral de duración (ej: alerta si OPEN durante más de 2 minutos).

**Malas prácticas:**
- Activar `register-health-indicator: true` en todos los CBs sin separar liveness/readiness: puede causar reinicios en cascada en Kubernetes.
- Exponer `/actuator/prometheus` sin autenticación: las métricas pueden revelar la topología y el comportamiento del sistema.

## Verificación y práctica

> [EXAMEN] 1. ¿Qué valor devuelve la métrica `resilience4j.circuitbreaker.state{name="X",state="open"}` cuando el CB está en estado CLOSED?

> [EXAMEN] 2. ¿Cómo se configura en application.yml que solo `paymentService` contribuya al health check de Kubernetes readiness?

> [EXAMEN] 3. ¿Qué implementación de Spring Boot 3.x usa `CircuitBreakerHealthIndicator` y qué permite esta interfaz?

> [EXAMEN] 4. ¿Cuál es la diferencia entre las métricas `resilience4j.circuitbreaker.calls{kind="failed"}` y `resilience4j.circuitbreaker.calls{kind="not_permitted"}`?

> [EXAMEN] 5. ¿Qué métrica de Micrometer indica cuántos threads están esperando un permiso de RateLimiter?

---

← [4.11 Integración con Spring Cloud Gateway](sc-circuitbreaker-gateway.md) | [Índice](README.md) | [4.13 Testing con Resilience4j](sc-circuitbreaker-testing.md) →
