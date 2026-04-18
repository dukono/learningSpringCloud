# 4.9 Eventos y Métricas — Observabilidad completa

← [4.8 Spring Cloud Circuit Breaker — Abstraction Layer](sc-circuitbreaker-sc-abstraction.md) | [Índice](README.md) | [4.10 Integración con Feign](sc-circuitbreaker-feign.md) →

---

## Introducción

Los Circuit Breakers y los demás patrones de Resilience4j generan eventos internos que permiten observar su comportamiento en tiempo de ejecución sin necesidad de polling ni de leer logs: cuando un circuito abre, cuando una llamada es rechazada, cuando Retry agota sus intentos. Además, Resilience4j se integra con Micrometer para publicar métricas en sistemas de monitorización como Prometheus/Grafana, y con Spring Boot Actuator para exponer el estado mediante endpoints HTTP. Sin esta capa de observabilidad, los fallos en los patrones de resiliencia son invisibles hasta que producen incidentes en producción.

> [PREREQUISITO] Requiere `spring-boot-starter-actuator` y `resilience4j-micrometer` (o `resilience4j-spring-boot3` que los incluye) en el classpath para activar métricas y endpoints.

## Sistema de eventos Resilience4j

Cada instancia de CircuitBreaker, Retry, Bulkhead, RateLimiter y TimeLimiter expone un `EventPublisher` propio. Se registran listeners que se ejecutan sincrónicamente cuando ocurre el evento correspondiente.

Los tipos de eventos del CircuitBreaker son:

| Tipo de evento | Descripción |
|----------------|-------------|
| `SUCCESS` | La llamada fue exitosa |
| `ERROR` | La llamada lanzó una excepción registrada como fallo |
| `IGNORED_ERROR` | La llamada lanzó una excepción en `ignore-exceptions` |
| `NOT_PERMITTED` | La llamada fue rechazada (CB en OPEN) |
| `STATE_TRANSITION` | El CB cambió de estado |
| `SLOW_CALL` | La llamada superó `slowCallDurationThreshold` |
| `FAILURE_RATE_EXCEEDED` | La tasa de fallos superó el umbral |
| `SLOW_CALL_RATE_EXCEEDED` | La tasa de llamadas lentas superó el umbral |

## Ejemplo central

El ejemplo muestra el registro de listeners de eventos para CircuitBreaker y Retry, junto con la configuración de métricas y el endpoint de Actuator:

```java
package com.example.observability;

import io.github.resilience4j.circuitbreaker.CircuitBreaker;
import io.github.resilience4j.circuitbreaker.CircuitBreakerRegistry;
import io.github.resilience4j.retry.RetryRegistry;
import org.springframework.boot.ApplicationRunner;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class ResilienceObservabilityConfig {

    @Bean
    public ApplicationRunner registerEventListeners(
            CircuitBreakerRegistry cbRegistry,
            RetryRegistry retryRegistry) {
        return args -> {
            // Registro de listeners sobre la instancia "paymentService"
            CircuitBreaker cb = cbRegistry.circuitBreaker("paymentService");

            // Listener para transiciones de estado
            cb.getEventPublisher().onStateTransition(event -> {
                System.out.printf("[CIRCUIT BREAKER] '%s': %s → %s%n",
                    event.getCircuitBreakerName(),
                    event.getStateTransition().getFromState(),
                    event.getStateTransition().getToState());
            });

            // Listener para llamadas rechazadas (estado OPEN)
            cb.getEventPublisher().onCallNotPermitted(event -> {
                System.out.printf("[CIRCUIT BREAKER] '%s': call not permitted%n",
                    event.getCircuitBreakerName());
            });

            // Listener para fallos individuales
            cb.getEventPublisher().onError(event -> {
                System.out.printf("[CIRCUIT BREAKER] '%s': error after %.2fms — %s%n",
                    event.getCircuitBreakerName(),
                    event.getElapsedDuration().toMillis() * 1.0,
                    event.getThrowable().getMessage());
            });

            // Listener para tasa de fallos superada
            cb.getEventPublisher().onFailureRateExceeded(event -> {
                System.out.printf("[CIRCUIT BREAKER] '%s': failure rate %.2f%%%n",
                    event.getCircuitBreakerName(),
                    event.getFailureRate());
            });

            // Listener de Retry
            retryRegistry.retry("paymentService")
                .getEventPublisher()
                .onRetry(event -> System.out.printf(
                    "[RETRY] '%s': attempt %d, last error: %s%n",
                    event.getName(),
                    event.getNumberOfRetryAttempts(),
                    event.getLastThrowable().getMessage()))
                .onSuccess(event -> System.out.printf(
                    "[RETRY] '%s': succeeded after %d attempts%n",
                    event.getName(),
                    event.getNumberOfRetryAttempts()))
                .onError(event -> System.out.printf(
                    "[RETRY] '%s': all attempts exhausted%n",
                    event.getName()));
        };
    }
}
```

