# 13.8 Resiliencia transversal en patrones distribuidos

← [13.7 Sidecar Pattern con Spring Cloud Netflix Sidecar](sc-patrones-sidecar.md) | [Índice (README.md)](README.md) | [13.9 Testing de patrones distribuidos con Spring Cloud](sc-patrones-testing.md) →

---

Los patrones distribuidos (CQRS, Saga, Outbox, API Composition) introducen rutas de fallo específicas que la resiliencia básica de un servicio individual no cubre: una Saga puede quedar en estado inconsistente si el orquestador cae a mitad del flujo; un Outbox relay puede acumular eventos pendientes si Kafka no está disponible; una API Composition puede fallar completamente si uno de los microservicios downstream no responde. La resiliencia transversal añade mecanismos que van por encima del Circuit Breaker individual: timeouts de Saga, dead-letter queues para eventos no procesados, respuestas degradadas en composición, y políticas de retry específicas por tipo de operación. Spring Cloud y Resilience4j proveen los building blocks; la arquitectura del sistema determina cómo combinarlos.

> [PREREQUISITO] Requiere `spring-cloud-starter-circuitbreaker-resilience4j` y, según el patrón: `spring-cloud-starter-stream-kafka` (para DLQ en Saga/Outbox) y `spring-cloud-starter-openfeign` (para resiliencia en API Composition).

## Resiliencia por patrón

```
┌────────────────────────────────────────────────────────────────┐
│  Saga         → Timeout de Saga + compensación automática      │
│  Outbox       → Retry con backoff + dead-letter queue          │
│  CQRS         → Circuit Breaker en proyecciones + fallback     │
│  API Comp.    → Timeout paralelo + respuesta degradada         │
└────────────────────────────────────────────────────────────────┘
```

## Ejemplo central: resiliencia en Saga, Outbox y API Composition

### Timeout de Saga con @Scheduled

```java
package com.example.resilience;

import org.springframework.cloud.stream.function.StreamBridge;
import org.springframework.messaging.support.MessageBuilder;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;
import org.springframework.transaction.annotation.Transactional;

import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.List;

// Proceso que detecta Sagas bloqueadas y las compensa automáticamente
@Component
public class SagaTimeoutMonitor {

    private final SagaStateRepository sagaStateRepository;
    private final StreamBridge streamBridge;

    private static final int SAGA_TIMEOUT_MINUTES = 30;

    public SagaTimeoutMonitor(SagaStateRepository sagaStateRepository,
                               StreamBridge streamBridge) {
        this.sagaStateRepository = sagaStateRepository;
        this.streamBridge = streamBridge;
    }

    @Scheduled(fixedDelay = 60_000)  // revisar cada minuto
    @Transactional
    public void detectTimedOutSagas() {
        Instant cutoff = Instant.now().minus(SAGA_TIMEOUT_MINUTES, ChronoUnit.MINUTES);

        // Buscar Sagas en STARTED que no han progresado en SAGA_TIMEOUT_MINUTES
        List<SagaState> timedOut = sagaStateRepository
            .findByStatusAndCreatedAtBefore(SagaStatus.STARTED, cutoff);

        for (SagaState saga : timedOut) {
            saga.fail(saga.getCurrentStep(), "Saga timeout after " + SAGA_TIMEOUT_MINUTES + " minutes");
            sagaStateRepository.save(saga);

            // Publicar evento de fallo para que los participantes compensen
            streamBridge.send("saga-timeout-out-0",
                MessageBuilder.withPayload(new SagaTimedOutEvent(saga.getSagaId(), saga.getCurrentStep()))
                    .build());
        }
    }
}
```

### Dead-Letter Queue para el Outbox Relay

```yaml
# application.yml del Outbox Relay con DLQ en Kafka
spring:
  cloud:
    stream:
      bindings:
        # Canal principal de eventos de dominio
        domain-events-out-0:
          destination: domain-events
          producer:
            error-channel-enabled: true  # habilitar canal de error

      kafka:
        bindings:
          domain-events-out-0:
            producer:
              # Configurar dead-letter topic automático
              dlq-name: domain-events.DLT
              # Número de reintentos antes de enviar a DLQ
              retries: 3
              # Backoff entre reintentos
              retry-backoff-initial-interval: 1000
              retry-backoff-multiplier: 2.0
              retry-backoff-max-interval: 30000
```

```java
package com.example.resilience;

import org.springframework.context.annotation.Bean;
import org.springframework.integration.annotation.ServiceActivator;
import org.springframework.messaging.Message;
import org.springframework.stereotype.Component;

// Procesador del Dead-Letter Queue: alertar cuando un evento no puede publicarse
@Component
public class OutboxDlqProcessor {

    @ServiceActivator(inputChannel = "domain-events-out-0.errors")
    public void handleOutboxPublishError(Message<?> failedMessage) {
        // Loggear el mensaje fallido con suficiente contexto para diagnóstico
        String aggregateId = (String) failedMessage.getHeaders().get("aggregateId");
        String eventType = (String) failedMessage.getHeaders().get("eventType");
        
        // Alertar: un evento no pudo publicarse tras los reintentos
        System.err.println("OUTBOX PUBLISH FAILED: eventType=" + eventType 
            + " aggregateId=" + aggregateId);
        
        // En producción: enviar alerta a PagerDuty/alerting
        // El mensaje ya está en el DLT → no está perdido, requiere intervención manual
    }
}
```

### Circuit Breaker en API Composition con respuesta degradada

