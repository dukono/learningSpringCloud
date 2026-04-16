# 13.3 Saga de orquestación con Spring Cloud Stream

← [13.2 Saga de coreografía con Spring Cloud Stream](sc-patrones-saga-coreografia.md) | [Índice (README.md)](README.md) | [13.4 Outbox Pattern con Spring Cloud Stream](sc-patrones-outbox.md) →

---

La Saga de orquestación introduce un Saga Orchestrator: un componente central que conoce la secuencia completa de pasos, envía Commands a los participantes, espera sus respuestas y decide el siguiente paso o la compensación. A diferencia de la coreografía, el estado de la Saga (qué pasos se han completado, qué pasos han fallado) está centralizado en el orquestador, lo que facilita el debugging, el monitoring y la implementación de flujos condicionales complejos. La desventaja es que el orquestador introduce un punto de acoplamiento central y debe ser altamente disponible. Spring Cloud Stream implementa el orquestador como un conjunto de funciones que reaccionan a respuestas de los participantes y publican el siguiente Command.

> [PREREQUISITO] Requiere `spring-cloud-starter-stream-kafka`. El patrón complementa la Saga de coreografía del fichero 13.2; ambos son válidos, con trade-offs distintos.

## Ejemplo central: Saga Orchestrator para pedidos

### SagaOrchestrator — estado central y lógica de flujo

```java
package com.example.saga;

import org.springframework.cloud.stream.function.StreamBridge;
import org.springframework.context.annotation.Bean;
import org.springframework.messaging.support.MessageBuilder;
import org.springframework.stereotype.Component;
import org.springframework.transaction.annotation.Transactional;

import java.util.function.Consumer;

@Component
public class OrderSagaOrchestrator {

    private final SagaStateRepository sagaStateRepository;
    private final StreamBridge streamBridge;

    public OrderSagaOrchestrator(SagaStateRepository sagaStateRepository,
                                  StreamBridge streamBridge) {
        this.sagaStateRepository = sagaStateRepository;
        this.streamBridge = streamBridge;
    }

    // Punto de entrada: inicia la Saga
    @Transactional
    public String startOrderSaga(CreateOrderRequest request) {
        // Guardar estado inicial de la Saga
        SagaState state = SagaState.create(
            request.orderId(), SagaStep.RESERVE_INVENTORY);
        sagaStateRepository.save(state);

        // Enviar Command al InventoryService
        streamBridge.send("reserve-inventory-out-0",
            MessageBuilder.withPayload(new ReserveInventoryCommand(
                request.orderId(), request.productId(), request.quantity()))
                .setHeader("sagaId", request.orderId())
                .build());

        return request.orderId();
    }

    // Respuesta del InventoryService: reserva exitosa
    @Bean
    public Consumer<InventoryReservedEvent> inventoryReserved() {
        return event -> {
            SagaState state = sagaStateRepository.findBySagaId(event.orderId());
            if (state.getCurrentStep() != SagaStep.RESERVE_INVENTORY) return; // idempotencia

            state.advance(SagaStep.PROCESS_PAYMENT);
            sagaStateRepository.save(state);

            // Enviar Command al PaymentService
            streamBridge.send("process-payment-out-0",
                MessageBuilder.withPayload(new ProcessPaymentCommand(
                    event.orderId(), state.getUserId(), state.getAmount()))
                    .setHeader("sagaId", event.orderId())
                    .build());
        };
    }

    // Respuesta del InventoryService: reserva fallida
    @Bean
    public Consumer<InventoryFailedEvent> inventoryFailed() {
        return event -> {
            SagaState state = sagaStateRepository.findBySagaId(event.orderId());
            state.fail(SagaStep.RESERVE_INVENTORY, event.reason());
            sagaStateRepository.save(state);

            // No hay pasos anteriores que compensar → marcar pedido como fallido
            streamBridge.send("order-failed-out-0",
                MessageBuilder.withPayload(new OrderFailedEvent(
                    event.orderId(), "Inventory reservation failed: " + event.reason()))
                    .build());
        };
    }

    // Respuesta del PaymentService: pago exitoso
    @Bean
    public Consumer<PaymentProcessedEvent> paymentProcessed() {
        return event -> {
            SagaState state = sagaStateRepository.findBySagaId(event.orderId());
            if (state.getCurrentStep() != SagaStep.PROCESS_PAYMENT) return;

            state.complete();
            sagaStateRepository.save(state);

            // Saga completada exitosamente → confirmar pedido
            streamBridge.send("order-confirmed-out-0",
                MessageBuilder.withPayload(new OrderConfirmedEvent(event.orderId()))
                    .build());
        };
    }

    // Respuesta del PaymentService: pago fallido → compensar inventario
    @Bean
    public Consumer<PaymentFailedEvent> paymentFailed() {
        return event -> {
            SagaState state = sagaStateRepository.findBySagaId(event.orderId());
            state.compensating(SagaStep.RESERVE_INVENTORY);
            sagaStateRepository.save(state);

            // Enviar Command de compensación al InventoryService
            streamBridge.send("release-inventory-out-0",
                MessageBuilder.withPayload(new ReleaseInventoryCommand(event.orderId()))
                    .setHeader("compensating", true)
                    .build());
        };
    }

    // Compensación completada: inventario liberado
    @Bean
    public Consumer<InventoryReleasedEvent> inventoryReleased() {
        return event -> {
            SagaState state = sagaStateRepository.findBySagaId(event.orderId());
            state.fail(SagaStep.PROCESS_PAYMENT, "Payment failed, inventory compensated");
            sagaStateRepository.save(state);

            // Notificar que la Saga terminó con fallo
            streamBridge.send("order-failed-out-0",
                MessageBuilder.withPayload(new OrderFailedEvent(
                    event.orderId(), "Payment failed"))
                    .build());
        };
    }
}
```

