# 13.5 CQRS y Event Sourcing: patrón ES independiente, separación de modelos, proyecciones y consistencia eventual

← [13.4 Patrón Saga](sc-patrones-saga.md) | [Índice](README.md) | [13.6 API Composition y Aggregator](sc-patrones-api-composition-aggregator.md) →

---

## Introducción

CQRS (Command Query Responsibility Segregation) y Event Sourcing son dos patrones distintos que frecuentemente se combinan, pero cada uno tiene valor independiente. CQRS separa el modelo de escritura del de lectura para optimizar cada uno por separado. Event Sourcing almacena el historial completo de cambios de estado como una secuencia inmutable de eventos en lugar del estado actual. Su combinación es potente pero añade complejidad operacional significativa: solo se justifica cuando el negocio requiere audit trail completo, capacidad de replay o múltiples proyecciones de lectura optimizadas.

## CQRS: separación del modelo de escritura y lectura

> [CONCEPTO] **Command side vs Query side**: en CQRS, las operaciones de escritura (Commands) utilizan el modelo de dominio completo con toda su lógica de negocio e invariantes. Las operaciones de lectura (Queries) utilizan proyecciones desnormalizadas optimizadas para las consultas específicas del cliente. Cada lado puede usar una tecnología de persistencia diferente — el command side puede usar una base de datos relacional y el query side una base de datos orientada a documentos o un índice de búsqueda.

El beneficio principal es que cada modelo se optimiza para su propósito: el modelo de escritura garantiza consistencia, el de lectura garantiza velocidad. El coste es la complejidad de mantener ambos modelos sincronizados mediante eventos.

```
┌─────────────┐    Command     ┌──────────────────┐
│   Cliente   │ ─────────────► │  Command Handler  │ ── escribe en Write DB
└─────────────┘                └──────────────────┘
                                        │
                                 Domain Event
                                        │
                                        ▼
                               ┌──────────────────┐
                               │ Event Projector   │ ── actualiza Read DB
                               └──────────────────┘
┌─────────────┐    Query       ┌──────────────────┐
│   Cliente   │ ─────────────► │  Query Handler   │ ── lee de Read DB (proyección)
└─────────────┘                └──────────────────┘
```

## Event Sourcing como patrón independiente

> [CONCEPTO] **Event Sourcing**: en lugar de almacenar el estado actual de un agregado, el sistema almacena la secuencia de eventos que llevaron a ese estado. El estado actual se reconstruye aplicando los eventos en orden. El event store es append-only: los eventos son inmutables una vez escritos.

Event Sourcing tiene sentido cuando:
- Se necesita **audit trail** completo de todos los cambios (sectores regulados: banca, salud).
- Se necesita **time travel**: reconstruir el estado en cualquier punto del pasado.
- Se necesita **replay**: re-proyectar los eventos para construir nuevas vistas.
- Se generan múltiples **proyecciones** a partir del mismo stream de eventos.

Event Sourcing NO tiene sentido cuando:
- Alta frecuencia de escritura sin necesidad de historial (telemetría, logs de acceso).
- Sistema simple CRUD donde el historial no aporta valor.
- El equipo no tiene experiencia con el patrón y el contexto no lo justifica.

> [ADVERTENCIA] Event Sourcing introduce complejidad operacional significativa: schema evolution de eventos (cómo migrar eventos pasados cuando el esquema cambia), crecimiento ilimitado del event store (requiere snapshots), y complejidad en las proyecciones. No aplicar por defecto.

## Proyecciones: modelos de lectura en CQRS/ES

> [CONCEPTO] **Proyección**: una proyección es una vista materializada construida a partir de los eventos del event store. Es el "modelo de lectura" en CQRS. Las proyecciones se actualizan de forma asíncrona cuando se publican nuevos eventos — esto introduce consistencia eventual entre el command side y el query side.

Las proyecciones son **reconstruibles**: si se necesita una nueva proyección o se corrige un bug en una existente, se puede hacer replay de todos los eventos para construir la proyección desde cero.

> [CONCEPTO] **Eventual consistency en CQRS**: un cliente que escribe un comando y luego inmediatamente lee la proyección puede ver datos desactualizados. El sistema es eventualmente consistente — las proyecciones convergen al estado correcto, pero con un lag. Estrategias para mitigar el lag: devolver el estado actualizado directamente desde el command handler en el response, o usar un correlation ID para que el cliente sepa cuándo su escritura se refleja en la proyección.

