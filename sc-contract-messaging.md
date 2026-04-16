# 10.8 Contract Messaging — contratos de eventos

← [10.7 Stub Runner en el consumer](sc-contract-stub-runner.md) | [Índice](README.md) | [10.9 Repositorio de contratos y publicación de stubs](sc-contract-stubs-repo.md) →

---

## Introducción

Los contratos HTTP verifican APIs REST, pero en una arquitectura de microservicios orientada a eventos, el mayor riesgo de incompatibilidad está en los mensajes: un producer cambia el campo `orderId` por `order_id` en su evento Kafka y todos los consumers fallan en silencio porque no hay un test que verifique el contrato del mensaje. Spring Cloud Contract extiende el modelo CDC a la mensajería: permite definir contratos para eventos publicados en Kafka o RabbitMQ, generar tests en el producer que verifican que el evento cumple el contrato y usar el StubRunner en el consumer para simular la recepción del evento. Sin contratos de mensajería, las incompatibilidades de schema de eventos solo se detectan en integración tardía o en producción.

> [TAREA] El desarrollador necesita definir contratos de mensajería para garantizar la compatibilidad entre producers y consumers de eventos.

> [RESULTADO] Después de leer este fichero el desarrollador puede escribir un contrato con estructura `label/input/outputMessage`, configurar la clase base para tests de mensajería con `@AutoConfigureMessageVerifier`, y usar el StubRunner para disparar mensajes en el consumer y verificar que el handler los procesa correctamente.

> [PREREQUISITO] DSL base (10.4.1), clase base de tests (10.5) y noción básica de Spring Cloud Stream (bindings, binders).

## Representación visual

El siguiente diagrama muestra el flujo de verificación para un contrato de mensajería en ambos lados.

```
LADO PRODUCER (test generado por el plugin)
─────────────────────────────────────────────
[Contrato messaging .groovy]
  label: "orderCreated"
  input: triggerMessage("orderCreated")   ← label para disparar
  outputMessage:
    sentTo: "orders-out-0"
    body: { orderId: 42, status: "CREATED" }
         │
         ▼
[Test generado llama: contractVerifierMessaging.send("orderCreated")]
         │
         ▼
[Base class dispara el evento via OutputDestination / MessageChannel]
         │
         ▼
[Test verifica el mensaje en el canal de salida]

LADO CONSUMER (test con StubRunner)
─────────────────────────────────────────────
[StubRunner trigger: stubTrigger.trigger("orderCreated")]
         │
         ▼
[StubRunner publica el outputMessage del contrato en el canal del consumer]
         │
         ▼
[Handler del consumer recibe el mensaje y procesa]
         │
         ▼
[Test verifica el estado resultante en el consumer]
```

## Ejemplo central

El siguiente ejemplo completo cubre un contrato de mensajería, la clase base del producer y el test del consumer con StubRunner.

**Contrato de mensajería (src/test/resources/contracts/messaging/orderCreated.groovy):**

```groovy
import org.springframework.cloud.contract.spec.Contract

Contract.make {
    description "Producer publica evento OrderCreated al crear un pedido"
    label "orderCreated"                    // identificador para el trigger

    input {
        // Contrato disparado por un trigger programático (no por mensaje entrante)
        triggeredBy("publishOrderCreated()") // método en la clase base
    }

    outputMessage {
        sentTo "orders-out-0"               // nombre del binding de salida
        headers {
            header("contentType", "application/json")
        }
        body(
            orderId:   value(producer(42), consumer(42)),
            status:    value(producer("CREATED"), consumer("CREATED")),
            timestamp: $(producer(regex(iso8601WithOffset())), consumer("2025-01-15T10:00:00Z")),
            amount:    value(producer(99.99), consumer(99.99))
        )
        bodyMatchers {
            jsonPath("$.orderId", byType())
            jsonPath("$.timestamp", byRegex(iso8601WithOffset()))
        }
    }
}
```

**Contrato fuego-y-olvida (mensaje entrante sin salida):**

```groovy
Contract.make {
    description "Consumer recibe OrderCancelled y no produce output"
    label "orderCancelled"

    input {
        messageFrom "orders-in-0"
        messageBody(orderId: 42, reason: "OUT_OF_STOCK")
        messageHeaders { header("contentType", "application/json") }
    }
    // Sin outputMessage: contrato de tipo "fuego-y-olvida"
    // El test solo verifica que el producer procesa el mensaje sin errores
}
```

**Contrato HTTP que dispara mensaje de salida:**

```groovy
Contract.make {
    description "POST /orders dispara evento OrderCreated"
    request {
        method POST()
        url "/orders"
        body(productId: 10, quantity: 2)
        headers { contentType(applicationJson()) }
    }
    response {
        status CREATED()
        body(orderId: 42)
    }
    outputMessage {
        sentTo "orders-out-0"
        body(orderId: 42, status: "CREATED")
    }
}
```

**Clase base del producer para mensajería (src/test/java/com/example/BaseMessagingContractTest.java):**

```java
package com.example.contract;

import org.junit.jupiter.api.BeforeEach;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.cloud.contract.verifier.messaging.boot.AutoConfigureMessageVerifier;
import org.springframework.cloud.stream.binder.test.OutputDestination;
import org.springframework.cloud.stream.binder.test.TestChannelBinderConfiguration;
import org.springframework.context.annotation.Import;

import com.example.OrderService;

@SpringBootTest
@AutoConfigureMessageVerifier
@Import(TestChannelBinderConfiguration.class)  // binder de test de Spring Cloud Stream
public abstract class BaseMessagingContractTest {

    @Autowired
    private OrderService orderService;   // servicio que publica el evento

    // método referenciado en el contrato: triggeredBy("publishOrderCreated()")
    public void publishOrderCreated() {
        orderService.createOrder(42L);   // el servicio publica el evento
    }
}
```

