# 13.9 Testing de patrones distribuidos con Spring Cloud

← [13.8 Resiliencia transversal en patrones distribuidos](sc-patrones-resiliencia-transversal.md) | [Índice (README.md)](README.md) →

---

Testear los patrones distribuidos (CQRS, Saga, Outbox, API Composition) es más complejo que testear un servicio individual porque el comportamiento correcto emerge de la interacción entre múltiples componentes. Un test unitario del servicio de pedidos no puede detectar que la Saga de coreografía queda bloqueada cuando el InventoryService publica un evento con un campo renombrado. Existen tres estrategias complementarias: tests de integración con TestChannelBinder que verifican el flujo de eventos de una Saga o Outbox en memoria, tests de contrato (CDC) que verifican la compatibilidad de los eventos entre servicios, y tests end-to-end con Testcontainers que verifican el flujo completo con brokers y bases de datos reales.

> [PREREQUISITO] Requiere `spring-cloud-stream-test-binder` para tests con broker simulado, `spring-cloud-starter-contract-verifier` para CDC de eventos, y `org.testcontainers:kafka` + `org.testcontainers:postgresql` para tests e2e.

## Estrategias de testing de patrones distribuidos

| Estrategia | Infraestructura | Velocidad | Qué verifica |
|---|---|---|---|
| 1 — Test unitario del step | Sin Spring | Muy rápida | Lógica del paso individual de Saga/Outbox |
| 2 — Test con TestChannelBinder | Broker simulado | Rápida | Flujo de eventos Saga/CQRS en memoria |
| 3 — CDC de eventos con Contract | Sin broker real | Media | Compatibilidad del schema de eventos entre servicios |
| 4 — E2E con Testcontainers | Docker real | Lenta | Flujo completo con Kafka, PostgreSQL y servicios reales |

## Ejemplo central: testing de Saga y Outbox con TestChannelBinder

### Estrategia 2: Test de flujo Saga con TestChannelBinder

```java
package com.example.saga;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.cloud.stream.binder.test.InputDestination;
import org.springframework.cloud.stream.binder.test.OutputDestination;
import org.springframework.cloud.stream.binder.test.TestChannelBinderConfiguration;
import org.springframework.context.annotation.Import;
import org.springframework.messaging.Message;
import org.springframework.messaging.support.MessageBuilder;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
@Import(TestChannelBinderConfiguration.class)
class OrderSagaFlowTest {

    @Autowired
    private InputDestination inputDestination;

    @Autowired
    private OutputDestination outputDestination;

    @Autowired
    private SagaStateRepository sagaStateRepository;

    @Test
    void whenInventoryReserved_thenOrchestratorSendsPaymentCommand() throws Exception {
        String orderId = "order-test-1";
        // Crear estado inicial de la Saga en BD de test
        sagaStateRepository.save(SagaState.create(orderId, SagaStep.RESERVE_INVENTORY));

        // Simular respuesta del InventoryService
        Message<byte[]> inventoryReservedMsg = MessageBuilder
            .withPayload("""
                {"orderId":"%s","productId":"p1","quantity":2}
                """.formatted(orderId).getBytes())
            .setHeader("eventType", "InventoryReserved")
            .build();

        inputDestination.send(inventoryReservedMsg, "inventory-reserved-events");

        // Verificar que el orquestador publicó el Command de pago
        Message<byte[]> paymentCommand = outputDestination
            .receive(1000, "process-payment-commands");

        assertThat(paymentCommand).isNotNull();
        String payloadStr = new String(paymentCommand.getPayload());
        assertThat(payloadStr).contains(orderId);

        // Verificar que el estado de la Saga avanzó
        SagaState state = sagaStateRepository.findBySagaId(orderId);
        assertThat(state.getCurrentStep()).isEqualTo(SagaStep.PROCESS_PAYMENT);
    }

    @Test
    void whenPaymentFails_thenOrchestratorTriggersCompensation() throws Exception {
        String orderId = "order-test-2";
        sagaStateRepository.save(SagaState.create(orderId, SagaStep.PROCESS_PAYMENT));

        // Simular fallo de pago
        Message<byte[]> paymentFailedMsg = MessageBuilder
            .withPayload("""
                {"orderId":"%s","reason":"Insufficient funds"}
                """.formatted(orderId).getBytes())
            .build();

        inputDestination.send(paymentFailedMsg, "payment-failed-events");

        // Verificar que se envió el Command de compensación de inventario
        Message<byte[]> releaseInventoryCmd = outputDestination
            .receive(1000, "release-inventory-commands");

        assertThat(releaseInventoryCmd).isNotNull();

        SagaState state = sagaStateRepository.findBySagaId(orderId);
        assertThat(state.getStatus()).isEqualTo(SagaStatus.COMPENSATING);
    }
}
```

### Estrategia 3: CDC para eventos entre servicios

```groovy
// contracts/events/order-created-event.groovy
// Contrato del evento que el OrderService publica en Kafka
import org.springframework.cloud.contract.spec.Contract

Contract.make {
    description "OrderService publica OrderCreatedEvent en Kafka"
    label "order-created"

    input {
        // El evento es generado por el producer (no hay request HTTP)
        triggeredBy("createOrder()")  // método que dispara el evento en el test
    }

    outputMessage {
        sentTo "order-created-events"  // topic Kafka
        body([
            orderId: $(consumer(regex('[a-f0-9\\-]+')), producer(value("order-123"))),
            productId: $(consumer(regex('[a-z0-9\\-]+')), producer(value("product-456"))),
            quantity: $(consumer(regex('[0-9]+')), producer(value(2))),
            userId: $(consumer(regex('[a-f0-9\\-]+')), producer(value("user-789")))
        ])
        headers {
            header("eventType", "OrderCreated")
            messagingContentType(applicationJson())
        }
    }
}
```