### SagaState — persistencia del estado del flujo

```java
package com.example.saga;

import jakarta.persistence.*;

@Entity
@Table(name = "saga_state")
public class SagaState {

    @Id
    private String sagaId;
    private String userId;
    private double amount;

    @Enumerated(EnumType.STRING)
    private SagaStep currentStep;

    @Enumerated(EnumType.STRING)
    private SagaStatus status;  // STARTED, COMPLETED, FAILED, COMPENSATING

    private String failureReason;

    public static SagaState create(String sagaId, SagaStep initialStep) {
        SagaState state = new SagaState();
        state.sagaId = sagaId;
        state.currentStep = initialStep;
        state.status = SagaStatus.STARTED;
        return state;
    }

    public void advance(SagaStep nextStep) {
        this.currentStep = nextStep;
        this.status = SagaStatus.STARTED;
    }

    public void complete() {
        this.status = SagaStatus.COMPLETED;
    }

    public void fail(SagaStep failedStep, String reason) {
        this.failureReason = reason;
        this.status = SagaStatus.FAILED;
    }

    public void compensating(SagaStep stepToCompensate) {
        this.currentStep = stepToCompensate;
        this.status = SagaStatus.COMPENSATING;
    }

    public SagaStep getCurrentStep() { return currentStep; }
    public String getSagaId() { return sagaId; }
    public String getUserId() { return userId; }
    public double getAmount() { return amount; }
}

enum SagaStep { RESERVE_INVENTORY, PROCESS_PAYMENT, COMPLETED }
enum SagaStatus { STARTED, COMPLETED, FAILED, COMPENSATING }
```

```yaml
# application.yml del Saga Orchestrator
spring:
  cloud:
    stream:
      bindings:
        # Salidas hacia participantes
        reserve-inventory-out-0:
          destination: reserve-inventory-commands
        process-payment-out-0:
          destination: process-payment-commands
        release-inventory-out-0:
          destination: release-inventory-commands
        order-confirmed-out-0:
          destination: order-confirmed-events
        order-failed-out-0:
          destination: order-failed-events
        # Entradas desde participantes
        inventoryReserved-in-0:
          destination: inventory-reserved-events
          group: orchestrator-group
        inventoryFailed-in-0:
          destination: inventory-failed-events
          group: orchestrator-group
        paymentProcessed-in-0:
          destination: payment-processed-events
          group: orchestrator-group
        paymentFailed-in-0:
          destination: payment-failed-events
          group: orchestrator-group
        inventoryReleased-in-0:
          destination: inventory-released-events
          group: orchestrator-group
```

> [CONCEPTO] El `SagaState` persistido en base de datos es la fuente de verdad del estado de una Saga en curso. En caso de reinicio del orquestador, el estado persiste y las Sagas en progreso pueden reanudarse desde el último paso completado. La tabla `saga_state` es el corazón del orquestador y debe tener índices adecuados sobre `sagaId` y `status`.

## Tabla de elementos clave

| Elemento | Descripción |
|---|---|
| Saga Orchestrator | Componente central que coordina los pasos; conoce el flujo completo y el estado actual |
| `SagaState` | Entidad persistida que almacena el paso actual, el estado y el motivo de fallo de una Saga |
| Command → Participante | El orquestador envía Commands (no eventos); los participantes responden con eventos |
| Compensación ordenada | El orquestador sabe exactamente qué pasos compensar y en qué orden; ventaja sobre coreografía |
| `status: COMPENSATING` | Estado intermedio durante la ejecución de transacciones compensatorias |
| Idempotencia por `currentStep` | Verificar que el paso actual es el esperado antes de procesar; evita procesar respuestas duplicadas |

## Buenas y malas prácticas

**Hacer:**
- Persistir el `SagaState` en la misma base de datos que los datos del dominio, con el mismo schema de transacciones; si el orquestador cae entre el guardado del estado y el envío del Command, al reiniciar puede reenviar el Command desde el estado persistido.
- Implementar un proceso de "Saga monitor" que detecte Sagas en estado `STARTED` sin progreso durante más de X minutos y las marque como fallidas o las reintenta; las Sagas huérfanas por caída del broker son invisibles sin monitorización.
- Diseñar el orquestador para ser stateless en términos de memoria: todo el estado debe estar en la base de datos, no en variables de instancia; permite escalar horizontalmente el orquestador sin coordinación adicional.

**Evitar:**
- Usar el orquestador para lógica de negocio compleja (validaciones, cálculos); el orquestador debe ser un coordinador puro que secuencia Commands y reacciona a respuestas sin lógica de dominio.
- Compartir el topic de respuestas entre múltiples instancias del orquestador sin usar consumer groups: todas las instancias recibirían todas las respuestas de todos los sagaIds, no solo las de las Sagas que gestionan.
- Crear topics separados por cada tipo de respuesta cuando el número de tipos es muy alto; considerar un único topic de respuestas con un campo `eventType` y filtrar en el consumer del orquestador.

---

← [13.2 Saga de coreografía con Spring Cloud Stream](sc-patrones-saga-coreografia.md) | [Índice (README.md)](README.md) | [13.4 Outbox Pattern con Spring Cloud Stream](sc-patrones-outbox.md) →
