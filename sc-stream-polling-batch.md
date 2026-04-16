# 6.9 Polling y batch consumers

← [6.8 Gestión de errores y DLQ](sc-stream-error-handling.md) | [Índice (README.md)](README.md) | [6.10 StreamBridge — producción de mensajes programática](sc-stream-streambridge.md) →

## Introducción

El modelo funcional de Spring Cloud Stream (6.1) es reactivo por naturaleza: el framework entrega mensajes al handler tan pronto como llegan al broker. Sin embargo, existen escenarios donde este modelo no es adecuado. El polling consumer resuelve el problema del consumo bajo demanda: en lugar de que el framework entregue mensajes continuamente, la aplicación decide cuándo consumir, a qué ritmo y cuántos mensajes a la vez. Esto es necesario en integraciones con sistemas externos lentos, en pipelines con control de backpressure explícito o cuando el procesamiento depende de recursos externos que no se quieren saturar.

El batch consumer resuelve un problema diferente: la ineficiencia de procesar un mensaje a la vez cuando el sistema puede aprovechar el procesamiento en lote. Con batch mode habilitado, el handler recibe una lista de mensajes en lugar de uno solo, permitiendo inserciones en base de datos en bulk, llamadas a APIs externas en batch y otras optimizaciones de throughput.

> [CONCEPTO] Estos son modos de consumo alternativos al modelo funcional reactivo estándar. No son el modo de consumo primario recomendado. Para casos de uso event-driven estándar, el modelo funcional (6.1) es siempre preferible.

## Representación visual

La diferencia entre el modelo reactivo, el polling y el batch afecta la estrategia de entrega de mensajes al handler. La siguiente tabla compara los tres modos.

```
MODELO REACTIVO (por defecto):
────────────────────────────────────────────────────────
Broker ──► [msg1] ──► handler(msg1)
       ──► [msg2] ──► handler(msg2)   (entrega continua por el framework)
       ──► [msg3] ──► handler(msg3)

POLLING CONSUMER (PollableMessageSource):
────────────────────────────────────────────────────────
Broker                              Aplicación
  [msg1]                 ◄── poll() (la app decide cuándo consumir)
  [msg2]  ─ reservados ─
  [msg3]                 ◄── poll() (tras N segundos configurados)

BATCH CONSUMER (batch-mode: true):
────────────────────────────────────────────────────────
Broker ──► [msg1, msg2, msg3, ...] ──► handler(List<Message<T>>)
                                       (recibe una lista entera)
```

## Ejemplo central

El siguiente ejemplo implementa ambos patrones en un mismo servicio: un polling consumer que consulta un topic cada 5 segundos y lo procesa con `@Scheduled`, y un batch consumer que recibe listas de pedidos para inserciones en bulk.

**pom.xml**
```xml
<dependencies>
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
      definition: batchProcessOrders
    stream:
      bindings:
        # Binding para el polling consumer (no hay un bean funcional estándar para él)
        # El PollableMessageSource se configura manualmente en código
        manual-poll-in-0:
          destination: orders-polling
          group: polling-consumers
        # Binding para el batch consumer
        batchProcessOrders-in-0:
          destination: orders-batch
          group: batch-processors
          consumer:
            batch-mode: true              # habilitar modo batch
            max-attempts: 1              # en batch mode, los reintentos son por lote
        batchProcessOrders-out-0:
          destination: orders-processed-batch
      kafka:
        binder:
          brokers: localhost:9092
        bindings:
          orders-batch:
            consumer:
              max-poll-records: 50       # máximo de mensajes por poll en batch mode
```

**OrderEvent.java**
```java
package com.example.polling;

public record OrderEvent(String orderId, String customerId, double amount) {}
```

