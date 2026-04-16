# 6.1 Modelo de programación funcional

← [5.14 Testing del Circuit Breaker, Retry y Bulkhead](sc-circuitbreaker-testing.md) | [Índice (README.md)](README.md) | [6.2 Bindings y binders](sc-stream-bindings.md) →

## Introducción

Spring Cloud Stream adopta a partir de la versión 3.x el modelo de programación funcional de Spring Cloud Function como mecanismo primario para definir handlers de mensajes. El problema que resuelve es la acoplación entre la lógica de negocio y la infraestructura de mensajería: en versiones anteriores, un desarrollador tenía que anotar métodos con `@StreamListener` y gestionar manualmente canales de entrada y salida. Con el modelo funcional, basta con declarar un `@Bean` de tipo `Function<I,O>`, `Consumer<I>` o `Supplier<O>` en el contexto de Spring para que el framework infiera automáticamente los bindings de entrada y salida, conectándolos al broker subyacente (Kafka o RabbitMQ) sin ningún código adicional de infraestructura.

> [PREREQUISITO] Se asume conocimiento de lambdas y programación funcional en Java 21 (interfaces funcionales, referencias a métodos, composición). Sin esta base, el modelo resulta opaco.

Este módulo es necesario en cualquier arquitectura de microservicios orientada a eventos donde los servicios reaccionan a mensajes de forma continua. No es necesario si el servicio solo produce mensajes de forma imperativa desde un REST controller; en ese caso, `StreamBridge` (6.10) es la herramienta adecuada.

## Representación visual

El modelo funcional sitúa la lógica de negocio en el centro y delega la conexión con el broker al framework. El siguiente diagrama muestra cómo un `@Bean` funcional se enlaza automáticamente con los topics del broker a través del sistema de bindings.

```
Broker (Kafka / RabbitMQ)
        │
        │  topic: orders-in  (destination)
        ▼
┌─────────────────────────────┐
│  Binding: processOrder-in-0 │  ← creado automáticamente por Spring Cloud Stream
└────────────┬────────────────┘
             │  Message<OrderEvent>
             ▼
┌─────────────────────────────┐
│  @Bean Function<OrderEvent, │
│         OrderConfirmation>  │  ← lógica de negocio pura (POJO)
│  processOrder               │
└────────────┬────────────────┘
             │  Message<OrderConfirmation>
             ▼
┌─────────────────────────────┐
│  Binding: processOrder-out-0│  ← creado automáticamente por Spring Cloud Stream
└────────────┬────────────────┘
             │
             ▼  topic: orders-out  (destination)
        Broker (Kafka / RabbitMQ)
```

El naming convention es determinista: para un bean llamado `processOrder`, Spring Cloud Stream genera `processOrder-in-0` para el canal de entrada y `processOrder-out-0` para el de salida. El sufijo numérico (`-0`) permite composición de múltiples entradas/salidas.

## Ejemplo central

El siguiente ejemplo implementa tres tipos de handler funcional en un solo servicio Spring Boot 4.0.x. El proyecto recibe eventos de pedido, los procesa, emite confirmaciones y genera periódicamente notificaciones de estado.

**pom.xml**
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-stream</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-stream-binder-kafka</artifactId>
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

**application.yml**
```yaml
spring:
  cloud:
    function:
      definition: processOrder;notifyShipment;statusSupplier
    stream:
      bindings:
        processOrder-in-0:
          destination: orders-raw
          group: order-processor
        processOrder-out-0:
          destination: orders-confirmed
        notifyShipment-in-0:
          destination: shipments
          group: shipment-notifier
        statusSupplier-out-0:
          destination: status-events
      kafka:
        binder:
          brokers: localhost:9092
```

**OrderEvent.java**
```java
package com.example.stream;

public record OrderEvent(String orderId, String customerId, double amount) {}
```

**OrderConfirmation.java**
```java
package com.example.stream;

public record OrderConfirmation(String orderId, String status, long processedAt) {}
```

