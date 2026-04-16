# 13.2 Saga de coreografía con Spring Cloud Stream

← [13.1 CQRS con Spring Cloud Stream y Gateway](sc-patrones-cqrs.md) | [Índice (README.md)](README.md) | [13.3 Saga de orquestación con Spring Cloud Stream](sc-patrones-saga-orquestacion.md) →

---

Una Saga es el patrón que gestiona transacciones distribuidas entre múltiples microservicios cuando no se puede usar una transacción ACID distribuida (two-phase commit). En la variante de coreografía, no existe un coordinador central: cada servicio publica eventos cuando completa su parte de la transacción, y otros servicios reaccionan a esos eventos. Si un paso falla, el servicio publica un evento de fallo y los servicios anteriores ejecutan sus transacciones compensatorias (rollback de negocio). Spring Cloud Stream es la tecnología que implementa la comunicación entre pasos: cada paso es un `Consumer` o una función Spring Cloud Stream que reacciona a un evento y publica el siguiente.

> [PREREQUISITO] Requiere `spring-cloud-starter-stream-kafka`. Familiaridad con el modelo funcional de Spring Cloud Stream (fichero 6.x).

## Flujo de Saga de coreografía para un pedido

```
OrderService          InventoryService        PaymentService
     │                      │                      │
     │ OrderCreated ────────→│                      │
     │                      │                      │
     │               InventoryReserved ────────────→│
     │                      │                      │
     │                      │             PaymentProcessed ─→ ÉXITO
     │                      │                      │
     │ (si falla pago)       │            PaymentFailed ─────→│
     │                      │                      │
     │               InventoryReleased ←────────────│
     │                      │                      │
     │ OrderCancelled ←──────│                      │
```

## Ejemplo central: Saga de pedido con compensación

```java
package com.example.saga;

import org.springframework.cloud.stream.function.StreamBridge;
import org.springframework.context.annotation.Bean;
import org.springframework.messaging.Message;
import org.springframework.messaging.support.MessageBuilder;
import org.springframework.stereotype.Component;
import org.springframework.transaction.annotation.Transactional;

import java.util.function.Consumer;

// OrderService: inicia la saga publicando OrderCreated
@Component
public class OrderSagaParticipant {

    private final OrderRepository orderRepository;
    private final StreamBridge streamBridge;

    public OrderSagaParticipant(OrderRepository orderRepository, StreamBridge streamBridge) {
        this.orderRepository = orderRepository;
        this.streamBridge = streamBridge;
    }

    @Transactional
    public String createOrder(CreateOrderRequest request) {
        Order order = Order.create(request.productId(), request.quantity(), request.userId());
        orderRepository.save(order);

        // Publicar evento para que InventoryService reserve el stock
        streamBridge.send("order-created-out-0",
            MessageBuilder.withPayload(new OrderCreatedEvent(
                order.getId(), order.getProductId(), order.getQuantity()))
                .setHeader("sagaId", order.getId())
                .build());

        return order.getId();
    }

    // Transacción compensatoria: cancelar el pedido cuando el pago falla
    @Bean
    public Consumer<PaymentFailedEvent> paymentFailed() {
        return event -> {
            orderRepository.findById(event.orderId()).ifPresent(order -> {
                order.cancel("Payment failed: " + event.reason());
                orderRepository.save(order);
            });
        };
    }
}
```

```java
package com.example.saga;

import org.springframework.cloud.stream.function.StreamBridge;
import org.springframework.context.annotation.Bean;
import org.springframework.messaging.support.MessageBuilder;
import org.springframework.stereotype.Component;
import org.springframework.transaction.annotation.Transactional;

import java.util.function.Consumer;

// InventoryService: reserva stock y publica InventoryReserved o InventoryFailed
@Component
public class InventorySagaParticipant {

    private final InventoryRepository inventoryRepository;
    private final StreamBridge streamBridge;

    public InventorySagaParticipant(InventoryRepository inventoryRepository,
                                     StreamBridge streamBridge) {
        this.inventoryRepository = inventoryRepository;
        this.streamBridge = streamBridge;
    }

    @Bean
    public Consumer<OrderCreatedEvent> orderCreated() {
        return event -> {
            boolean reserved = tryReserveStock(event.productId(), event.quantity());
            if (reserved) {
                // Publicar para que PaymentService procese el pago
                streamBridge.send("inventory-reserved-out-0",
                    MessageBuilder.withPayload(new InventoryReservedEvent(
                        event.orderId(), event.productId(), event.quantity()))
                        .setHeader("sagaId", event.orderId())
                        .build());
            } else {
                // La reserva falló: notificar para compensar
                streamBridge.send("inventory-failed-out-0",
                    MessageBuilder.withPayload(new InventoryFailedEvent(
                        event.orderId(), "Insufficient stock"))
                        .build());
            }
        };
    }

    // Transacción compensatoria: liberar el stock cuando el pago falla
    @Bean
    public Consumer<PaymentFailedEvent> releaseInventory() {
        return event -> {
            releaseStock(event.orderId());
            streamBridge.send("inventory-released-out-0",
                MessageBuilder.withPayload(new InventoryReleasedEvent(event.orderId()))
                    .build());
        };
    }

    @Transactional
    private boolean tryReserveStock(String productId, int quantity) {
        return inventoryRepository.findByProductId(productId)
            .map(inv -> inv.reserve(quantity))
            .orElse(false);
    }

    @Transactional
    private void releaseStock(String orderId) {
        inventoryRepository.releaseByOrderId(orderId);
    }
}
```