## Ejemplo central: Event Sourcing con event store manual en Spring Boot

El siguiente ejemplo implementa Event Sourcing completo para un agregado `Order`: el event store, la reconstrucción del estado desde eventos, y una proyección de lectura actualizada por un event listener.

```java
// Evento de dominio base
package com.example.orders.eventsourcing;

import java.time.Instant;

public abstract class DomainEvent {
    private final String aggregateId;
    private final Instant occurredOn;
    private final long version;

    protected DomainEvent(String aggregateId, long version) {
        this.aggregateId = aggregateId;
        this.occurredOn = Instant.now();
        this.version = version;
    }

    public String getAggregateId() { return aggregateId; }
    public Instant getOccurredOn() { return occurredOn; }
    public long getVersion() { return version; }
}

// Eventos concretos del agregado Order
public class OrderPlacedEvent extends DomainEvent {
    private final String customerId;
    private final double amount;

    public OrderPlacedEvent(String orderId, String customerId, double amount, long version) {
        super(orderId, version);
        this.customerId = customerId;
        this.amount = amount;
    }

    public String getCustomerId() { return customerId; }
    public double getAmount() { return amount; }
}

public class OrderConfirmedEvent extends DomainEvent {
    public OrderConfirmedEvent(String orderId, long version) {
        super(orderId, version);
    }
}

public class OrderCancelledEvent extends DomainEvent {
    private final String reason;

    public OrderCancelledEvent(String orderId, String reason, long version) {
        super(orderId, version);
        this.reason = reason;
    }

    public String getReason() { return reason; }
}
```

```java
// Agregado Order reconstruido desde eventos
package com.example.orders.eventsourcing;

import java.util.ArrayList;
import java.util.List;

public class Order {

    private String id;
    private String customerId;
    private double amount;
    private String status;
    private long version = 0;

    private final List<DomainEvent> uncommittedEvents = new ArrayList<>();

    // Constructor privado: solo se instancia desde el factory o desde replay
    private Order() {}

    // Factory para nuevas órdenes
    public static Order place(String orderId, String customerId, double amount) {
        Order order = new Order();
        order.apply(new OrderPlacedEvent(orderId, customerId, amount, 1));
        return order;
    }

    // Reconstrucción desde eventos del event store
    public static Order reconstitute(List<DomainEvent> history) {
        Order order = new Order();
        for (DomainEvent event : history) {
            order.applyHistoric(event);
        }
        return order;
    }

    public void confirm() {
        if (!"PENDING".equals(status)) {
            throw new IllegalStateException("Only PENDING orders can be confirmed");
        }
        apply(new OrderConfirmedEvent(id, version + 1));
    }

    public void cancel(String reason) {
        if ("CANCELLED".equals(status) || "COMPLETED".equals(status)) {
            throw new IllegalStateException("Cannot cancel order in status: " + status);
        }
        apply(new OrderCancelledEvent(id, reason, version + 1));
    }

    // Aplica evento nuevo (lo añade a uncommitted para persistencia)
    private void apply(DomainEvent event) {
        applyHistoric(event);
        uncommittedEvents.add(event);
    }

    // Aplica evento histórico (reconstrucción del estado)
    private void applyHistoric(DomainEvent event) {
        if (event instanceof OrderPlacedEvent e) {
            this.id = e.getAggregateId();
            this.customerId = e.getCustomerId();
            this.amount = e.getAmount();
            this.status = "PENDING";
        } else if (event instanceof OrderConfirmedEvent) {
            this.status = "CONFIRMED";
        } else if (event instanceof OrderCancelledEvent) {
            this.status = "CANCELLED";
        }
        this.version = event.getVersion();
    }

    public List<DomainEvent> getUncommittedEvents() {
        return List.copyOf(uncommittedEvents);
    }

    public void markEventsAsCommitted() {
        uncommittedEvents.clear();
    }

    public String getId() { return id; }
    public String getStatus() { return status; }
    public long getVersion() { return version; }
}
```

