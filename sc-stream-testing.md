# 6.13 Testing / Verificación de Spring Cloud Stream

← [6.12 Actuator y monitorización de bindings](sc-stream-actuator.md) | [Índice (README.md)](README.md) | [7.1 Arquitectura de Spring Cloud Bus](sc-bus-arquitectura.md) →

## Introducción

Testear handlers de mensajería con un broker real (Kafka o RabbitMQ) introduce dependencias de infraestructura en los tests que ralentizan el ciclo de desarrollo y aumentan la fragilidad de la suite. Spring Cloud Stream resuelve este problema con `TestChannelBinderConfiguration`, un binder de prueba en memoria que reemplaza el binder real durante los tests. Con esta configuración, los tests no requieren ningún broker externo: los mensajes se envían y reciben a través de canales en memoria usando `InputDestination` y `OutputDestination`.

Este enfoque permite verificar la lógica de negocio del handler, la serialización del payload y las cabeceras del mensaje de forma completamente aislada. Para tests de integración que requieren garantías de interoperabilidad con un broker real, se puede complementar con Testcontainers (Kafka o RabbitMQ).

## Representación visual

El binder de test en memoria reemplaza el binder Kafka/RabbitMQ en el contexto de Spring durante la ejecución de tests. El diagrama muestra la arquitectura del test con `TestChannelBinderConfiguration`.

```
PRODUCCIÓN:
  Kafka/RabbitMQ ──► [binding-in-0] ──► @Bean Handler ──► [binding-out-0] ──► Kafka/RabbitMQ

TEST (con TestChannelBinderConfiguration):
  InputDestination.send()
         │
         ▼
  [binding-in-0 — canal en memoria]
         │
         ▼
  @Bean Handler (lógica real)
         │
         ▼
  [binding-out-0 — canal en memoria]
         │
         ▼
  OutputDestination.receive()

  ─── No hay Kafka ni RabbitMQ en los tests ───
```

## Ejemplo central

El siguiente ejemplo muestra un test completo de integración con `TestChannelBinderConfiguration` para verificar un handler `Function<I,O>`, incluyendo verificación del payload y de las cabeceras del mensaje de salida.

**pom.xml**
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-stream-binder-kafka</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
    <!-- Binder de prueba en memoria — siempre en scope test -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-stream</artifactId>
        <classifier>test-binder</classifier>
        <type>test-jar</type>
        <scope>test</scope>
    </dependency>
</dependencies>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>2025.1.1</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

**OrderEvent.java**
```java
package com.example.testing;

public record OrderEvent(String orderId, String customerId, double amount) {}
```

**OrderConfirmation.java**
```java
package com.example.testing;

public record OrderConfirmation(String orderId, String status, String processedBy) {}
```

**OrderProcessorConfig.java**
```java
package com.example.testing;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.Message;
import org.springframework.messaging.support.MessageBuilder;

import java.util.function.Function;

@Configuration
public class OrderProcessorConfig {

    @Bean
    public Function<Message<OrderEvent>, Message<OrderConfirmation>> processOrder() {
        return message -> {
            OrderEvent order = message.getPayload();

            if (order.amount() <= 0) {
                throw new IllegalArgumentException(
                    "Amount must be positive: " + order.amount());
            }

            String status = order.amount() > 1000 ? "PREMIUM" : "STANDARD";
            OrderConfirmation confirmation = new OrderConfirmation(
                order.orderId(), status, "order-processor-v1");

            return MessageBuilder
                .withPayload(confirmation)
                .setHeader("X-Processed-By", "order-processor-v1")
                .setHeader("X-Status", status)
                .copyHeaders(message.getHeaders())
                .build();
        };
    }
}
```

**application.yml (src/main/resources)**
```yaml
spring:
  cloud:
    function:
      definition: processOrder
    stream:
      bindings:
        processOrder-in-0:
          destination: orders
          group: order-processors
        processOrder-out-0:
          destination: orders-confirmed
      kafka:
        binder:
          brokers: localhost:9092
```