**PollingAndBatchConfig.java**
```java
package com.example.polling;

import org.springframework.cloud.stream.binding.BinderAwareChannelResolver;
import org.springframework.cloud.stream.messaging.PollableMessageSource;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.Message;
import org.springframework.messaging.support.MessageBuilder;
import org.springframework.scheduling.annotation.EnableScheduling;
import org.springframework.scheduling.annotation.Scheduled;

import java.util.ArrayList;
import java.util.List;
import java.util.function.Consumer;

@Configuration
@EnableScheduling
public class PollingAndBatchConfig {

    private final PollableMessageSource pollableSource;

    /**
     * El PollableMessageSource se inyecta por nombre de binding.
     * Spring Cloud Stream crea automáticamente un bean de tipo PollableMessageSource
     * para cada binding configurado en application.yml.
     */
    public PollingAndBatchConfig(PollableMessageSource pollableSource) {
        this.pollableSource = pollableSource;
    }

    /**
     * Polling consumer: en lugar de que el framework entregue mensajes,
     * esta tarea @Scheduled invoca poll() cada 5 segundos.
     * Procesa hasta 10 mensajes por ciclo de polling.
     */
    @Scheduled(fixedDelay = 5000)
    public void pollMessages() {
        int count = 0;
        boolean hadMessages = true;

        while (hadMessages && count < 10) {
            hadMessages = pollableSource.poll(message -> {
                OrderEvent order = (OrderEvent) message.getPayload();
                System.out.printf("[POLL] Processing order: %s%n", order.orderId());
                // Aquí va la lógica de procesamiento
            }, OrderEvent.class);

            if (hadMessages) {
                count++;
            }
        }

        System.out.printf("[POLL] Processed %d messages in this cycle%n", count);
    }

    /**
     * Batch consumer: recibe una lista de mensajes en lugar de uno a uno.
     * El tipo de parámetro es List<Message<OrderEvent>> cuando batch-mode=true.
     * Ideal para inserciones bulk en base de datos.
     */
    @Bean
    public Consumer<List<Message<OrderEvent>>> batchProcessOrders() {
        return messages -> {
            System.out.printf("[BATCH] Processing batch of %d orders%n", messages.size());

            List<OrderEvent> orders = new ArrayList<>();
            for (Message<OrderEvent> message : messages) {
                orders.add(message.getPayload());
            }

            // Simulación de inserción bulk (en producción: jdbcTemplate.batchUpdate(...))
            for (OrderEvent order : orders) {
                System.out.printf("[BATCH]   Order: %s customer: %s amount: %.2f%n",
                    order.orderId(), order.customerId(), order.amount());
            }

            System.out.printf("[BATCH] Batch completed. %d orders processed.%n", orders.size());
        };
    }
}
```

**PollingApplication.java**
```java
package com.example.polling;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class PollingApplication {
    public static void main(String[] args) {
        SpringApplication.run(PollingApplication.class, args);
    }
}
```

## Tabla de elementos clave

Los parámetros de polling y batch afectan directamente el throughput y la latencia del sistema. La siguiente tabla cubre los más relevantes.

| Parámetro | Tipo | Default | Descripción |
|-----------|------|---------|-------------|
| `spring.cloud.stream.bindings.<name>.consumer.batch-mode` | `boolean` | `false` | Habilita el modo batch. El handler recibe `List<Message<T>>` en lugar de `Message<T>`. |
| `spring.cloud.stream.kafka.bindings.<name>.consumer.max-poll-records` | `int` | `500` | Máximo de registros que el cliente Kafka lee en un poll. En batch-mode, determina el tamaño máximo del batch entregado al handler. |
| `spring.cloud.stream.bindings.<name>.consumer.max-attempts` | `int` | `3` | En batch-mode, los reintentos son para el lote completo. Si falla un mensaje del lote, todo el lote se reintenta. Configurar `1` para manejar errores en el handler con lógica propia. |
| `PollableMessageSource.poll(handler, type)` | Método | — | Consume un mensaje del binding bajo demanda. Retorna `true` si había un mensaje, `false` si la queue estaba vacía. |
| `spring.cloud.stream.kafka.bindings.<name>.consumer.idle-event-interval` | `long` (ms) | `30000` | Tiempo de inactividad tras el cual se dispara un evento de idle. Útil para detectar topics vacíos en el polling consumer. |

## Buenas y malas prácticas

**Hacer:**

- Usar batch mode cuando el handler escribe en una base de datos. Una inserción en bulk de 50 registros es 10-50 veces más rápida que 50 inserciones individuales. La diferencia de latencia se nota especialmente con bases de datos relacionales donde cada INSERT tiene overhead de transacción.
- Configurar `max-attempts: 1` en batch mode y gestionar los errores dentro del handler con lógica propia. Con `max-attempts > 1`, si un mensaje del lote falla, todo el lote se reintenta, lo que puede duplicar el procesamiento de los mensajes correctos del mismo lote.
- Usar `PollableMessageSource.poll()` cuando la velocidad de consumo debe acoplarse a la velocidad de un recurso externo limitado. El polling explícito es el mecanismo de backpressure más directo disponible en Spring Cloud Stream fuera del modelo reactivo con Flux.

**Evitar:**

- Combinar batch mode con `enable-dlq: true` sin entender la semántica del batch en DLQ. Con batch mode, si el lote entero falla después de los reintentos, todos los mensajes del lote van a la DLQ, incluyendo los mensajes que eran válidos pero que fueron procesados en el mismo lote que el inválido.
- Usar `PollableMessageSource.poll()` en un bucle sin delay como sustituto del modelo reactivo. Sin delay entre polls, el servicio hace polling continuo al broker, consumiendo CPU y generando tráfico de red innecesario en topics con bajo volumen de mensajes.

---

← [6.8 Gestión de errores y DLQ](sc-stream-error-handling.md) | [Índice (README.md)](README.md) | [6.10 StreamBridge — producción de mensajes programática](sc-stream-streambridge.md) →
