# 6.5 Binder RabbitMQ — configuración y propiedades

← [6.4 Kafka Streams binder](sc-stream-binder-kafka-streams.md) | [Índice (README.md)](README.md) | [6.6 Serialización y deserialización (SerDes)](sc-stream-serdes.md) →

## Introducción

El binder de RabbitMQ traduce el modelo de bindings de Spring Cloud Stream a la API AMQP de RabbitMQ. El problema que resuelve respecto al binder Kafka es diferente: mientras Kafka es adecuado para streams de eventos de alto volumen con retención a largo plazo, RabbitMQ es la opción natural para comunicación orientada a mensajes con enrutamiento flexible por routing keys, priorización de mensajes y casos donde el acknowledgement explícito es necesario para garantías transaccionales.

Spring Cloud Stream crea automáticamente la infraestructura AMQP al arrancar: un exchange de tipo `topic` con el nombre del `destination`, y una queue con el nombre `<destination>.<group>` enlazada al exchange mediante el routing key configurado. El desarrollador no necesita crear manualmente la infraestructura en RabbitMQ.

> [CONCEPTO] Spring Cloud Stream crea automáticamente el exchange (`<destination>`) de tipo topic y la queue (`<destination>.<group>`) al arrancar. La gestión de vhosts, usuarios y políticas de HA se realiza desde la consola RabbitMQ o su API de gestión, fuera del alcance de este módulo. Ver https://www.rabbitmq.com/documentation.html

> [PREREQUISITO] El modelo de bindings (6.2) y conceptos básicos de RabbitMQ: exchange, queue, routing key, acknowledgement.

## Representación visual

El binder de RabbitMQ mapea los conceptos del binding a la infraestructura AMQP de forma automática. El siguiente diagrama muestra la infraestructura que crea Spring Cloud Stream al arrancar.

```
┌─────────────────────────────────────────────────────────────┐
│  RabbitMQ Broker                                            │
│                                                             │
│  Exchange: orders (tipo: topic)                             │
│     │                                                       │
│     ├─ routing key: "#"  ──► Queue: orders.order-processors │
│     │                         (durable, consumer group)     │
│     │                                                       │
│     └─ routing key: "high.#" ► Queue: orders.priority-group │
│                               (si bindingRoutingKey=high.#) │
└──────────────────────────────────────────────────────────────┘

Spring Cloud Stream Application:
  processOrder-in-0  ──►  subscribe a Queue: orders.order-processors
  processOrder-out-0 ──►  publish a Exchange: orders-out (routingKeyExpression)
```

El routing key por defecto es `#` (acepta todos los mensajes del exchange). Configurar `bindingRoutingKey` permite filtrar mensajes en el broker sin procesarlos en la aplicación, reduciendo el consumo de recursos.

## Ejemplo central

El siguiente ejemplo configura un servicio que consume pedidos prioritarios de RabbitMQ con acknowledgement manual, y produce notificaciones con routing key basada en el tipo de pedido.

**pom.xml**
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-stream-binder-rabbit</artifactId>
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
            max-attempts: 3
            back-off-initial-interval: 1000
        processOrder-out-0:
          destination: notifications
      rabbit:
        bindings:
          processOrder-in-0:
            consumer:
              acknowledge-mode: AUTO          # AUTO, MANUAL o NONE
              binding-routing-key: "orders.#" # filtro en el exchange
              prefetch: 10                    # QoS: máximo de mensajes sin ACK en vuelo
              requeue-rejected: false          # no reencolar mensajes rechazados (van a DLQ)
              auto-bind-dlq: true             # crear automáticamente la DLQ en RabbitMQ
              dlq-ttl: 60000                  # TTL de mensajes en DLQ (ms)
          processOrder-out-0:
            producer:
              routing-key-expression: "headers['order-type']"  # routing key dinámica por cabecera
              delayed-exchange: false
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest
    virtual-host: /
```

**OrderEvent.java**
```java
package com.example.rabbit;

public record OrderEvent(String orderId, String type, double amount) {}
```

**Notification.java**
```java
package com.example.rabbit;

public record Notification(String orderId, String message, String level) {}
```

**OrderRabbitConfig.java**
```java
package com.example.rabbit;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.Message;
import org.springframework.messaging.support.MessageBuilder;

import java.util.function.Function;

@Configuration
public class OrderRabbitConfig {