**pom.xml — dependencias para messaging contracts:**

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-contract-verifier</artifactId>
        <scope>test</scope>
    </dependency>
    <!-- Binder de test para Spring Cloud Stream -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-stream-test-binder</artifactId>
        <scope>test</scope>
    </dependency>
    <!-- Binder real (Kafka o RabbitMQ) para el entorno de producción -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-stream-kafka</artifactId>
    </dependency>
</dependencies>
```

**application.yml del producer (test):**

```yaml
spring:
  cloud:
    stream:
      bindings:
        orders-out-0:
          destination: orders
      default:
        binder: test  # usa el test binder en tests de contrato
```

**Test del consumer usando StubRunner para trigger de mensajes:**

```java
@SpringBootTest
@AutoConfigureStubRunner(
    ids = "com.example:order-service:+:stubs",
    stubsMode = StubRunnerProperties.StubsMode.CLASSPATH
)
class OrderConsumerMessagingTest {

    @Autowired
    private StubTrigger stubTrigger;   // inyectado por @AutoConfigureStubRunner

    @Autowired
    private OrderRepository orderRepository;  // estado resultante del handler

    @Test
    void shouldProcessOrderCreatedEvent() {
        // Disparar el mensaje usando el label del contrato
        stubTrigger.trigger("orderCreated");

        // Verificar que el consumer procesó el mensaje correctamente
        assertThat(orderRepository.findById(42L))
            .isPresent()
            .hasValueSatisfying(order ->
                assertThat(order.getStatus()).isEqualTo("CREATED"));
    }
}
```

## Tabla de elementos clave

Los parámetros y anotaciones críticos para configurar contratos de mensajería.

| Elemento | Tipo | Descripción |
|---|---|---|
| `label` | String en contrato | Identificador del mensaje; usado por `stubTrigger.trigger(label)` en el consumer |
| `triggeredBy(method)` | DSL contrato | Método de la clase base que dispara el evento; el plugin lo llama en el test generado |
| `outputMessage.sentTo` | String | Nombre del binding de salida (Spring Cloud Stream) o canal donde se publica |
| `messageFrom` | String | Canal de entrada para contratos con mensaje entrante |
| `@AutoConfigureMessageVerifier` | Anotación | Activa la infraestructura de verificación de mensajes en la clase base |
| `TestChannelBinderConfiguration` | Clase | Binder de test de Spring Cloud Stream; reemplaza el binder real en tests |
| `StubTrigger` | Interfaz | Bean inyectable en el consumer; `trigger(label)` publica el mensaje del stub |
| `OutputDestination` | Clase | Permite leer mensajes publicados en el test del producer para verificarlos |
| `bodyMatchers` | Bloque DSL | Validaciones adicionales sobre el body del mensaje; funciona igual que en contratos HTTP |

## Buenas y malas prácticas

**Hacer:**

- Usar `TestChannelBinderConfiguration` de `spring-cloud-stream-test-binder` en la clase base del producer: evita arrancar un broker Kafka/RabbitMQ real en los tests de contrato, haciendo el test reproducible y rápido en CI.
- Nombrar el método de `triggeredBy()` de forma descriptiva (`publishOrderCreated`, `cancelOrder`): el test generado llama a ese método por reflexión, y un nombre genérico como `trigger()` hace imposible saber qué estado de negocio activa.
- Verificar en el test del consumer el estado resultante del handler (registros en BD, cambios de estado), no solo que el `StubTrigger.trigger()` no lanzó excepción: un trigger sin verificación de efecto es un test que nunca puede fallar.
- Añadir `bodyMatchers` con `byType()` sobre campos como `orderId` y `timestamp` para que el test del producer no falle cuando los valores concretos varían entre ejecuciones.

**Evitar:**

- No usar el binder real (Kafka) en tests de contrato del producer: arranca un broker real o requiere Testcontainers, aumentando el tiempo del test de milisegundos a decenas de segundos y creando dependencias de infraestructura en el ciclo de build.
- No omitir el `label` en el contrato de mensajería: sin label, el StubRunner no puede identificar qué mensaje disparar en el consumer y `stubTrigger.trigger("orderCreated")` lanza `IllegalArgumentException`.
- No mezclar contratos HTTP y de mensajería en el mismo directorio sin `baseClassMappings`: el plugin asignará la misma clase base a todos, y la clase base de mensajería (con `@AutoConfigureMessageVerifier`) puede fallar al procesar contratos HTTP.
- No asumir que el orden de mensajes está garantizado al disparar múltiples `stubTrigger.trigger()` en secuencia: el binder de test de Spring Cloud Stream no garantiza orden FIFO entre triggers en el mismo test.

## Comparación: contrato HTTP vs contrato de mensajería

Entender la diferencia de estructura ayuda a elegir el tipo de contrato correcto según el protocolo del servicio.

| Criterio | Contrato HTTP | Contrato de Mensajería |
|---|---|---|
| Disparador del test producer | Request HTTP real | `triggeredBy()` / `messageFrom` |
| Verificación del producer | Response HTTP | Mensaje en canal de salida |
| Simulación en consumer | WireMock HTTP server | `StubTrigger.trigger(label)` |
| Infraestructura de test | MockMvc / WebTestClient | TestChannelBinderConfiguration |
| Canal de transporte | HTTP/HTTPS | Kafka, RabbitMQ (via Stream) |

---

← [10.7 Stub Runner en el consumer](sc-contract-stub-runner.md) | [Índice](README.md) | [10.9 Repositorio de contratos y publicación de stubs](sc-contract-stubs-repo.md) →

