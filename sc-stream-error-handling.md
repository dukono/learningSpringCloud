# 6.8 Gestión de errores y DLQ

← [6.7 Particionado de mensajes](sc-stream-particionado.md) | [Índice (README.md)](README.md) | [6.9 Polling y batch consumers](sc-stream-polling-batch.md) →

## Introducción

La gestión de errores en Spring Cloud Stream resuelve uno de los problemas más críticos de los sistemas de mensajería en producción: qué hacer con un mensaje que el handler no puede procesar. Sin una estrategia de errores, el primer mensaje inválido puede bloquear una partición entera de Kafka indefinidamente o saturar una queue de RabbitMQ con reintentos infinitos, deteniendo el procesamiento de todos los mensajes subsiguientes.

Spring Cloud Stream ofrece una cadena de gestión de errores en capas: primero, reintentos con backoff exponencial (gestionados por un `RetryTemplate`); segundo, si los reintentos se agotan, envío del mensaje a una Dead Letter Queue (DLQ); tercero, un error channel donde el desarrollador puede escribir lógica de manejo personalizado. El mecanismo de DLQ y su configuración difieren entre el binder Kafka y el binder RabbitMQ, por lo que este fichero cubre ambos.

> [PREREQUISITO] Binder Kafka (6.3) y Binder RabbitMQ (6.5). Las propiedades `enableDlq` (Kafka) y `auto-bind-dlq`/`requeue-rejected` (RabbitMQ) son específicas del binder.

## Representación visual

El flujo de gestión de errores sigue una secuencia de escalada. El diagrama muestra el recorrido de un mensaje fallido a través de las capas.

```
Mensaje recibido del broker
        │
        ▼
┌───────────────────────────────────┐
│  Handler funcional (@Bean)        │
│  Function<I,O> / Consumer<I>      │
└────────────┬──────────────────────┘
             │ EXCEPCIÓN lanzada
             ▼
┌───────────────────────────────────┐
│  RetryTemplate                    │
│  maxAttempts: 3 (configurable)    │
│  backoff: 500ms × 2.0 (hasta 5s) │
│                                   │
│  ¿retry agotado?                  │
└──────────┬──────────────┬─────────┘
  NO: retry │              │ SÍ: agotado
            │              ▼
            │   ┌─────────────────────────┐
            │   │  ¿DLQ habilitada?        │
            │   └──────┬──────────────────┘
            │     SÍ   │          NO
            │          ▼          ▼
            │   ┌──────────┐  ┌────────────────┐
            │   │ DLQ topic │  │ Error channel  │
            │   │ (Kafka:   │  │ (descarta o    │
            │   │  .dlq)    │  │  @ServiceActivator) │
            │   └──────────┘  └────────────────┘
```

## Ejemplo central

El siguiente ejemplo configura la gestión de errores completa para Kafka y RabbitMQ: reintentos con backoff, DLQ habilitada, y un `@ServiceActivator` que monitoriza el error channel para mensajes con reintentos agotados.

**pom.xml**
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-stream-binder-kafka</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.integration</groupId>
        <artifactId>spring-integration-core</artifactId>
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
      definition: processOrder
    stream:
      bindings:
        processOrder-in-0:
          destination: orders
          group: order-processors
          consumer:
            max-attempts: 3                       # reintentos máximos antes de DLQ
            back-off-initial-interval: 500         # ms de espera antes del primer reintento
            back-off-multiplier: 2.0              # factor de incremento del intervalo
            back-off-max-interval: 10000          # ms máximo de backoff (cap)
            default-retryable: true               # todas las excepciones disparan retry por defecto
            retryable-exceptions:                 # lista explícita (complementa default-retryable)
              java.lang.RuntimeException: true
              java.lang.IllegalArgumentException: false  # NO reintentar esta excepción
        processOrder-out-0:
          destination: orders-processed
      kafka:
        binder:
          brokers: localhost:9092
        bindings:
          processOrder-in-0:
            consumer:
              enable-dlq: true                    # habilitar DLQ
              dlq-name: orders.dlq                # nombre explícito del topic DLQ
              dlq-partitions: 3                   # particiones en la DLQ