**OrderStreamConfig.java**
```java
package com.example.stream;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.Message;
import org.springframework.messaging.MessageHeaders;
import org.springframework.messaging.support.MessageBuilder;

import java.time.Instant;
import java.util.function.Consumer;
import java.util.function.Function;
import java.util.function.Supplier;

@Configuration
public class OrderStreamConfig {

    /**
     * Function<I,O>: recibe un OrderEvent y produce una OrderConfirmation.
     * Spring Cloud Stream infiere:
     *   - entrada: processOrder-in-0
     *   - salida:  processOrder-out-0
     */
    @Bean
    public Function<Message<OrderEvent>, Message<OrderConfirmation>> processOrder() {
        return message -> {
            OrderEvent event = message.getPayload();
            MessageHeaders headers = message.getHeaders();

            // Acceso a cabeceras del mensaje entrante
            String correlationId = headers.getOrDefault("X-Correlation-Id", "unknown").toString();

            OrderConfirmation confirmation = new OrderConfirmation(
                event.orderId(),
                event.amount() > 0 ? "ACCEPTED" : "REJECTED",
                Instant.now().toEpochMilli()
            );

            return MessageBuilder
                .withPayload(confirmation)
                .setHeader("X-Correlation-Id", correlationId)
                .setHeader("X-Processed-By", "order-processor")
                .build();
        };
    }

    /**
     * Consumer<I>: recibe un evento de envío y no produce salida.
     * Spring Cloud Stream infiere:
     *   - entrada: notifyShipment-in-0
     *   - (sin salida)
     */
    @Bean
    public Consumer<Message<String>> notifyShipment() {
        return message -> {
            String shipmentId = message.getPayload();
            String source = (String) message.getHeaders().getOrDefault("source", "unknown");
            System.out.printf("Shipment %s received from %s%n", shipmentId, source);
        };
    }

    /**
     * Supplier<O>: produce mensajes periódicamente (cada 1 segundo por defecto).
     * Spring Cloud Stream infiere:
     *   - (sin entrada)
     *   - salida: statusSupplier-out-0
     */
    @Bean
    public Supplier<String> statusSupplier() {
        return () -> "SERVICE_OK:" + Instant.now().toEpochMilli();
    }
}
```

**StreamApplication.java**
```java
package com.example.stream;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class StreamApplication {
    public static void main(String[] args) {
        SpringApplication.run(StreamApplication.class, args);
    }
}
```

## Tabla de elementos clave

La siguiente tabla resume los parámetros de configuración que gobiernan el modelo funcional en Spring Cloud Stream. Un profesional senior debe conocerlos de memoria para entrevistas y diagnósticos en producción.

| Parámetro | Tipo | Default | Descripción |
|-----------|------|---------|-------------|
| `spring.cloud.function.definition` | `String` | (ninguno) | Lista de nombres de beans funcionales separados por `;`. Obligatorio si hay más de un bean Function/Consumer/Supplier en el contexto. |
| `spring.cloud.stream.bindings.<name>-in-0.destination` | `String` | nombre del binding | Nombre del topic/queue de entrada en el broker. |
| `spring.cloud.stream.bindings.<name>-out-0.destination` | `String` | nombre del binding | Nombre del topic/queue de salida en el broker. |
| `spring.cloud.stream.bindings.<name>.group` | `String` | `null` (anónimo) | Consumer group. Sin group, cada instancia crea su propia queue anónima (fan-out). Con group, las instancias comparten la cola (competing consumers). |
| `spring.cloud.stream.bindings.<name>.contentType` | `String` | `application/json` | MIME type para serialización/deserialización automática del payload. |
| `spring.cloud.function.definition` (composición) | `String` | — | Permite componer funciones con `\|`: `definition: filterOrders\|processOrder` encadena dos beans. |
| `spring.cloud.stream.poller.fixed-delay` | `long` (ms) | `1000` | Intervalo de polling para `Supplier<O>`. Controla cada cuánto se invoca el supplier para generar mensajes. |
| `spring.cloud.stream.poller.max-messages-per-poll` | `int` | `1` | Número máximo de mensajes generados por invocación del poller del Supplier. |

