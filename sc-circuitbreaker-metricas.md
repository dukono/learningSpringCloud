# 5.11 Métricas de Resilience4j con Micrometer y endpoints Actuator

← [5.10 Anotaciones AOP de Resilience4j y orden de decoradores combinados](sc-circuitbreaker-aop.md) | [Índice](README.md) | [5.12 Health check y eventos CircuitBreakerEvent en producción](sc-circuitbreaker-eventos.md) →

## Introducción

Resilience4j se integra con Micrometer para exportar métricas de todos sus componentes (Circuit Breaker, Retry, Bulkhead, RateLimiter) al sistema de observabilidad de la aplicación. Esta integración es lo que hace que el patrón de resiliencia sea operable en producción: sin métricas, el Circuit Breaker funciona pero es opaco; con métricas, se puede detectar cuándo el circuito está abriendo, con qué frecuencia se reintentan llamadas, y si el bulkhead está rechazando peticiones antes de que un incidente se escale.

Spring Boot Actuator expone endpoints específicos de Resilience4j que complementan las métricas de Micrometer: mientras las métricas son series temporales apropiadas para Prometheus/Grafana, los endpoints de Actuator proporcionan el estado instantáneo de cada instancia y el historial de eventos recientes.

> [PREREQUISITO] Las métricas de Resilience4j requieren `spring-boot-starter-actuator` en el classpath y la dependencia `resilience4j-micrometer` (incluida transitivamente en el starter de Spring Cloud Circuit Breaker). No es necesario código adicional para la exportación.

## Representación visual

El flujo de métricas desde los componentes de Resilience4j hasta el sistema de observabilidad:

```
Resilience4j componentes:
  CircuitBreaker ─┐
  Retry          ─┤  registran métricas en
  Bulkhead       ─┤──────────────────────→ MeterRegistry (Spring Boot)
  RateLimiter    ─┘                              │
                                                  ├──→ /actuator/metrics/resilience4j.*
                                                  ├──→ Prometheus endpoint /actuator/prometheus
                                                  └──→ Grafana, Datadog, etc.

Endpoints específicos de Resilience4j (via Actuator):
  /actuator/circuitbreakers         → estado de todas las instancias CB
  /actuator/circuitbreakerevents    → historial de eventos CB
  /actuator/retries                 → estado de todas las instancias Retry
  /actuator/retryevents             → historial de eventos Retry
```

| Familia de métricas                    | Nombre de la métrica                                          | Tags principales                   |
|----------------------------------------|---------------------------------------------------------------|------------------------------------|
| Circuit Breaker — llamadas             | `resilience4j.circuitbreaker.calls`                           | `name`, `kind` (successful/failed/not_permitted/ignored) |
| Circuit Breaker — estado               | `resilience4j.circuitbreaker.state`                           | `name`, `state` (0=closed,1=open,2=half_open) |
| Circuit Breaker — tasa de fallos       | `resilience4j.circuitbreaker.failure.rate`                    | `name`                             |
| Circuit Breaker — llamadas lentas      | `resilience4j.circuitbreaker.slow.call.rate`                  | `name`                             |
| Retry                                  | `resilience4j.retry.calls`                                    | `name`, `kind` (successful_with_retry/without_retry/failed_with_retry/without_retry) |
| Bulkhead                               | `resilience4j.bulkhead.available.concurrent.calls`            | `name`                             |
| Bulkhead                               | `resilience4j.bulkhead.max.allowed.concurrent.calls`          | `name`                             |
| RateLimiter                            | `resilience4j.ratelimiter.available.permissions`              | `name`                             |
| RateLimiter                            | `resilience4j.ratelimiter.waiting.threads`                    | `name`                             |

## Ejemplo central

Configuración completa con exportación a Prometheus, consulta de métricas por Actuator y una alerta básica con PromQL para detectar Circuit Breakers abiertos.

```xml
<!-- pom.xml -->
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <!-- Exportación a Prometheus -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-registry-prometheus</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-aop</artifactId>
    </dependency>
</dependencies>
```

```yaml
# application.yml — configuración de Actuator y métricas
spring:
  application:
    name: order-service

resilience4j:
  circuitbreaker:
    instances:
      paymentService:
        sliding-window-size: 10
        minimum-number-of-calls: 5
        failure-rate-threshold: 50
        wait-duration-in-open-state: 30s
        automatic-transition-from-open-to-half-open-enabled: true
        # Activar registro de indicadores de salud
        register-health-indicator: true
  retry:
    instances:
      paymentService:
        max-attempts: 3
        wait-duration: 500ms

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus,circuitbreakers,circuitbreakerevents,retries,retryevents
  endpoint:
    health:
      show-details: always
  health:
    circuitbreakers:
      enabled: true
  # Etiquetas comunes a todas las métricas (útil para distinguir instancias en Grafana)
  metrics:
    tags:
      application: ${spring.application.name}
      environment: production
```

