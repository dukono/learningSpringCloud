# 5.12 Health check y eventos CircuitBreakerEvent en producción

← [5.11 Métricas de Resilience4j con Micrometer y endpoints Actuator](sc-circuitbreaker-metricas.md) | [Índice](README.md) | [5.13 Integración del Circuit Breaker con OpenFeign, RestClient y WebClient](sc-circuitbreaker-feign.md) →

## Introducción

Las métricas de Micrometer son el mecanismo pull de observabilidad: el sistema de monitoreo consulta el estado periódicamente. Los eventos `CircuitBreakerEvent` son el mecanismo push: el Circuit Breaker notifica en tiempo real cada transición, fallo o éxito a los suscriptores registrados. Esta diferencia es importante en producción porque permite reaccionar a eventos específicos (p.ej. transición a OPEN) en microsegundos, sin esperar el siguiente scrape de Prometheus.

El health check integrado con Spring Boot Actuator traduce el estado del Circuit Breaker al modelo de salud de la aplicación: un circuito OPEN puede marcar el servicio como DOWN en el endpoint `/actuator/health`, lo que interactúa directamente con los readiness probes de Kubernetes. Esta integración permite que Kubernetes retire el servicio del balanceo cuando detecta que sus dependencias están degradadas, evitando enviar tráfico a instancias que van a fallar igualmente.

> [ADVERTENCIA] Marcar la aplicación como DOWN en `/actuator/health` cuando un circuito está OPEN puede causar que Kubernetes reinicie los pods indefinidamente si todos los pods abren el mismo circuito simultáneamente. Evaluar cuidadosamente si la readiness probe debe depender del estado de los Circuit Breakers.

## Representación visual

El modelo de eventos de Resilience4j y su integración con el health check de Spring Boot:

```
Circuit Breaker Events (push model):
──────────────────────────────────────────────────────────────
  CircuitBreaker emite eventos:
  ┌─────────────────────────────────────────────────────┐
  │  ON_SUCCESS          → llamada exitosa              │
  │  ON_ERROR            → llamada fallida              │
  │  ON_TIMEOUT          → llamada con timeout          │
  │  ON_REJECTED_CALL    → llamada rechazada (CB OPEN)  │
  │  ON_STATE_TRANSITION → cambio de estado             │
  │  ON_RESET            → reset manual del circuito    │
  └─────────────────────────────┬───────────────────────┘
                                 │ EventPublisher
                                 ▼
  CircuitBreaker.getEventPublisher().onSuccess(e -> log(e))
  CircuitBreaker.getEventPublisher().onStateTransition(e -> alert(e))

Health Check Integration:
──────────────────────────────────────────────────────────────
  CircuitBreakerHealthIndicator
         │
         ├─ Estado CLOSED   → UP
         ├─ Estado HALF_OPEN → UP (configurable)
         └─ Estado OPEN     → DOWN

  /actuator/health responde:
  {
    "status": "DOWN",
    "components": {
      "circuitBreakers": {
        "status": "DOWN",
        "details": {
          "paymentService": { "status": "CIRCUIT_OPEN" },
          "inventoryService": { "status": "CIRCUIT_CLOSED" }
        }
      }
    }
  }
```

| Tipo de evento               | Cuándo se emite                                                 | Información incluida                              |
|------------------------------|-----------------------------------------------------------------|---------------------------------------------------|
| `ON_SUCCESS`                 | Llamada completada sin excepción en tiempo aceptable            | Duración, nombre del CB                           |
| `ON_ERROR`                   | Llamada lanzó excepción registrada como fallo                   | Excepción, duración, nombre del CB                |
| `ON_TIMEOUT`                 | Llamada superó `slowCallDurationThreshold` o TimeLimiter        | Duración, nombre del CB                           |
| `ON_REJECTED_CALL`           | Llamada rechazada porque el CB está OPEN                        | Nombre del CB, estado                             |
| `ON_STATE_TRANSITION`        | Transición entre estados (CLOSED↔OPEN↔HALF_OPEN)               | Estado anterior, estado nuevo                     |
| `ON_RESET`                   | Reset manual del circuito vía API o Actuator                    | Nombre del CB                                     |
| `ON_IGNORED_ERROR`           | Excepción lanzada pero configurada en `ignore-exceptions`       | Excepción, duración                               |

## Ejemplo central