```java
package com.example.resilience;

import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.TimeUnit;

@Service
public class ResilientCompositionService {

    private final OrderServiceClient orderClient;
    private final ProductServiceClient productClient;
    private final UserServiceClient userClient;

    public ResilientCompositionService(OrderServiceClient orderClient,
                                        ProductServiceClient productClient,
                                        UserServiceClient userClient) {
        this.orderClient = orderClient;
        this.productClient = productClient;
        this.userClient = userClient;
    }

    // Composición resiliente: timeout global y respuesta degradada por servicio
    public OrderDashboardView getOrderDashboardResilient(String orderId) {
        OrderDto order = getOrderWithFallback(orderId);

        CompletableFuture<ProductDto> productFuture = CompletableFuture
            .supplyAsync(() -> getProductWithFallback(order.getProductId()))
            .orTimeout(2, TimeUnit.SECONDS)
            .exceptionally(ex -> ProductDto.unavailable(order.getProductId()));

        CompletableFuture<UserDto> userFuture = CompletableFuture
            .supplyAsync(() -> getUserWithFallback(order.getUserId()))
            .orTimeout(2, TimeUnit.SECONDS)
            .exceptionally(ex -> UserDto.anonymous(order.getUserId()));

        return new OrderDashboardView(order, productFuture.join(), userFuture.join());
    }

    @CircuitBreaker(name = "order-service", fallbackMethod = "orderFallback")
    public OrderDto getOrderWithFallback(String orderId) {
        return orderClient.getOrder(orderId);
    }

    public OrderDto orderFallback(String orderId, Throwable ex) {
        // Respuesta degradada: datos mínimos del pedido sin acceder a la BD
        return OrderDto.placeholder(orderId);
    }

    @CircuitBreaker(name = "product-service", fallbackMethod = "productFallback")
    public ProductDto getProductWithFallback(String productId) {
        return productClient.getProduct(productId);
    }

    public ProductDto productFallback(String productId, Throwable ex) {
        return ProductDto.unavailable(productId);
    }

    @CircuitBreaker(name = "user-service", fallbackMethod = "userFallback")
    public UserDto getUserWithFallback(String userId) {
        return userClient.getUser(userId);
    }

    public UserDto userFallback(String userId, Throwable ex) {
        return UserDto.anonymous(userId);
    }
}
```

```yaml
# application.yml — configuración Resilience4j para la composición
resilience4j:
  circuitbreaker:
    instances:
      order-service:
        slidingWindowSize: 10
        failureRateThreshold: 50
        waitDurationInOpenState: 10s
        permittedNumberOfCallsInHalfOpenState: 3
      product-service:
        slidingWindowSize: 10
        failureRateThreshold: 60    # más tolerante: un fallo de producto no es crítico
        waitDurationInOpenState: 5s
      user-service:
        slidingWindowSize: 10
        failureRateThreshold: 60
        waitDurationInOpenState: 5s
  timelimiter:
    instances:
      order-service:
        timeoutDuration: 2s
      product-service:
        timeoutDuration: 1s
      user-service:
        timeoutDuration: 1s
```

> [CONCEPTO] La resiliencia en patrones distribuidos requiere decisiones de diseño explícitas sobre qué datos son imprescindibles (si `order-service` falla, la composición falla completamente) y qué son enriquecimientos opcionales (si `product-service` falla, se devuelven los datos del pedido con información de producto parcial). Esta decisión de degradación debe estar en el diseño del sistema, no descubierta en producción.

## Tabla de elementos clave

| Componente | Descripción |
|---|---|
| `SagaTimeoutMonitor` | Proceso que detecta Sagas bloqueadas más de X minutos y las marca como fallidas con compensación |
| Dead-Letter Topic (DLT) | Topic de Kafka donde van los mensajes que no pueden procesarse tras los reintentos; requiere monitorización |
| `retry-backoff-multiplier` | Backoff exponencial entre reintentos del Outbox relay; reduce la presión sobre un Kafka degradado |
| `orTimeout(2, TimeUnit.SECONDS)` | Timeout en `CompletableFuture`; la composición no espera más de 2s por cada servicio downstream |
| Respuesta degradada | Objeto con datos mínimos o placeholder cuando un servicio downstream no responde |
| `@CircuitBreaker(name=..., fallbackMethod=...)` | Circuit Breaker de Resilience4j por servicio downstream; fallback específico por tipo de dato |

## Buenas y malas prácticas

**Hacer:**
- Definir un timeout de Saga apropiado para el dominio (negocio); un timeout demasiado corto genera compensaciones innecesarias en picos de carga; uno demasiado largo deja recursos bloqueados y pedidos en estado inconsistente visible para el usuario.
- Monitorizar el Dead-Letter Topic con alertas: un DLT con mensajes acumulándose indica un problema sistémico en el Outbox o en los consumers; requiere intervención antes de que los eventos sean demasiados para procesarse manualmente.
- Distinguir entre fallos de servicios primarios (requieren fallo completo de la composición) y servicios de enriquecimiento (pueden devolver datos parciales); diseñar el `fallbackMethod` de acuerdo a esta clasificación.

**Evitar:**
- Usar el mismo `waitDurationInOpenState` para todos los Circuit Breakers en la composición; un servicio de catálogo que es lento debería tener un timeout de apertura más corto que un servicio de pagos que es crítico.
- Ignorar los mensajes del Dead-Letter Topic hasta que el volumen sea inmanejable; cada mensaje en el DLT representa un evento de dominio que no fue procesado — la consistencia del sistema depende de procesarlos eventualmente.
- Compensar Sagas en timeout sin verificar el estado actual en la base de datos; un timeout puede ocurrir cuando el paso está en proceso — compensar en ese momento puede causar una condición de carrera entre la compensación y la respuesta en camino del participante.

---

← [13.7 Sidecar Pattern con Spring Cloud Netflix Sidecar](sc-patrones-sidecar.md) | [Índice (README.md)](README.md) | [13.9 Testing de patrones distribuidos con Spring Cloud](sc-patrones-testing.md) →