Configuración Actuator para exponer endpoints y health:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, circuitbreakers, circuitbreakerevents, retries, retryevents,
                 bulkheads, ratelimiters
  endpoint:
    health:
      show-details: always
  health:
    circuitbreakers:
      enabled: true        # activa CircuitBreakerHealthIndicator
    retryevents:
      enabled: true

resilience4j:
  circuitbreaker:
    instances:
      paymentService:
        register-health-indicator: true   # incluye en /actuator/health
        sliding-window-size: 10
        failure-rate-threshold: 50
```

## Métricas Micrometer disponibles

Resilience4j publica las siguientes métricas cuando `resilience4j-micrometer` está en classpath. Todas tienen el tag `name` con el nombre de la instancia y `kind` con el tipo de llamada.

| Métrica | Tags clave | Descripción |
|---------|------------|-------------|
| `resilience4j.circuitbreaker.calls` | `name`, `kind` (successful/failed/not_permitted/ignored) | Contador de llamadas por tipo |
| `resilience4j.circuitbreaker.state` | `name`, `state` (closed/open/half_open) | Estado actual (0/1) |
| `resilience4j.circuitbreaker.failure.rate` | `name` | Tasa de fallos actual (0-100) |
| `resilience4j.circuitbreaker.slow.call.rate` | `name` | Tasa de llamadas lentas |
| `resilience4j.retry.calls` | `name`, `kind` (successful_without_retry/successful_with_retry/failed_with_retry/failed_without_retry) | Reintentos por resultado |
| `resilience4j.bulkhead.available.concurrent.calls` | `name` | Permisos libres en SEMAPHORE |
| `resilience4j.ratelimiter.available.permissions` | `name` | Permisos disponibles en el período |

> [EXAMEN] La métrica `resilience4j.circuitbreaker.state` devuelve 1 cuando el estado coincide con el tag `state` y 0 cuando no coincide. Para saber si el CB está abierto, busca `resilience4j.circuitbreaker.state{name="X",state="open"} = 1`.

## Endpoints Actuator de Resilience4j

Los endpoints expuestos cuando `management.endpoints.web.exposure.include` los incluye:

| Endpoint | Descripción |
|----------|-------------|
| `GET /actuator/circuitbreakers` | Estado de todas las instancias |
| `GET /actuator/circuitbreakerevents` | Últimos eventos de todas las instancias |
| `GET /actuator/circuitbreakerevents/{name}` | Eventos de una instancia específica |
| `GET /actuator/retries` | Estado de las instancias Retry |
| `GET /actuator/retryevents` | Últimos eventos de Retry |
| `GET /actuator/bulkheads` | Estado de las instancias Bulkhead |
| `GET /actuator/ratelimiters` | Estado de las instancias RateLimiter |
| `GET /actuator/health` | Incluye estado de CB si `register-health-indicator: true` |

> [CONCEPTO] `CircuitBreakerHealthIndicator` implementa `WritableHealthContributor` (API Spring Boot 3.x). El estado de salud es `UP` cuando el CB está CLOSED, `DOWN` cuando está OPEN, y `UNKNOWN` cuando está HALF_OPEN. Si un CB está DOWN, el endpoint `/actuator/health` devolverá HTTP 503.

## Buenas y malas prácticas

**Buenas prácticas:**
- Registrar listeners `onStateTransition` en todos los CBs de producción para enviar alertas cuando el circuito abre.
- Usar `register-health-indicator: true` selectivamente solo en los CBs de servicios críticos para evitar falsos positivos en `/actuator/health`.
- Configurar retención de eventos (`event-consumer-buffer-size`) en Actuator para poder ver el historial de eventos en `/circuitbreakerevents`.

**Malas prácticas:**
- Registrar listeners que lanzan excepciones: los listeners se ejecutan en el mismo hilo de la llamada y un error en el listener puede afectar la llamada protegida.
- Exponer `/actuator/circuitbreakerevents` sin autenticación en producción: puede revelar información de topología y patrones de fallo.

## Verificación y práctica

> [EXAMEN] 1. ¿Qué dependencia activa la publicación de métricas de Resilience4j en Micrometer?

> [EXAMEN] 2. ¿Qué propiedad YAML activa el `CircuitBreakerHealthIndicator` para una instancia específica?

> [EXAMEN] 3. ¿Cuál es el nombre de la métrica que indica si un CircuitBreaker está actualmente en estado OPEN?

> [EXAMEN] 4. Si `management.health.circuitbreakers.enabled=true` y el CB "paymentService" está en estado OPEN, ¿qué código HTTP devuelve `/actuator/health`?

> [EXAMEN] 5. ¿Qué diferencia hay entre el evento `ERROR` y el evento `IGNORED_ERROR` en el `EventPublisher` del CircuitBreaker?

---

← [4.8 Spring Cloud Circuit Breaker — Abstraction Layer](sc-circuitbreaker-sc-abstraction.md) | [Índice](README.md) | [4.10 Integración con Feign](sc-circuitbreaker-feign.md) →