Servicio con suscripción a eventos de Circuit Breaker para alertas en tiempo real y configuración del health check para integración con Kubernetes readiness probes.

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
# application.yml
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
        register-health-indicator: true
        # Número de eventos a retener en el buffer del EventPublisher
        event-consumer-buffer-size: 50
      inventoryService:
        sliding-window-size: 20
        minimum-number-of-calls: 5
        failure-rate-threshold: 60
        wait-duration-in-open-state: 15s
        register-health-indicator: true

management:
  endpoints:
    web:
      exposure:
        include: health,circuitbreakers,circuitbreakerevents
  endpoint:
    health:
      show-details: always
      # Separar liveness de readiness (Kubernetes)
      probes:
        enabled: true
  health:
    circuitbreakers:
      enabled: true
    livenessstate:
      enabled: true
    readinessstate:
      enabled: true
```

```java
// CircuitBreakerEventObserver.java — suscripción a eventos en tiempo real
package com.example.orders.observability;

import io.github.resilience4j.circuitbreaker.CircuitBreaker;
import io.github.resilience4j.circuitbreaker.CircuitBreakerRegistry;
import io.github.resilience4j.circuitbreaker.event.CircuitBreakerOnStateTransitionEvent;
import io.github.resilience4j.circuitbreaker.event.CircuitBreakerOnErrorEvent;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.boot.context.event.ApplicationReadyEvent;
import org.springframework.context.event.EventListener;
import org.springframework.stereotype.Component;

@Component
public class CircuitBreakerEventObserver {

    private static final Logger log = LoggerFactory.getLogger(CircuitBreakerEventObserver.class);
    private final CircuitBreakerRegistry circuitBreakerRegistry;

    public CircuitBreakerEventObserver(CircuitBreakerRegistry circuitBreakerRegistry) {
        this.circuitBreakerRegistry = circuitBreakerRegistry;
    }

    /**
     * Se registra en todos los Circuit Breakers cuando la aplicación está lista.
     * Usar ApplicationReadyEvent garantiza que los CBs ya están registrados.
     */
    @EventListener(ApplicationReadyEvent.class)
    public void registerEventSubscribers() {
        circuitBreakerRegistry.getAllCircuitBreakers().forEach(this::registerEvents);

        // También registrar en CBs que se creen después del arranque
        circuitBreakerRegistry.getEventPublisher()
                .onEntryAdded(event -> registerEvents(event.getAddedEntry()));
    }

    private void registerEvents(CircuitBreaker cb) {
        cb.getEventPublisher()
                .onStateTransition(this::handleStateTransition)
                .onError(this::handleError)
                .onCallNotPermitted(event ->
                        log.warn("[CB:{}] Llamada rechazada — circuito OPEN",
                                event.getCircuitBreakerName()))
                .onSuccess(event ->
                        log.debug("[CB:{}] Llamada exitosa en {}ms",
                                event.getCircuitBreakerName(),
                                event.getElapsedDuration().toMillis()));
    }

    private void handleStateTransition(CircuitBreakerOnStateTransitionEvent event) {
        String name = event.getCircuitBreakerName();
        CircuitBreaker.StateTransition transition = event.getStateTransition();

        log.warn("[CB:{}] Transición de estado: {} → {}",
                name,
                transition.getFromState(),
                transition.getToState());

        // Lógica de alerta específica para transición a OPEN
        if (transition.getToState() == CircuitBreaker.State.OPEN) {
            log.error("[CB:{}] CIRCUITO ABIERTO — las llamadas serán rechazadas durante {}",
                    name, "waitDurationInOpenState configurado");
            // Aquí se podría publicar un evento de aplicación, enviar a Slack, PagerDuty, etc.
        }
    }

    private void handleError(CircuitBreakerOnErrorEvent event) {
        log.error("[CB:{}] Error registrado: {} — duración: {}ms",
                event.getCircuitBreakerName(),
                event.getThrowable().getClass().getSimpleName(),
                event.getElapsedDuration().toMillis());
    }
}
```

```java
// CircuitBreakerManagementService.java — operaciones manuales en el CB (para testing/ops)
package com.example.orders.service;

import io.github.resilience4j.circuitbreaker.CircuitBreaker;
import io.github.resilience4j.circuitbreaker.CircuitBreakerRegistry;
import org.springframework.stereotype.Service;

@Service
public class CircuitBreakerManagementService {

    private final CircuitBreakerRegistry registry;

    public CircuitBreakerManagementService(CircuitBreakerRegistry registry) {
        this.registry = registry;
    }