## Buenas y malas prácticas

**Hacer:**

- Declarar `spring.cloud.function.definition` explícitamente aunque solo haya un bean funcional. En producción con múltiples beans en el classpath (tests, configuraciones condicionales), la detección automática puede fallar y generar un `BeanDefinitionOverrideException` de difícil diagnóstico.
- Usar `Message<T>` como parámetro de entrada en `Function` y `Consumer` cuando se necesite acceder a cabeceras. Las cabeceras transportan correlation IDs, tracing context y metadatos del broker que son esenciales para observabilidad distribuida.
- Hacer los beans funcionales stateless y sin efectos secundarios directos. El estado compartido en un handler de mensajes bajo concurrencia produce race conditions que solo se manifiestan bajo carga.
- Definir un `consumer group` en todos los bindings de entrada. Sin group, cada reinicio de la aplicación crea una nueva suscripción anónima y pierde todos los mensajes que llegaron mientras la instancia estaba caída.

**Evitar:**

- Usar `@StreamListener` (deprecated). En Spring Cloud Stream 4.x, `@StreamListener` fue eliminado. El código legado con esta anotación no compila con la versión actual.
- Poner lógica de negocio compleja directamente en el lambda del `@Bean`. Los lambdas inline en configuración son imposibles de testear unitariamente. Extraer la lógica a un servicio inyectable y delegar desde el bean funcional.
- Mezclar `Function<Flux<T>, Flux<R>>` (modo reactivo) con `Function<T,R>` (modo imperativo) en el mismo servicio sin entender las diferencias de threading. El modo reactivo desactiva el acknowledgement automático y requiere gestión explícita del error channel.
- Ignorar el `spring.cloud.stream.poller.fixed-delay` de los `Supplier`. El valor por defecto de 1 segundo puede generar cientos de miles de mensajes vacíos por día si el supplier retorna `null` en lugar de omitir la emisión.

## Comparación de tipos funcionales

Los tres tipos de bean funcional cubren casos de uso distintos. La elección incorrecta genera configuraciones innecesariamente complejas o introduce bindings no deseados en el broker.

| Tipo | Signature | Bindings creados | Caso de uso |
|------|-----------|------------------|-------------|
| `Function<I,O>` | `Function<OrderEvent, Confirmation>` | 1 entrada + 1 salida | Transformación/enriquecimiento: recibe evento, procesa y emite resultado |
| `Consumer<I>` | `Consumer<ShipmentEvent>` | 1 entrada + 0 salidas | Sink: persistir en BD, enviar email, actualizar estado — sin respuesta al broker |
| `Supplier<O>` | `Supplier<StatusEvent>` | 0 entradas + 1 salida | Source: generar eventos periódicos (health beats, reportes, CDC polling) |
| `Function<Flux<I>,Flux<O>>` | `Function<Flux<Order>,Flux<Conf>>` | 1 entrada + 1 salida | Transformación reactiva con backpressure nativo de Project Reactor |

> [ADVERTENCIA] `Supplier<O>` se invoca por un poller en un thread separado. Si el supplier accede a recursos compartidos sin sincronización (caché, contadores), se producen race conditions bajo múltiples instancias del servicio.

> [EXAMEN] Pregunta frecuente en entrevistas senior: "¿Qué ocurre si tienes dos beans de tipo `Consumer` en el contexto sin configurar `spring.cloud.function.definition`?" Respuesta: Spring Cloud Stream lanza una excepción en el arranque porque no puede determinar cuál de los dos consumers enlazar al binding por defecto. La propiedad `definition` es obligatoria cuando hay ambigüedad.

---

← [5.14 Testing del Circuit Breaker, Retry y Bulkhead](sc-circuitbreaker-testing.md) | [Índice (README.md)](README.md) | [6.2 Bindings y binders](sc-stream-bindings.md) →
