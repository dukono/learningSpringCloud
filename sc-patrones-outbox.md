# 13.4 Outbox Pattern con Spring Cloud Stream

← [13.3 Saga de orquestación con Spring Cloud Stream](sc-patrones-saga-orquestacion.md) | [Índice (README.md)](README.md) | [13.5 Strangler Fig con Spring Cloud Gateway](sc-patrones-strangler.md) →

---

El Outbox Pattern resuelve el problema de dualidad write: al guardar un cambio en la base de datos Y publicar un evento en Kafka, si la base de datos confirma pero Kafka falla (o viceversa), el sistema queda en estado inconsistente. El patrón usa la base de datos como intermediario: el servicio escribe tanto el cambio de dominio como el evento en la misma transacción local (en una tabla `outbox`); un proceso separado (relay) lee la tabla `outbox` y publica los eventos en Kafka, con la garantía de at-least-once delivery. Spring Cloud Stream implementa el relay mediante Debezium (Change Data Capture) o un scheduler con polling. La integración con Spring Boot y Spring Cloud Stream hace que este patrón sea transparente para la lógica de negocio.

> [PREREQUISITO] Requiere `spring-cloud-starter-stream-kafka` y `spring-boot-starter-data-jpa`. Para CDC con Debezium: `debezium-embedded` o un conector Kafka Debezium externo.

## Flujo del Outbox Pattern

```
┌────────────────────────────────────────────────────────────────┐
│  OrderService (misma transacción JPA)                           │
│                                                                  │
│  BEGIN TRANSACTION                                               │
│    INSERT INTO orders (id, ...)   ← cambio de dominio           │
│    INSERT INTO outbox_events      ← evento pendiente de envío   │
│       (id, aggregateId, type, payload, status=PENDING)          │
│  COMMIT                                                          │
└────────────────────────────────────────────────────────────────┘
                        │
                        │ (proceso separado, polling o CDC)
                        ▼
┌────────────────────────────────────────────────────────────────┐
│  OutboxRelay                                                     │
│    SELECT * FROM outbox_events WHERE status=PENDING             │
│    → StreamBridge.send(event) → Kafka                           │
│    UPDATE outbox_events SET status=SENT WHERE id=...            │
└────────────────────────────────────────────────────────────────┘
```

## Ejemplo central: Outbox con polling scheduler

### Entidad Outbox y repositorio

```java
package com.example.outbox;

import jakarta.persistence.*;
import java.time.Instant;

@Entity
@Table(name = "outbox_events",
       indexes = @Index(columnList = "status, createdAt"))
public class OutboxEvent {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private String id;

    private String aggregateId;    // ej: orderId
    private String aggregateType;  // ej: "Order"
    private String eventType;      // ej: "OrderCreated"

    @Column(columnDefinition = "TEXT")
    private String payload;         // JSON serializado del evento

    @Enumerated(EnumType.STRING)
    private OutboxStatus status = OutboxStatus.PENDING;

    private Instant createdAt = Instant.now();
    private Instant processedAt;

    public static OutboxEvent create(String aggregateId, String aggregateType,
                                      String eventType, String payload) {
        OutboxEvent event = new OutboxEvent();
        event.aggregateId = aggregateId;
        event.aggregateType = aggregateType;
        event.eventType = eventType;
        event.payload = payload;
        return event;
    }

    public void markSent() {
        this.status = OutboxStatus.SENT;
        this.processedAt = Instant.now();
    }

    public void markFailed() {
        this.status = OutboxStatus.FAILED;
    }

    // Getters
    public String getId() { return id; }
    public String getAggregateId() { return aggregateId; }
    public String getEventType() { return eventType; }
    public String getPayload() { return payload; }
    public OutboxStatus getStatus() { return status; }
}

enum OutboxStatus { PENDING, SENT, FAILED }
```

### Servicio de dominio — escribe dominio y outbox en la misma transacción

```java
package com.example.outbox;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class OrderService {

    private final OrderRepository orderRepository;
    private final OutboxRepository outboxRepository;
    private final ObjectMapper objectMapper;

    public OrderService(OrderRepository orderRepository,
                         OutboxRepository outboxRepository,
                         ObjectMapper objectMapper) {
        this.orderRepository = orderRepository;
        this.outboxRepository = outboxRepository;
        this.objectMapper = objectMapper;
    }

    @Transactional
    public Order createOrder(CreateOrderRequest request) throws Exception {
        // 1. Persistir el pedido
        Order order = new Order(request.productId(), request.quantity(), request.userId());
        orderRepository.save(order);

        // 2. En la MISMA transacción, guardar el evento en outbox
        String payload = objectMapper.writeValueAsString(new OrderCreatedEvent(
            order.getId(), order.getProductId(), order.getQuantity(), order.getUserId()
        ));

        OutboxEvent outboxEvent = OutboxEvent.create(
            order.getId(), "Order", "OrderCreated", payload);
        outboxRepository.save(outboxEvent);

        // 3. Si este método falla DESPUÉS del save → rollback de AMBAS tablas
        // Si falla al enviar a Kafka → outboxRelay lo reintentará
        return order;
    }
}
```