```java
// MetricsObserver.java — lectura programática de métricas Resilience4j vía Micrometer
package com.example.orders.observability;

import io.micrometer.core.instrument.Gauge;
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.search.Search;
import org.springframework.stereotype.Component;
import java.util.Optional;

@Component
public class MetricsObserver {

    private final MeterRegistry meterRegistry;

    public MetricsObserver(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }

    /**
     * Consulta programática del estado del Circuit Breaker via Micrometer.
     * Útil para lógica de negocio que necesita saber el estado actual del CB.
     * En producción, preferir las alertas de Prometheus sobre código ad-hoc.
     */
    public double getCircuitBreakerFailureRate(String name) {
        return Optional.ofNullable(
                meterRegistry.find("resilience4j.circuitbreaker.failure.rate")
                        .tag("name", name)
                        .gauge()
        ).map(Gauge::value).orElse(0.0);
    }

    public boolean isCircuitBreakerOpen(String name) {
        // state: 0=closed, 1=open, 2=half_open, 3=disabled, 4=metrics_only
        double state = Optional.ofNullable(
                meterRegistry.find("resilience4j.circuitbreaker.state")
                        .tag("name", name)
                        .tag("state", "open")
                        .gauge()
        ).map(Gauge::value).orElse(0.0);
        return state == 1.0;
    }
}
```

```java
// Uso en un endpoint de diagnóstico (opcional, solo para demostración)
package com.example.orders.web;

import com.example.orders.observability.MetricsObserver;
import org.springframework.web.bind.annotation.*;
import java.util.Map;

@RestController
@RequestMapping("/internal/diagnostics")
public class DiagnosticsController {

    private final MetricsObserver metricsObserver;

    public DiagnosticsController(MetricsObserver metricsObserver) {
        this.metricsObserver = metricsObserver;
    }

    @GetMapping("/circuit-breaker/{name}")
    public Map<String, Object> circuitBreakerStatus(@PathVariable String name) {
        return Map.of(
                "name", name,
                "isOpen", metricsObserver.isCircuitBreakerOpen(name),
                "failureRate", metricsObserver.getCircuitBreakerFailureRate(name)
        );
    }
}
```

```promql
# Ejemplos de alertas PromQL para Grafana/Prometheus

# Alerta: Circuit Breaker abierto
resilience4j_circuitbreaker_state{state="open"} == 1

# Alerta: tasa de fallos superior al 40% durante más de 2 minutos
resilience4j_circuitbreaker_failure_rate > 40

# Dashboard: llamadas exitosas vs fallidas por servicio
sum by (name, kind) (rate(resilience4j_circuitbreaker_calls_seconds_count[1m]))

# Alerta: muchos threads esperando en RateLimiter (saturación de la API externa)
resilience4j_ratelimiter_waiting_threads > 5
```

## Tabla de elementos clave

Configuración de Actuator para exponer los endpoints de Resilience4j.

| Endpoint / Propiedad                                           | Descripción                                                                       |
|----------------------------------------------------------------|-----------------------------------------------------------------------------------|
| `GET /actuator/circuitbreakers`                                | Lista todas las instancias CB con su estado actual y configuración                |
| `GET /actuator/circuitbreakerevents`                           | Historial de eventos de todos los CB (últimos 100 por defecto)                    |
| `GET /actuator/circuitbreakerevents/{name}`                    | Eventos de una instancia CB específica                                            |
| `GET /actuator/retries`                                        | Estado de todas las instancias Retry                                              |
| `GET /actuator/retryevents`                                    | Historial de eventos de Retry                                                     |
| `GET /actuator/metrics/resilience4j.circuitbreaker.calls`     | Métrica de llamadas del CB con todos sus tags                                     |
| `management.health.circuitbreakers.enabled`                    | `true` para incluir CBs en `/actuator/health`                                     |
| `resilience4j.circuitbreaker.instances.[name].register-health-indicator` | `true` para incluir la instancia en el health indicator          |
| `management.endpoints.web.exposure.include`                    | Debe incluir `circuitbreakers,circuitbreakerevents` para exponer los endpoints    |

## Buenas y malas prácticas

**Hacer:**

- Añadir tags de aplicación y entorno a las métricas con `management.metrics.tags.*`. Sin estos tags, las métricas de diferentes servicios o entornos se mezclan en Grafana y es imposible distinguirlos.
- Crear alertas de Prometheus/Grafana sobre `resilience4j.circuitbreaker.state{state="open"}` para recibir notificaciones cuando un circuito abre. Una alerta que se dispara en menos de un minuto tras la apertura del circuito permite reaccionar antes de que el incidente escale.
- Revisar periódicamente el endpoint `/actuator/circuitbreakerevents` durante incidentes. Los eventos incluyen el tipo de error que causó el fallo, lo que acelera el diagnóstico.
- Monitorizar `resilience4j.retry.calls{kind="failed_with_retry"}`: un valor alto indica que los reintentos están fallando sistemáticamente, señal de un problema más profundo que los fallos transitorios.

**Evitar:**

- No exponer los endpoints de Actuator sin autenticación en producción. Los endpoints `/actuator/circuitbreakers` y `/actuator/circuitbreakerevents` pueden revelar información sobre la topología de servicios y los modos de fallo del sistema.
- Evitar leer métricas de Resilience4j desde código de negocio con `MeterRegistry` para tomar decisiones de flujo. El state management del CB es interno a Resilience4j; interferir desde fuera puede causar comportamientos inconsistentes.
- No confiar solo en las métricas de Actuator para el monitoreo en tiempo real. Los endpoints de Actuator son síncronos y muestran el estado en el momento de la consulta; Prometheus con alertas reactivas es más adecuado para producción.

---

← [5.10 Anotaciones AOP de Resilience4j y orden de decoradores combinados](sc-circuitbreaker-aop.md) | [Índice](README.md) | [5.12 Health check y eventos CircuitBreakerEvent en producción](sc-circuitbreaker-eventos.md) →