**OrderProcessorTest.java**
```java
package com.example.testing;

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

/**
 * Test de integración del handler processOrder.
 * @Import(TestChannelBinderConfiguration.class) reemplaza el binder Kafka por el binder en memoria.
 * No requiere ningún broker externo.
 */
@SpringBootTest
@Import(TestChannelBinderConfiguration.class)
class OrderProcessorTest {

    @Autowired
    private InputDestination inputDestination;

    @Autowired
    private OutputDestination outputDestination;

    @Test
    void whenOrderAmountIsPositive_thenConfirmationIsStandard() {
        // GIVEN: un mensaje de entrada con amount dentro del rango STANDARD
        OrderEvent order = new OrderEvent("ORD-001", "CUST-123", 500.0);
        Message<OrderEvent> inputMessage = MessageBuilder
            .withPayload(order)
            .setHeader("X-Correlation-Id", "test-correlation-123")
            .build();

        // WHEN: enviamos al binding de entrada del handler
        inputDestination.send(inputMessage, "orders");  // nombre del destination, no del binding

        // THEN: verificamos el mensaje de salida en el binding de salida
        Message<byte[]> outputMessage = outputDestination.receive(5000, "orders-confirmed");

        assertThat(outputMessage).isNotNull();

        // Verificar cabeceras
        assertThat(outputMessage.getHeaders().get("X-Processed-By"))
            .isEqualTo("order-processor-v1");
        assertThat(outputMessage.getHeaders().get("X-Status"))
            .isEqualTo("STANDARD");
    }

    @Test
    void whenOrderAmountExceedsThreshold_thenConfirmationIsPremium() {
        // GIVEN: un pedido con amount > 1000 (umbral PREMIUM)
        OrderEvent order = new OrderEvent("ORD-002", "CUST-456", 1500.0);
        Message<OrderEvent> inputMessage = MessageBuilder
            .withPayload(order)
            .build();

        // WHEN
        inputDestination.send(inputMessage, "orders");

        // THEN
        Message<byte[]> outputMessage = outputDestination.receive(5000, "orders-confirmed");

        assertThat(outputMessage).isNotNull();
        assertThat(outputMessage.getHeaders().get("X-Status")).isEqualTo("PREMIUM");
    }

    @Test
    void whenOrderAmountIsZero_thenNoMessageIsProduced() {
        // GIVEN: un pedido inválido (amount = 0)
        OrderEvent invalidOrder = new OrderEvent("ORD-003", "CUST-789", 0.0);
        Message<OrderEvent> inputMessage = MessageBuilder
            .withPayload(invalidOrder)
            .build();

        // WHEN: enviamos el mensaje inválido (el handler lanzará excepción)
        inputDestination.send(inputMessage, "orders");

        // THEN: no debe haber mensaje en el binding de salida
        // El timeout de 0ms retorna null inmediatamente si no hay mensaje
        Message<byte[]> outputMessage = outputDestination.receive(0, "orders-confirmed");

        assertThat(outputMessage).isNull();
    }
}
```

**TestingApplication.java**
```java
package com.example.testing;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class TestingApplication {
    public static void main(String[] args) {
        SpringApplication.run(TestingApplication.class, args);
    }
}
```

## Tabla de elementos clave

La API de testing de Spring Cloud Stream se reduce a tres clases principales. La tabla siguiente describe sus métodos más utilizados.

| Clase / Método | Descripción |
|---------------|-------------|
| `TestChannelBinderConfiguration` | Configuración de Spring que registra el binder en memoria. Se importa con `@Import(TestChannelBinderConfiguration.class)` en el test. |
| `InputDestination.send(Message<?>, String destination)` | Envía un mensaje al canal de entrada correspondiente al `destination` configurado en el binding. El segundo parámetro es el valor de `destination` en `application.yml`, no el nombre del binding. |
| `OutputDestination.receive(long timeout, String bindingName)` | Recibe el próximo mensaje del canal de salida del binding especificado. `timeout` en ms: `0` retorna inmediatamente, `>0` espera hasta que llega un mensaje. Retorna `null` si no hay mensaje en el timeout. |
| `OutputDestination.receive(long timeout)` | Versión sin `bindingName`: recibe del primer binding de salida disponible. Usar cuando hay un solo binding de salida. |
| `@SpringBootTest` | Arranque completo del contexto de Spring. Necesario para que el binder de test se configure correctamente con todos los beans del contexto. |
| `@Import(TestChannelBinderConfiguration.class)` | Reemplaza el binder real (Kafka/RabbitMQ) por el binder en memoria. Sin este import, `@SpringBootTest` intentará conectarse al broker real. |