```yaml
# application.yml del InventoryService
spring:
  cloud:
    stream:
      bindings:
        # Entrada: escucha OrderCreated del OrderService
        orderCreated-in-0:
          destination: order-created-events
          group: inventory-service-group
        # Salida: publica InventoryReserved
        inventory-reserved-out-0:
          destination: inventory-reserved-events
        # Salida: publica InventoryFailed
        inventory-failed-out-0:
          destination: inventory-failed-events
        # Entrada: escucha PaymentFailed para compensación
        releaseInventory-in-0:
          destination: payment-failed-events
          group: inventory-compensation-group
        # Salida: confirma que el stock fue liberado
        inventory-released-out-0:
          destination: inventory-released-events
```

> [CONCEPTO] La idempotencia de los pasos es crítica en la Saga de coreografía: Kafka puede entregar el mismo mensaje más de una vez (at-least-once delivery). Cada step debe implementar idempotencia — si recibe `OrderCreatedEvent` dos veces, la reserva de stock debe aplicarse solo una vez. El mecanismo estándar es guardar el `sagaId` (o `orderId`) en una tabla de "eventos procesados" y verificar antes de ejecutar la operación.

## Tabla de elementos clave

| Componente | Descripción |
|---|---|
| Saga de coreografía | Transacción distribuida sin coordinador central; cada servicio reacciona a eventos y publica el siguiente |
| Transacción compensatoria | Operación que deshace el efecto de un paso anterior cuando un paso posterior falla; NO es rollback transaccional |
| `Consumer<Event>` | Función Spring Cloud Stream que implementa un paso de la Saga |
| Idempotencia | Garantía de que un step produce el mismo resultado si recibe el mismo mensaje varias veces |
| `sagaId` en headers | Identificador de correlación que permite rastrear el estado de una Saga específica a través de los servicios |
| Evento de compensación | Evento publicado cuando un step falla; dispara las transacciones compensatorias de pasos anteriores |

## Buenas y malas prácticas

**Hacer:**
- Implementar idempotencia en todos los steps de la Saga: guardar el `sagaId` en una tabla de pasos completados y verificar antes de ejecutar la operación; Kafka entrega at-least-once.
- Diseñar las compensaciones para ser idempotentes también: si `releaseInventory` se ejecuta dos veces, el stock debe quedar en el estado correcto sin duplicar la liberación.
- Añadir un topic de "dead-letter" para mensajes que no pueden procesarse tras los reintentos; los steps que fallan repetidamente deben ser visibles en un sistema de monitorización para intervención manual.

**Evitar:**
- Implementar Sagas con transacciones síncronas entre servicios (llamadas REST síncronas en lugar de eventos); elimina la tolerancia a fallos parciales y crea acoplamiento temporal entre servicios.
- Ignorar el orden de las compensaciones: deben ejecutarse en orden inverso al de los steps completados; compensar el InventoryService antes de que el OrderService haya recibido el `PaymentFailed` puede dejar el sistema en estado inconsistente.
- Usar Sagas para operaciones que pueden ser atómicas dentro de un único servicio; si los datos que se modifican pertenecen al mismo dominio y base de datos, usar una transacción ACID local es siempre preferible.

---

← [13.1 CQRS con Spring Cloud Stream y Gateway](sc-patrones-cqrs.md) | [Índice (README.md)](README.md) | [13.3 Saga de orquestación con Spring Cloud Stream](sc-patrones-saga-orquestacion.md) →
