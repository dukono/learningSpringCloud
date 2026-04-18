# 6.14 Spring Cloud Stream — Testing con TestChannelBinder

← [6.13 Escenarios de examen](sc-stream-escenarios-examen.md) | [Índice](README.md) | [7.1 Spring Cloud Bus — Arquitectura](sc-bus-arquitectura.md) →

---

## Introducción

El TestChannelBinder es el mecanismo oficial de Spring Cloud Stream para testear aplicaciones de mensajería sin necesidad de un broker real. Resuelve el problema de hacer tests de integración deterministas y rápidos para `Function`, `Consumer` y `Supplier` beans sin infraestructura externa. Existe porque las pruebas que dependen de Kafka o RabbitMQ son lentas, no deterministas y complejas de configurar en CI/CD. Se necesita en cualquier suite de tests de un microservicio Spring Cloud Stream para verificar la lógica de procesamiento de mensajes de forma aislada.

## Arquitectura del test binder

El TestChannelBinder reemplaza todos los binders reales con un binder en memoria. `InputDestination` permite inyectar mensajes directamente en el binding de entrada; `OutputDestination` permite capturar los mensajes producidos en el binding de salida:

```
@Test method
    │
  InputDestination.send(message, "processOrder-in-0")
    │
  TestChannelBinder (en memoria)
    │
  processOrder bean (Function/Consumer)
    │
  TestChannelBinder (en memoria)
    │
  OutputDestination.receive(timeout, "processOrder-out-0")
    │
  Assertions sobre payload y headers
```

## Ejemplo central — tests de Function, Consumer y Supplier

El siguiente ejemplo muestra tests completos para los tres tipos de beans funcionales usando `TestChannelBinderConfiguration`:

```java
package com.example.stream;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.cloud.stream.binder.test.InputDestination;
import org.springframework.cloud.stream.binder.test.OutputDestination;
import org.springframework.cloud.stream.binder.test.TestChannelBinderConfiguration;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Import;
import org.springframework.messaging.Message;
import org.springframework.messaging.support.GenericMessage;
import org.springframework.messaging.support.MessageBuilder;
import java.util.function.Consumer;
import java.util.function.Function;
import java.util.function.Supplier;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
@Import(TestChannelBinderConfiguration.class)
class StreamProcessorTest {

    @Autowired
    private InputDestination inputDestination;

    @Autowired
    private OutputDestination outputDestination;

    // TEST 1: Function<String, String> — entrada y salida
    @Test
    void testProcessOrderFunction() {
        // Arrange: crear mensaje de entrada con payload String
        Message<byte[]> inputMessage = MessageBuilder
            .withPayload("ORDER-123".getBytes())
            .setHeader("contentType", "text/plain")
            .build();

        // Act: inyectar el mensaje en el binding de entrada
        inputDestination.send(inputMessage, "processOrder-in-0");

        // Assert: capturar el mensaje de salida del binding de salida
        Message<byte[]> outputMessage = outputDestination.receive(1000, "processOrder-out-0");

        assertThat(outputMessage).isNotNull();
        assertThat(new String(outputMessage.getPayload())).isEqualTo("PROCESSED:ORDER-123");
    }

    // TEST 2: Consumer<String> — solo entrada, sin salida
    @Test
    void testHandleOrderConsumer() {
        // Arrange
        Message<byte[]> inputMessage = new GenericMessage<>("ORDER-456".getBytes());

        // Act: el consumer no tiene salida, se verifica el efecto secundario
        inputDestination.send(inputMessage, "handleOrder-in-0");

        // Assert: no hay mensaje en el canal de salida (Consumer no produce)
        Message<byte[]> noOutput = outputDestination.receive(100, "handleOrder-out-0");
        assertThat(noOutput).isNull();
        // Verificar efecto secundario (repositorio, log, etc.) con mocks adicionales
    }

    // TEST 3: Function con JSON payload — deserialización automática
    @Test
    void testProcessOrderWithJson() {
        // Arrange: payload JSON que Spring Cloud Stream deserializará a POJO
        String jsonPayload = "{\"id\":\"ORD-789\",\"amount\":99.99}";
        Message<byte[]> inputMessage = MessageBuilder
            .withPayload(jsonPayload.getBytes())
            .setHeader("contentType", "application/json")
            .build();

        // Act
        inputDestination.send(inputMessage, "processOrder-in-0");

        // Assert
        Message<byte[]> result = outputDestination.receive(1000, "processOrder-out-0");
        assertThat(result).isNotNull();
        String resultPayload = new String(result.getPayload());
        assertThat(resultPayload).contains("ORD-789");
    }
}

// Configuración de la aplicación bajo test
@SpringBootTest
@Import(TestChannelBinderConfiguration.class)
class StreamApplicationConfig {

    @Configuration
    static class TestConfig {

        @Bean
        public Function<String, String> processOrder() {
            return order -> "PROCESSED:" + order;
        }

        @Bean
        public Consumer<String> handleOrder() {
            return order -> System.out.println("Handling: " + order);
        }
    }
}
```