## Buenas y malas prácticas

**Hacer:**

- Pasar el valor del `destination` (el nombre del topic, no el nombre del binding) como segundo parámetro de `InputDestination.send()` y `OutputDestination.receive()`. Es un error frecuente usar el nombre del binding (`processOrder-in-0`) en lugar del destination (`orders`).
- Verificar tanto el payload como las cabeceras en los tests. Las cabeceras transportan información de correlación, tracing y metadatos de negocio que son tan importantes como el payload para garantizar la correcta propagación del contexto en sistemas distribuidos.
- Usar `outputDestination.receive(0, bindingName)` con timeout 0 para verificar que NO se produce ningún mensaje en casos de error. Un timeout mayor introduce esperas innecesarias en la suite de tests.

**Evitar:**

- Usar `@SpringBootTest` sin `@Import(TestChannelBinderConfiguration.class)` cuando hay un binder real en el classpath. Sin el import, Spring Boot intentará conectarse al broker de producción durante los tests, fallando con un error de conexión.
- Hacer tests de integración con Testcontainers para verificar lógica de negocio que puede cubrirse con `TestChannelBinderConfiguration`. Testcontainers es costoso en tiempo de arranque (10-30 segundos por test suite). Reservarlo para tests de interoperabilidad con el broker (compresión, particionado real, DLQ real).

## Preguntas de entrevista

Las siguientes preguntas son frecuentes en entrevistas de nivel senior para posiciones que requieren experiencia con Spring Cloud Stream.

**P1: ¿Cuál es la diferencia entre `InputDestination.send(message, "orders")` y `InputDestination.send(message, "processOrder-in-0")`?**
El primer parámetro es el `destination` (nombre del topic en Kafka, nombre del exchange en RabbitMQ). El segundo sería el nombre del binding. `InputDestination.send()` acepta el `destination` como identificador del canal de test, no el nombre del binding.

**P2: ¿Por qué `OutputDestination.receive()` puede retornar `null` aunque el handler haya procesado el mensaje correctamente?**
Porque el timeout expiró antes de que el mensaje llegara al canal de salida. En tests, el procesamiento es síncrono pero puede haber latencia si el handler hace operaciones IO. Aumentar el timeout a 1000-5000ms resuelve el problema en la mayoría de casos.

**P3: ¿Cómo se testea un `Consumer<I>` (sin salida) con `TestChannelBinderConfiguration`?**
Se envía el mensaje con `InputDestination.send()` y se verifica el efecto secundario del consumer (llamada a un servicio mock, escritura en base de datos H2, etc.). No hay `OutputDestination.receive()` porque el consumer no produce mensajes.

**P4: ¿Qué ocurre con los errores de desserialización en tests con `TestChannelBinderConfiguration`?**
Con el binder de test, los mensajes se envían como `byte[]` internamente. Si el tipo del payload enviado no coincide con el tipo esperado por el handler, se lanzará una `MessageConversionException` que el binder de test propaga como una excepción en el hilo del test, no como un mensaje en la DLQ.

**P5: ¿Cómo se testea el comportamiento de la DLQ con `TestChannelBinderConfiguration`?**
El binder de test no tiene soporte nativo para DLQ. Para testear el comportamiento de DLQ, se debe usar Testcontainers con el binder Kafka/RabbitMQ real, o mockear el `ErrorMessageSendingRecoverer` que Spring Cloud Stream usa internamente para enviar mensajes fallidos a la DLQ.

---

← [6.12 Actuator y monitorización de bindings](sc-stream-actuator.md) | [Índice (README.md)](README.md) | [7.1 Arquitectura de Spring Cloud Bus](sc-bus-arquitectura.md) →