```java
// Clase base para el contrato de mensajería del OrderService
package com.example.saga;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.cloud.contract.verifier.messaging.boot.AutoConfigureMessageVerifier;

@SpringBootTest
@AutoConfigureMessageVerifier
public abstract class MessagingContractBase {

    @Autowired
    private OrderSagaParticipant orderSagaParticipant;

    // Este método es invocado por el test generado del contrato
    public void createOrder() {
        orderSagaParticipant.createOrder(new CreateOrderRequest(
            "order-123", "product-456", 2, "user-789"));
    }
}
```

### Estrategia 4: E2E del Outbox con Testcontainers

```java
package com.example.outbox;

import org.awaitility.Awaitility;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.testcontainers.service.connection.ServiceConnection;
import org.springframework.kafka.core.KafkaTemplate;
import org.testcontainers.containers.KafkaContainer;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;
import org.testcontainers.utility.DockerImageName;

import java.util.concurrent.TimeUnit;

@SpringBootTest
@Testcontainers
class OutboxPatternE2ETest {

    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>(DockerImageName.parse("postgres:16"));

    @Container
    @ServiceConnection
    static KafkaContainer kafka =
        new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.6.0"));

    @Autowired
    private OrderService orderService;

    @Autowired
    private OutboxRepository outboxRepository;

    @Autowired
    private TestKafkaConsumer kafkaConsumer;  // consumer de test que recibe mensajes

    @Test
    void whenOrderCreated_thenEventPublishedToKafka() throws Exception {
        // Crear pedido: escribe en orders + outbox en la misma transacción
        String orderId = orderService.createOrder(
            new CreateOrderRequest("p1", 1, "user-1")).getId();

        // Verificar que el evento está en outbox con status PENDING inicialmente
        assertThat(outboxRepository.findByAggregateId(orderId)).isNotEmpty();

        // Esperar a que el OutboxRelay publique el evento (polling cada 500ms)
        Awaitility.await()
            .atMost(5, TimeUnit.SECONDS)
            .untilAsserted(() -> {
                // Verificar que el consumer de test recibió el evento en Kafka
                assertThat(kafkaConsumer.getReceivedEvents())
                    .anyMatch(event -> event.contains(orderId));

                // Verificar que el outbox marcó el evento como SENT
                assertThat(outboxRepository.findByAggregateId(orderId).get(0).getStatus())
                    .isEqualTo(OutboxStatus.SENT);
            });
    }
}
```

> [CONCEPTO] El test con TestChannelBinder (Estrategia 2) verifica la lógica de orquestación de la Saga: si el orquestador recibe el evento correcto, ¿publica el Command correcto y actualiza el estado? Sin este test, la lógica de composición de eventos y la actualización del estado de la Saga solo se descubren en producción. El test E2E con Testcontainers (Estrategia 4) verifica que el Outbox relay realmente publica en Kafka — algo que el TestChannelBinder no puede verificar.

## Tabla de elementos clave

| Componente | Descripción |
|---|---|
| `TestChannelBinderConfiguration` | Binder en memoria para tests; reemplaza Kafka/Rabbit sin Docker |
| `InputDestination.send(msg, destination)` | Inyectar un mensaje al canal de entrada de un consumer de test |
| `OutputDestination.receive(timeout, destination)` | Leer el siguiente mensaje publicado por el servicio en un canal de test |
| `@AutoConfigureMessageVerifier` | Auto-configuración de Spring Cloud Contract para tests de mensajería |
| `triggeredBy("method()")` | En contratos de mensaje: el método que dispara el evento en el test del producer |
| `Awaitility.await().untilAsserted()` | Assertion asíncrona con timeout; necesaria para esperar la propagación en Kafka |
| `@ServiceConnection` con Testcontainers | Configura automáticamente las properties de conexión al contenedor; no requiere `@DynamicPropertySource` |

## Buenas y malas prácticas

**Hacer:**
- Testear tanto el happy path como los casos de compensación de la Saga con TestChannelBinder: el happy path es fácil de verificar; los bugs más críticos están en las rutas de compensación que solo se activan en producción cuando algo falla.
- Usar contratos CDC para los eventos de Saga/Outbox además de para las APIs REST: el renaming de un campo en el evento `OrderCreatedEvent` es un breaking change tan crítico como renombrar un campo de respuesta REST.
- Definir un consumer de test (`TestKafkaConsumer`) reutilizable en los tests e2e para Kafka: consumir y almacenar mensajes recibidos de forma thread-safe, con método de espera con timeout.

**Evitar:**
- Testear la Saga solo con tests unitarios del orquestador sin testear la interacción con los participants; los bugs de integración entre orquestador y participantes solo son detectables con tests que involucran el flujo de mensajes.
- Usar `Thread.sleep()` en lugar de `Awaitility` en tests con Kafka real; el sleep tiene una duración fija que puede ser insuficiente en CI con carga y excesiva en máquinas rápidas.
- Compartir el estado del `TestKafkaConsumer` entre tests en la misma clase de test sin limpiarlo en `@BeforeEach`; los mensajes de un test pueden contaminar las assertions del siguiente.

---

← [13.8 Resiliencia transversal en patrones distribuidos](sc-patrones-resiliencia-transversal.md) | [Índice (README.md)](README.md) →