```xml
<!-- pom.xml — dependencia de test -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-stream-test-binder</artifactId>
    <scope>test</scope>
</dependency>
```

```yaml
# application.yml — configuración de bindings para tests
# El TestChannelBinder ignora la configuración del broker real
spring:
  cloud:
    function:
      definition: processOrder
    stream:
      bindings:
        processOrder-in-0:
          destination: orders-topic
          group: order-service
          content-type: application/json
        processOrder-out-0:
          destination: processed-orders
```

## Tabla de componentes del test binder

| Componente | Rol | Método clave |
|------------|-----|-------------|
| `TestChannelBinderConfiguration` | Reemplaza binders reales en el contexto de test | `@Import(TestChannelBinderConfiguration.class)` |
| `InputDestination` | Inyecta mensajes en el binding de entrada | `send(Message<?>, String bindingName)` |
| `OutputDestination` | Captura mensajes del binding de salida | `receive(long timeout, String bindingName)` |
| `spring-cloud-stream-test-binder` | Artefacto Maven (scope test) | Incluye automáticamente TestChannelBinderConfiguration |

> [CONCEPTO] `InputDestination.send(message, bindingName)` requiere el nombre del binding (ej: `processOrder-in-0`), no el nombre del destino (ej: `orders-topic`). El segundo argumento es el nombre del binding derivado del bean funcional.

> [CONCEPTO] `OutputDestination.receive(timeout, bindingName)` acepta un timeout en milisegundos y el nombre del binding de salida. Si no llega ningún mensaje en el timeout, devuelve `null`. Es importante usar tiempos de espera razonables (100-1000ms para tests unitarios) para no hacer los tests lentos.

> [EXAMEN] `TestChannelBinderConfiguration` se importa con `@Import` en la clase de test. Con la dependencia `spring-cloud-stream-test-binder`, la configuración puede también autodetectarse si se usa `@SpringBootTest`. En ambos casos, todos los binders reales (Kafka, RabbitMQ) son sustituidos por el binder en memoria.

> [ADVERTENCIA] El test binder no reemplaza solo un binder: reemplaza todos los binders de la aplicación. Esto significa que no se pueden hacer tests de integración real con broker + test binder simultáneamente en el mismo contexto de Spring.

## Comparación — tipos de tests con Spring Cloud Stream

| Tipo de test | Herramienta | Broker real | Velocidad |
|-------------|-------------|-------------|-----------|
| Test unitario (sin Spring) | Mocks de función Java | No | Muy rápido |
| Test de integración (Spring) | TestChannelBinder + @SpringBootTest | No | Rápido |
| Test de contrato | Spring Cloud Contract | No (stubs) | Medio |
| Test de integración real | Testcontainers + Kafka/Rabbit | Sí | Lento |

## Buenas y malas prácticas

**Buenas prácticas:**
- Usar `spring-cloud-stream-test-binder` con scope `test` en todos los proyectos Spring Cloud Stream.
- Probar la lógica de negocio pura de los beans funcionales con tests unitarios sin Spring (más rápidos).
- Usar `TestChannelBinderConfiguration` para verificar la integración del binding, serialización y cabeceras.

**Malas prácticas:**
- Usar el nombre del destino (`orders-topic`) en lugar del nombre del binding (`processOrder-in-0`) en `InputDestination.send()`.
- No verificar el valor de retorno de `OutputDestination.receive()` (puede ser `null` si el test falla silenciosamente).
- Usar `Testcontainers` con Kafka en tests unitarios cuando el test binder es suficiente.

## Verificación y práctica

1. ¿Cómo se verifica en un test de Spring Cloud Stream que un `Function<String, String>` llamado `transform` recibe el mensaje `"hello"` y produce `"HELLO"` en el binding de salida?

2. ¿Qué parámetro requiere `InputDestination.send()` como segundo argumento: el nombre del binding o el nombre del `destination`?

3. ¿Qué ocurre si `OutputDestination.receive(1000, "myBean-out-0")` devuelve `null` en el test?

4. ¿Con qué anotación y configuración se activa el TestChannelBinder en un test `@SpringBootTest`?

5. ¿Por qué no se puede combinar `TestChannelBinderConfiguration` con un broker Kafka real en el mismo contexto de Spring?

---

← [6.13 Escenarios de examen](sc-stream-escenarios-examen.md) | [Índice](README.md) | [7.1 Spring Cloud Bus — Arquitectura](sc-bus-arquitectura.md) →