```

**OrderEvent.java**
```java
package com.example.errors;

public record OrderEvent(String orderId, String type, double amount) {}
```

**ErrorHandlingConfig.java**
```java
package com.example.errors;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.integration.annotation.ServiceActivator;
import org.springframework.messaging.Message;
import org.springframework.messaging.MessageHandlingException;
import org.springframework.messaging.support.MessageBuilder;

import java.util.function.Consumer;

@Configuration
public class ErrorHandlingConfig {

    /**
     * Handler principal. Lanza excepción para mensajes con amount <= 0.
     * Con maxAttempts=3 y backoff, se reintentará 3 veces antes de ir a la DLQ.
     * IllegalArgumentException tiene retryable=false: va directamente a DLQ sin reintentos.
     */
    @Bean
    public Consumer<Message<OrderEvent>> processOrder() {
        return message -> {
            OrderEvent order = message.getPayload();

            if (order.amount() <= 0) {
                // No retryable: va directo a DLQ sin reintentos
                throw new IllegalArgumentException(
                    "Invalid amount for order: " + order.orderId());
            }

            if ("FRAUD_SUSPECT".equals(order.type())) {
                // Retryable: se reintentará maxAttempts veces antes de ir a DLQ
                throw new RuntimeException("Fraud check service unavailable");
            }

            System.out.printf("[processOrder] OK: %s amount=%.2f%n",
                order.orderId(), order.amount());
        };
    }