    /**
     * processOrder: consume de exchange 'orders' con routing key 'orders.#'
     * Produce al exchange 'notifications' con routing key de la cabecera 'order-type'
     */
    @Bean
    public Function<Message<OrderEvent>, Message<Notification>> processOrder() {
        return message -> {
            OrderEvent order = message.getPayload();

            // Con acknowledge-mode: AUTO, el ACK se envía automáticamente al retornar.
            // Con MANUAL, habría que acceder al Channel desde las cabeceras y hacer ack() explícito.
            String level = order.amount() > 1000 ? "HIGH" : "NORMAL";
            String notificationMessage = String.format(
                "Order %s of type %s processed: %.2f", order.orderId(), order.type(), order.amount());

            Notification notification = new Notification(order.orderId(), notificationMessage, level);

            return MessageBuilder
                .withPayload(notification)
                // Esta cabecera se usa como routing key en el exchange de notificaciones
                .setHeader("order-type", order.type().toLowerCase())
                .build();
        };
    }
}
```

**RabbitApplication.java**
```java
package com.example.rabbit;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class RabbitApplication {
    public static void main(String[] args) {
        SpringApplication.run(RabbitApplication.class, args);
    }
}
```

## Tabla de elementos clave

Las propiedades del binder RabbitMQ se configuran bajo `spring.cloud.stream.rabbit.bindings.<name>.*`. Las propiedades de conexión al broker van bajo `spring.rabbitmq.*`.

| Parámetro | Tipo | Default | Descripción |
|-----------|------|---------|-------------|
| `spring.cloud.stream.rabbit.bindings.<name>.consumer.acknowledge-mode` | `String` | `AUTO` | Modo de acknowledgement: `AUTO` (tras procesamiento), `MANUAL` (ACK explícito en código), `NONE` (sin ACK, máximo throughput). |
| `spring.cloud.stream.rabbit.bindings.<name>.consumer.binding-routing-key` | `String` | `#` | Routing key para enlazar la queue al exchange. `#` acepta todo. `orders.#` acepta cualquier clave que empiece por `orders.`. |
| `spring.cloud.stream.rabbit.bindings.<name>.consumer.prefetch` | `int` | `1` | Número máximo de mensajes sin ACK en vuelo por consumidor (QoS). Aumentar mejora el throughput pero reduce el fairness entre consumidores. |
| `spring.cloud.stream.rabbit.bindings.<name>.consumer.requeue-rejected` | `boolean` | `true` | Si `false`, los mensajes rechazados van a la DLQ en lugar de reencolarese. Con `true`, un mensaje que falla se re-entrega indefinidamente si no hay DLQ. |
| `spring.cloud.stream.rabbit.bindings.<name>.consumer.auto-bind-dlq` | `boolean` | `false` | Crea automáticamente la Dead Letter Queue (`<destination>.<group>.dlq`) y la configura en el exchange. |
| `spring.cloud.stream.rabbit.bindings.<name>.consumer.dlq-ttl` | `int` (ms) | (ninguno) | TTL de los mensajes en la DLQ. Después de este tiempo, los mensajes se descartan o se mueven según la política de la queue. |
| `spring.cloud.stream.rabbit.bindings.<name>.producer.routing-key-expression` | `String` | (ninguno) | SpEL que evalúa el routing key de cada mensaje. Ejemplo: `headers['type']` usa la cabecera `type` como routing key. |
| `spring.rabbitmq.host` | `String` | `localhost` | Host del broker RabbitMQ. |
| `spring.rabbitmq.port` | `int` | `5672` | Puerto AMQP. |
| `spring.rabbitmq.virtual-host` | `String` | `/` | Virtual host de RabbitMQ. Permite aislar entornos en el mismo broker. |

## Buenas y malas prácticas

**Hacer:**

- Configurar `requeue-rejected: false` y `auto-bind-dlq: true` en todos los consumers de producción. Con `requeue-rejected: true` (el default), un mensaje que genera una excepción de procesamiento se reencola de inmediato y el consumer lo procesa de nuevo en bucle infinito, saturando el CPU del servicio y el broker.
- Ajustar `prefetch` según el tiempo de procesamiento del mensaje. Si el procesamiento tarda 500ms y hay 10 instancias del consumidor, un prefetch de 10 aprovecha mejor el throughput. Si el procesamiento tarda 5ms, un prefetch de 50 o 100 es más eficiente.
- Usar `acknowledge-mode: AUTO` como punto de partida. El modo MANUAL es necesario solo cuando el código necesita decidir condicionalmente si hacer ACK o NACK después de procesar parcialmente el mensaje (por ejemplo, en sagas de larga duración).

**Evitar:**

- Usar `acknowledge-mode: NONE` en consumidores que persisten datos. Este modo maximiza el throughput pero no envía confirmaciones al broker: si el servicio cae después de procesar un lote de mensajes pero antes de persistirlos, los mensajes se pierden permanentemente.
- Configurar `prefetch: 1` con muchas instancias del consumidor. Con prefetch 1, cada instancia hace round-trip al broker por cada mensaje, lo que puede generar una tasa de mensajes por segundo muy baja cuando el broker está en red con alta latencia (>1ms).

## Comparación: RabbitMQ vs Kafka en Spring Cloud Stream

La elección del binder determina las garantías de entrega, la retención de mensajes y la estrategia de enrutamiento disponible.

| Característica | Binder Kafka | Binder RabbitMQ |
|---------------|-------------|-----------------|
| Retención de mensajes | Configurable (días, semanas, indefinida) | Hasta que el consumidor hace ACK |
| Enrutamiento | Por partición (clave de particionado) | Por routing key y tipo de exchange |
| Reprocessing | Resetear offset a cualquier punto | Requiere DLQ + requeue manual |
| Ordering guarantees | Por partición | Por queue (sin prioridad) |
| Throughput típico | Muy alto (millones/seg) | Alto (cientos de miles/seg) |
| Caso de uso ideal | Event sourcing, auditoría, ETL | Workflow, tareas, RPC asíncrono |

---

← [6.4 Kafka Streams binder](sc-stream-binder-kafka-streams.md) | [Índice (README.md)](README.md) | [6.6 Serialización y deserialización (SerDes)](sc-stream-serdes.md) →