### OutboxRelay — proceso de polling que publica los eventos pendientes

```java
package com.example.outbox;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.cloud.stream.function.StreamBridge;
import org.springframework.data.domain.PageRequest;
import org.springframework.messaging.support.MessageBuilder;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Component
public class OutboxRelay {

    private final OutboxRepository outboxRepository;
    private final StreamBridge streamBridge;
    private final ObjectMapper objectMapper;

    public OutboxRelay(OutboxRepository outboxRepository,
                        StreamBridge streamBridge,
                        ObjectMapper objectMapper) {
        this.outboxRepository = outboxRepository;
        this.streamBridge = streamBridge;
        this.objectMapper = objectMapper;
    }

    // Polling cada 500ms — procesar en lotes de 100
    @Scheduled(fixedDelay = 500)
    @Transactional
    public void relayPendingEvents() {
        List<OutboxEvent> pendingEvents = outboxRepository
            .findByStatusOrderByCreatedAtAsc(OutboxStatus.PENDING, PageRequest.of(0, 100));

        for (OutboxEvent outboxEvent : pendingEvents) {
            try {
                // Enviar el payload JSON como mensaje a Kafka
                streamBridge.send(
                    topicForEventType(outboxEvent.getEventType()),
                    MessageBuilder.withPayload(outboxEvent.getPayload().getBytes())
                        .setHeader("eventType", outboxEvent.getEventType())
                        .setHeader("aggregateId", outboxEvent.getAggregateId())
                        .build()
                );
                outboxEvent.markSent();
            } catch (Exception e) {
                outboxEvent.markFailed();
            }
            outboxRepository.save(outboxEvent);
        }
    }

    private String topicForEventType(String eventType) {
        return switch (eventType) {
            case "OrderCreated" -> "order-created-out-0";
            case "OrderCancelled" -> "order-cancelled-out-0";
            default -> "domain-events-out-0";
        };
    }
}
```

> [CONCEPTO] El Outbox Pattern garantiza at-least-once delivery (no exactamente-once): si el relay falla después de enviar el evento a Kafka pero antes de actualizar el status a SENT, el evento se enviará de nuevo en el próximo ciclo de polling. Los consumers deben implementar idempotencia para manejar mensajes duplicados.

> [ADVERTENCIA] El polling con `@Scheduled` introduce latencia proporcional al `fixedDelay`. Para latencias menores de 100ms, considerar Debezium (Change Data Capture) que reacciona en tiempo real a los cambios en el WAL de PostgreSQL, sin necesidad de polling.

## Tabla de elementos clave

| Componente | Descripción |
|---|---|
| Tabla `outbox_events` | Buffer transaccional de eventos; escrita en la misma transacción que el cambio de dominio |
| `status=PENDING` | Estado inicial; el relay lo cambia a SENT o FAILED tras el intento de publicación |
| `@Scheduled(fixedDelay=500)` | Polling del relay; la latencia del relay es proporcional al `fixedDelay` |
| `StreamBridge.send(binding, message)` | Publicación imperativa del evento desde el relay al broker |
| At-least-once delivery | El Outbox garantiza que el evento se publica al menos una vez; los consumers deben ser idempotentes |
| Debezium CDC | Alternativa al polling: captura cambios del WAL de la BD y los publica en Kafka en tiempo real; latencia < 100ms |

## Buenas y malas prácticas

**Hacer:**
- Incluir un campo `attempts` en `outbox_events` para limitar los reintentos; después de N intentos fallidos, mover a una tabla `outbox_dead_letter` para inspección manual.
- Limpiar periódicamente los registros SENT de `outbox_events`; sin limpieza la tabla crece indefinidamente y degrada el rendimiento del índice de consulta `WHERE status=PENDING`.
- Usar Debezium con PostgreSQL en producción cuando la latencia requerida es < 1 segundo; el polling con 500ms introduce hasta 500ms de latencia adicional en cada evento.

**Evitar:**
- Llamar a `streamBridge.send()` directamente en el método de servicio de negocio dentro de la transacción JPA; si la transacción hace rollback después del envío a Kafka, el evento ya fue publicado y no puede deshacerse — es el problema exacto que el Outbox resuelve.
- Usar la tabla de outbox para eventos de alta frecuencia sin indexar correctamente; un millón de registros PENDING sin índice convierte el SELECT de polling en una operación de full table scan.
- Procesar los eventos del outbox en múltiples instancias del servicio sin coordinar quién procesa qué; puede resultar en duplicación de publicaciones y contención en la tabla. Usar `SELECT FOR UPDATE SKIP LOCKED` (PostgreSQL) para distribuir el trabajo entre instancias sin contención.

---

← [13.3 Saga de orquestación con Spring Cloud Stream](sc-patrones-saga-orquestacion.md) | [Índice (README.md)](README.md) | [13.5 Strangler Fig con Spring Cloud Gateway](sc-patrones-strangler.md) →