    /**
     * Fuerza apertura del circuito manualmente (útil para mantenimiento).
     * Equivalente al endpoint /actuator/circuitbreakers/{name} via PATCH.
     */
    public void forceOpen(String name) {
        registry.circuitBreaker(name).transitionToOpenState();
    }

    /**
     * Fuerza cierre del circuito (útil después de un mantenimiento programado).
     */
    public void forceClose(String name) {
        registry.circuitBreaker(name).transitionToClosedState();
    }

    /**
     * Reset completo del circuito: borra el estado de la ventana deslizante.
     */
    public void reset(String name) {
        registry.circuitBreaker(name).reset();
    }

    public CircuitBreaker.State getState(String name) {
        return registry.circuitBreaker(name).getState();
    }
}
```

## Tabla de elementos clave

Propiedades y conceptos del health check y los eventos de Circuit Breaker.

| Propiedad / Concepto                                         | Tipo / Valor           | Descripción                                                                          |
|--------------------------------------------------------------|------------------------|--------------------------------------------------------------------------------------|
| `management.health.circuitbreakers.enabled`                  | boolean                | Activa el `CircuitBreakerHealthIndicator` en `/actuator/health`                      |
| `resilience4j.circuitbreaker.instances.[n].register-health-indicator` | boolean     | Incluye esta instancia en el health indicator                                        |
| `resilience4j.circuitbreaker.instances.[n].event-consumer-buffer-size` | int         | Eventos retenidos en buffer del EventPublisher (circular); default `100`             |
| `management.endpoint.health.probes.enabled`                  | boolean                | Activa endpoints `/actuator/health/liveness` y `/actuator/health/readiness`          |
| `CircuitBreakerRegistry.getEventPublisher().onEntryAdded()`  | `Consumer<EntryAddedEvent<CircuitBreaker>>` | Callback cuando se añade un nuevo CB al registry          |
| `CircuitBreaker.getEventPublisher().onStateTransition()`     | `Consumer<...>`        | Suscripción a cambios de estado                                                      |
| `CircuitBreaker.transitionToOpenState()`                     | método                 | Fuerza apertura manual del circuito                                                  |
| `CircuitBreaker.transitionToClosedState()`                   | método                 | Fuerza cierre manual del circuito                                                    |
| `CircuitBreaker.reset()`                                     | método                 | Reset del circuito y borrado de la ventana deslizante                                |
| Estado OPEN en health                                        | `DOWN` (default)       | Configurable via `CircuitBreakerHealthIndicator.allowHealthIndicatorToFail`          |

## Buenas y malas prácticas

**Hacer:**

- Usar `management.endpoint.health.probes.enabled: true` en producción con Kubernetes para separar la readiness probe de la liveness probe. La readiness debería incluir el estado de los CB; la liveness no (un CB abierto no significa que la aplicación deba reiniciarse).
- Suscribirse a `onStateTransition` en el `EventPublisher` para enviar alertas cuando el circuito transiciona a OPEN. Esta notificación en tiempo real es más rápida que esperar el scrape de Prometheus.
- Usar `CircuitBreakerRegistry.getEventPublisher().onEntryAdded()` para registrar suscriptores de forma dinámica en lugar de hacerlo en `@PostConstruct` por nombre. Esto garantiza que los CBs creados dinámicamente también tengan suscriptores.
- Registrar en logs el tipo de excepción que causó la apertura del circuito (`event.getThrowable()`). Sin esta información, un incidente de "circuito abierto" requiere correlacionar logs de otros servicios para entender la causa raíz.

**Evitar:**

- No configurar `management.health.circuitbreakers.enabled: true` si la readiness probe de Kubernetes apunta a `/actuator/health` sin evaluar cuidadosamente el impacto. Si todos los pods abren el circuito simultáneamente, Kubernetes los marcará como no-ready y dejará el servicio sin instancias disponibles.
- Evitar `event-consumer-buffer-size` muy pequeño (< 20) en servicios con alto throughput. Con un buffer pequeño, los eventos antiguos se descartan antes de que un operador pueda consultarlos en `/actuator/circuitbreakerevents`.
- No implementar lógica de negocio crítica en los suscriptores de eventos. Los suscriptores se ejecutan en el thread que llama al CB; una operación lenta o fallida en el suscriptor puede añadir latencia a todas las llamadas protegidas.

---

← [5.11 Métricas de Resilience4j con Micrometer y endpoints Actuator](sc-circuitbreaker-metricas.md) | [Índice](README.md) | [5.13 Integración del Circuit Breaker con OpenFeign, RestClient y WebClient](sc-circuitbreaker-feign.md) →