    /**
     * Error channel handler: se invoca cuando un mensaje ha agotado todos los reintentos
     * y NO tiene DLQ habilitada, o cuando se quiere lógica adicional de monitorización.
     * El nombre del canal es: <bindingName>.errors
     *
     * NOTA: Con enableDlq=true en Kafka, el framework ya envía a DLQ automáticamente.
     * Este @ServiceActivator es complementario para alertas/logging antes de la DLQ.
     */
    @ServiceActivator(inputChannel = "orders.order-processors.errors")
    public void handleError(Message<MessageHandlingException> errorMessage) {
        MessageHandlingException exception = errorMessage.getPayload();
        Message<?> failedMessage = exception.getFailedMessage();

        System.err.printf("[ERROR_CHANNEL] Message failed after retries: %s | Cause: %s%n",
            failedMessage != null ? failedMessage.getPayload() : "unknown",
            exception.getCause() != null ? exception.getCause().getMessage() : "unknown");

        // Aquí se podría: enviar alerta a PagerDuty/Slack, escribir en BD de auditoría, etc.
    }
}
```

**ErrorHandlingApplication.java**
```java
package com.example.errors;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class ErrorHandlingApplication {
    public static void main(String[] args) {
        SpringApplication.run(ErrorHandlingApplication.class, args);
    }
}
```

## Tabla de elementos clave

La gestión de errores se configura en dos niveles: propiedades comunes del binding (para la política de retry) y propiedades del binder específico (para la DLQ). La tabla siguiente cubre las más relevantes.

| Parámetro | Tipo | Default | Descripción |
|-----------|------|---------|-------------|
| `spring.cloud.stream.bindings.<name>.consumer.max-attempts` | `int` | `3` | Número máximo de intentos de procesamiento antes de enviar a DLQ. `1` = sin reintentos. |
| `spring.cloud.stream.bindings.<name>.consumer.back-off-initial-interval` | `long` (ms) | `1000` | Tiempo de espera antes del primer reintento. |
| `spring.cloud.stream.bindings.<name>.consumer.back-off-multiplier` | `double` | `2.0` | Factor de incremento del intervalo entre reintentos sucesivos. |
| `spring.cloud.stream.bindings.<name>.consumer.back-off-max-interval` | `long` (ms) | `10000` | Intervalo máximo de backoff. Cap para evitar esperas de minutos. |
| `spring.cloud.stream.bindings.<name>.consumer.default-retryable` | `boolean` | `true` | Si `true`, todas las excepciones disparan retry. Si `false`, solo las listadas en `retryable-exceptions`. |
| `spring.cloud.stream.bindings.<name>.consumer.retryable-exceptions` | `Map<Class,Boolean>` | `{}` | Override por tipo de excepción: `true` = retryable, `false` = no retryable (va directo a DLQ). |
| `spring.cloud.stream.kafka.bindings.<name>.consumer.enable-dlq` | `boolean` | `false` | Habilita DLQ en Kafka. Sin esto, los mensajes que agotan reintentos son descartados. |
| `spring.cloud.stream.kafka.bindings.<name>.consumer.dlq-name` | `String` | `<topic>.dlq` | Nombre explícito del topic DLQ en Kafka. |
| `spring.cloud.stream.rabbit.bindings.<name>.consumer.auto-bind-dlq` | `boolean` | `false` | Habilita DLQ en RabbitMQ. Crea automáticamente la DLQ y la configura en el exchange. |
| `spring.cloud.stream.rabbit.bindings.<name>.consumer.requeue-rejected` | `boolean` | `true` | En RabbitMQ: si `false`, los mensajes rechazados van a DLQ en lugar de reencolarerse. |

## Buenas y malas prácticas

**Hacer:**

- Configurar siempre `enable-dlq: true` (Kafka) o `auto-bind-dlq: true` (RabbitMQ) en todos los consumers de producción. Sin DLQ, los mensajes que fallan después de `maxAttempts` son descartados silenciosamente. Los datos perdidos en mensajería son frecuentemente irrecuperables.
- Usar `retryable-exceptions` para distinguir errores transitorios (timeout de red, BD no disponible) de errores permanentes (payload inválido, violación de restricción). Los errores permanentes no deben reintentarse; enviarlos directamente a la DLQ ahorra tiempo y recursos.
- Implementar un proceso de reprocessing de DLQ. Una DLQ que solo acumula mensajes sin ser monitoreada es un punto ciego en producción. Configurar alertas cuando la DLQ supera un umbral de mensajes.

**Evitar:**

- Configurar `back-off-max-interval` muy alto (>60s) en consumers de Kafka. El binder Kafka bloquea el thread de polling durante el backoff, lo que en un consumer con muchas particiones puede causar un rebalanceo del consumer group (Kafka interpreta la inactividad como caída del consumer).
- Ignorar el error channel. Por defecto, si no hay un `@ServiceActivator` registrado en el canal de errores, los mensajes que agotan los reintentos sin DLQ habilitada se descartan en silencio. Este es el escenario de pérdida de datos más común en Spring Cloud Stream.

## Comparación: DLQ en Kafka vs RabbitMQ

El mecanismo de DLQ tiene diferencias importantes según el binder utilizado. Un profesional senior debe conocer estas diferencias para operar ambos sistemas.

| Aspecto | Kafka (`enable-dlq`) | RabbitMQ (`auto-bind-dlq`) |
|---------|---------------------|---------------------------|
| Infraestructura creada | Topic `<topic>.dlq` (o `dlq-name`) | Exchange DLX + Queue `<queue>.dlq` |
| Retención de mensajes | Según política de retención del topic DLQ | Hasta ACK o TTL (`dlq-ttl`) |
| Headers del mensaje en DLQ | `x-exception-message`, `x-exception-stacktrace`, `x-original-topic` | `x-death` (headers AMQP estándar) |
| Reprocessing | Resetear offset del topic DLQ desde el inicio | NACK + requeue o mover manualmente |
| Configuración complementaria | `dlq-partitions`, `dlq-name` | `dlq-ttl`, `dlq-dead-letter-exchange` |

> [ADVERTENCIA] En Kafka, el consumer del topic DLQ debe configurarse con `enable-dlq: false` para evitar el bucle DLQ → DLQ → DLQ. Si el consumer de la DLQ también falla, sin esta protección se crea una cadena infinita de DLQs.

---

← [6.7 Particionado de mensajes](sc-stream-particionado.md) | [Índice (README.md)](README.md) | [6.9 Polling y batch consumers](sc-stream-polling-batch.md) →