```java
// Event Store: persistencia append-only de eventos
package com.example.orders.eventsourcing;

import jakarta.persistence.*;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;
import java.util.List;

@Entity
@Table(name = "order_events")
public class OrderEventRecord {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String aggregateId;

    @Column(nullable = false)
    private String eventType;

    @Column(nullable = false, columnDefinition = "TEXT")
    private String payload; // JSON del evento

    @Column(nullable = false)
    private long version;

    // constructor, getters omitidos pero presentes en producción
    public OrderEventRecord() {}

    public OrderEventRecord(String aggregateId, String eventType, String payload, long version) {
        this.aggregateId = aggregateId;
        this.eventType = eventType;
        this.payload = payload;
        this.version = version;
    }

    public String getAggregateId() { return aggregateId; }
    public String getEventType() { return eventType; }
    public String getPayload() { return payload; }
    public long getVersion() { return version; }
}

@Repository
public interface OrderEventRepository extends JpaRepository<OrderEventRecord, Long> {
    List<OrderEventRecord> findByAggregateIdOrderByVersionAsc(String aggregateId);
}
```

```java
// Proyección de lectura: vista desnormalizada de órdenes
package com.example.orders.eventsourcing.projection;

import com.example.orders.eventsourcing.OrderConfirmedEvent;
import com.example.orders.eventsourcing.OrderPlacedEvent;
import jakarta.persistence.*;
import org.springframework.context.event.EventListener;
import org.springframework.stereotype.Component;

// Proyección en BD relacional: tabla desnormalizada para consultas rápidas
@Entity
@Table(name = "order_summary")
public class OrderSummary {

    @Id
    private String orderId;
    private String customerId;
    private double amount;
    private String status;

    public OrderSummary() {}

    public OrderSummary(String orderId, String customerId, double amount, String status) {
        this.orderId = orderId;
        this.customerId = customerId;
        this.amount = amount;
        this.status = status;
    }

    public void updateStatus(String newStatus) { this.status = newStatus; }
    public String getOrderId() { return orderId; }
    public String getStatus() { return status; }
}

// Proyector: actualiza la proyección al recibir eventos de dominio
@Component
public class OrderProjector {

    private final OrderSummaryRepository summaryRepository;

    public OrderProjector(OrderSummaryRepository summaryRepository) {
        this.summaryRepository = summaryRepository;
    }

    @EventListener
    public void on(OrderPlacedEvent event) {
        summaryRepository.save(new OrderSummary(
            event.getAggregateId(),
            event.getCustomerId(),
            event.getAmount(),
            "PENDING"
        ));
    }

    @EventListener
    public void on(OrderConfirmedEvent event) {
        summaryRepository.findById(event.getAggregateId())
            .ifPresent(summary -> {
                summary.updateStatus("CONFIRMED");
                summaryRepository.save(summary);
            });
    }
}
```

## Buenas y malas prácticas

**Buenas prácticas:**
- Usar snapshots periódicos para evitar que la reconstrucción desde el inicio del event store sea demasiado costosa conforme crece el historial.
- Versionar los eventos (schema versioning) para manejar la evolución del esquema sin romper el replay histórico.
- Separar el event store físico del write model cuando la carga lo justifique.
- Implementar idempotencia en los proyectores para tolerar entregas duplicadas.

**Malas prácticas:**
- Usar Event Sourcing en sistemas simples CRUD sin audit trail ni necesidad de replay.
- Mezclar eventos de dominio con eventos de integración sin distinguirlos.
- Proyectar directamente sin manejar el orden y la idempotencia — genera proyecciones corruptas ante mensajes duplicados o fuera de orden.

## Verificación y práctica

> [EXAMEN] 1. ¿Qué ventajas aporta separar el modelo de escritura del de lectura en CQRS, y qué complejidad operacional introduce?

> [EXAMEN] 2. ¿Qué es Event Sourcing como patrón independiente, y en qué tres escenarios NO es recomendable aplicarlo?

> [EXAMEN] 3. ¿Qué es una proyección en CQRS/Event Sourcing, cómo se actualiza y qué implica hacer un replay desde cero?

> [EXAMEN] 4. ¿Cómo explicas a un cliente que los datos de lectura pueden estar momentáneamente desactualizados respecto a los de escritura, y qué estrategias existen para mitigar el lag de las proyecciones?

> [EXAMEN] 5. ¿Cómo combinas Event Sourcing y CQRS? ¿Qué significa reconstruir el estado de un agregado a partir de su log de eventos?

---

← [13.4 Patrón Saga](sc-patrones-saga.md) | [Índice](README.md) | [13.6 API Composition y Aggregator](sc-patrones-api-composition-aggregator.md) →
