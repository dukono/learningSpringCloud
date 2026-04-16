# 6.3 Binder Kafka — configuración y propiedades

← [6.2 Bindings y binders](sc-stream-bindings.md) | [Índice (README.md)](README.md) | [6.4 Kafka Streams binder](sc-stream-binder-kafka-streams.md) →

## Introducción

El binder de Kafka es la implementación que traduce el modelo de bindings abstracto de Spring Cloud Stream a la API nativa del cliente Kafka. Resuelve el problema de integrar microservicios con Kafka sin escribir código de configuración de bajo nivel: sin este binder, un desarrollador tendría que configurar manualmente `KafkaConsumer`, `KafkaProducer`, gestionar offsets, particiones y grupos de consumidores. Con el binder, toda esta infraestructura se configura mediante propiedades en `application.yml`.

El binder expone dos niveles de configuración: propiedades a nivel de **binder** (afectan a la conexión con el clúster Kafka, aplican a todos los bindings) y propiedades a nivel de **binding** (afectan al comportamiento de un topic o grupo de consumidores concreto). Conocer esta distinción evita configurar en el nivel equivocado y pasar horas depurando comportamientos inesperados en producción.

> [PREREQUISITO] Kafka a nivel de usuario: saber publicar y consumir mensajes, entender el concepto de topic, partición, offset y consumer group. Spring Cloud Stream no enseña los internals de Kafka; solo gestiona la integración desde la capa de la aplicación.

## Representación visual

La siguiente tabla muestra cómo los conceptos de Kafka se mapean a los conceptos de Spring Cloud Stream a través del binder.

```
Spring Cloud Stream          Binder Kafka             Apache Kafka
─────────────────────────────────────────────────────────────────
Binding (canal lógico)  ──►  KafkaConsumer/Producer  ──►  Topic + Partitions
consumer group          ──►  group.id                ──►  Consumer Group
destination             ──►  topic name              ──►  Topic
contentType             ──►  value.serializer         ──►  Serialization format
max-attempts + backoff  ──►  RetryTemplate           ──►  (retry en app, no en broker)
enableDlq: true         ──►  DLQ topic creation      ──►  <topic>.dlq
startOffset: earliest   ──►  auto.offset.reset       ──►  earliest
partitionCount          ──►  partitions en topic     ──►  Partition count
```

> [CONCEPTO] La gestión interna de particiones, offsets y coordinación de grupos la realiza el cliente Kafka subyacente de forma transparente al desarrollador. Spring Cloud Stream solo expone la interfaz de configuración. Para detalles del protocolo Kafka ver https://kafka.apache.org/documentation/

## Ejemplo central

El siguiente ejemplo configura un servicio que consume eventos de pedidos desde un topic Kafka con control de offsets, DLQ habilitada, producción síncrona y compresión.

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
      definition: processOrder
    stream:
      bindings:
        processOrder-in-0:
          destination: orders-raw
          group: order-processors
          consumer:
            max-attempts: 3
            back-off-initial-interval: 500
            back-off-multiplier: 2.0
            back-off-max-interval: 5000
        processOrder-out-0:
          destination: orders-processed
      kafka:
        binder:
          brokers: kafka-host:9092         # lista separada por comas para múltiples brokers
          auto-create-topics: true          # crear topics automáticamente si no existen
          auto-add-partitions: false        # no añadir particiones a topics existentes
          min-partition-count: 1
          replication-factor: 3            # factor de replicación para topics creados automáticamente
          configuration:                   # propiedades nativas del producer/consumer Kafka
            security.protocol: PLAINTEXT
        bindings:
          processOrder-in-0:
            consumer:
              start-offset: earliest       # consumir desde el principio del topic al crear el grupo
              reset-offsets: false          # no resetear offsets en cada arranque
              enable-dlq: true             # habilitar Dead Letter Queue
              dlq-name: orders-raw.dlq     # nombre explícito de la DLQ (default: <topic>.dlq)
              dlq-partitions: 1
              auto-commit-offset: true     # commit automático tras procesamiento exitoso
          processOrder-out-0:
            producer:
              sync: true                   # producción síncrona: espera ACK del broker
              compression-type: snappy     # compresión del payload (none, gzip, snappy, lz4, zstd)
              batch-size: 16384            # tamaño de batch del producer en bytes
```

**OrderEvent.java**
```java
package com.example.kafka;

public record OrderEvent(String orderId, String customerId, double amount) {}
```

**OrderResult.java**
```java
package com.example.kafka;

public record OrderResult(String orderId, String status, String reason) {}
```

**OrderProcessor.java**
```java
package com.example.kafka;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.Message;
import org.springframework.messaging.support.MessageBuilder;

import java.util.function.Function;

@Configuration
public class OrderProcessor {

    @Bean
    public Function<Message<OrderEvent>, Message<OrderResult>> processOrder() {
        return message -> {
            OrderEvent event = message.getPayload();

            // Acceso a la partición y offset de Kafka a través de las cabeceras del mensaje
            Integer partition = (Integer) message.getHeaders()
                .getOrDefault("kafka_receivedPartitionId", -1);
            Long offset = (Long) message.getHeaders()
                .getOrDefault("kafka_offset", -1L);

            System.out.printf("[processOrder] Partition=%d Offset=%d OrderId=%s%n",
                partition, offset, event.orderId());

            if (event.amount() <= 0) {
                throw new IllegalArgumentException(
                    "Amount must be positive for orderId: " + event.orderId());
            }

            OrderResult result = new OrderResult(event.orderId(), "PROCESSED", "OK");

            return MessageBuilder
                .withPayload(result)
                .setHeader("X-Source-Partition", partition)
                .build();
        };
    }
}
```

**KafkaApplication.java**
```java
package com.example.kafka;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class KafkaApplication {
    public static void main(String[] args) {
        SpringApplication.run(KafkaApplication.class, args);
    }
}
```

## Tabla de elementos clave

Las propiedades del binder Kafka se organizan en dos grupos: propiedades del binder (conexión al clúster) y propiedades de binding (comportamiento por topic/consumer). La siguiente tabla cubre las más relevantes.

| Parámetro | Tipo | Default | Descripción |
|-----------|------|---------|-------------|
| `spring.cloud.stream.kafka.binder.brokers` | `String` | `localhost` | Lista de brokers Kafka separada por comas. Formato: `host1:9092,host2:9092`. |
| `spring.cloud.stream.kafka.binder.auto-create-topics` | `boolean` | `true` | Crea automáticamente topics si no existen. Desactivar en producción cuando los topics son gestionados por Terraform/Ansible. |
| `spring.cloud.stream.kafka.binder.auto-add-partitions` | `boolean` | `false` | Añade particiones a topics existentes si `partitionCount` es mayor al actual. |
| `spring.cloud.stream.kafka.binder.replication-factor` | `short` | `1` | Factor de replicación para topics creados automáticamente. En producción debe ser >= 3. |
| `spring.cloud.stream.kafka.bindings.<name>.consumer.start-offset` | `String` | `null` | Offset inicial: `earliest` (desde el inicio) o `latest` (solo mensajes nuevos). Solo aplica al crear el grupo por primera vez. |
| `spring.cloud.stream.kafka.bindings.<name>.consumer.reset-offsets` | `boolean` | `false` | Si `true`, resetea los offsets al `startOffset` en cada arranque. Útil para reprocessing; peligroso en producción. |
| `spring.cloud.stream.kafka.bindings.<name>.consumer.enable-dlq` | `boolean` | `false` | Habilita la Dead Letter Queue. Los mensajes que superan `max-attempts` se envían al topic `<destination>.dlq`. |
| `spring.cloud.stream.kafka.bindings.<name>.consumer.dlq-name` | `String` | `<topic>.dlq` | Nombre explícito del topic DLQ. Permite compartir una única DLQ entre múltiples topics. |
| `spring.cloud.stream.kafka.bindings.<name>.producer.sync` | `boolean` | `false` | Producción síncrona: el hilo del producer espera el ACK del broker antes de continuar. Reduce throughput pero garantiza entrega. |
| `spring.cloud.stream.kafka.bindings.<name>.producer.compression-type` | `String` | `none` | Tipo de compresión: `none`, `gzip`, `snappy`, `lz4`, `zstd`. |

## Buenas y malas prácticas

**Hacer:**

- Desactivar `auto-create-topics: false` en producción y gestionar los topics con herramientas de infraestructura como código (Terraform, Kafka CLI). La creación automática de topics puede generar topics con configuraciones incorrectas (factor de replicación 1, una sola partición) que son difíciles de corregir sin interrupción de servicio.
- Configurar `enable-dlq: true` en todos los consumers de producción. Sin DLQ, los mensajes que fallan sistemáticamente bloquean la partición entera: el consumer no avanza el offset y el lag crece indefinidamente hasta agotar el espacio de retención del topic.
- Usar `sync: true` solo cuando la garantía de entrega es más importante que el throughput. En producers de alta frecuencia (>10K msg/s), la producción síncrona puede introducir una latencia de 2-10ms por mensaje que se acumula en el pipeline.
- Establecer `replication-factor: 3` en producción. Con factor 1, la pérdida de un broker implica pérdida de datos. Con factor 3, se toleran 2 fallos simultáneos de broker.

**Evitar:**

- Configurar `reset-offsets: true` en un entorno de producción con datos reales. Esta propiedad hace que el servicio reprocese todos los mensajes desde el `startOffset` en cada arranque, duplicando el procesamiento y potencialmente corrompiendo el estado del sistema.
- Ignorar la cabecera `kafka_receivedPartitionId` en el handler. Esta cabecera es la única forma de saber en qué partición se recibió el mensaje, dato esencial para diagnosticar problemas de distribución de carga y procesamiento no ordenado.
- Mezclar binder-level `configuration` con binding-level properties sin entender la precedencia. Las propiedades nativas del cliente Kafka en `binder.configuration` aplican a todos los consumers y producers; una propiedad mal puesta ahí puede afectar comportamientos que el desarrollador creía aislar en un binding específico.

## Comparación: producción asíncrona vs síncrona

La elección entre producción asíncrona (por defecto) y síncrona (`sync: true`) tiene implicaciones directas en el throughput y las garantías de entrega.

| Aspecto | Asíncrona (`sync: false`) | Síncrona (`sync: true`) |
|---------|--------------------------|------------------------|
| Comportamiento | El producer no espera ACK del broker | El hilo espera confirmación del broker |
| Throughput | Alto (batching automático) | Bajo (una petición a la vez por hilo) |
| Latencia | Baja (fuego y olvida) | Alta (round-trip al broker) |
| Pérdida de mensajes ante caída del broker | Posible (mensajes en buffer no enviados) | Imposible (excepción si el broker no responde) |
| Caso de uso | Telemetría, logs de auditoría | Pedidos, pagos, operaciones con efecto financiero |

> [ADVERTENCIA] Si `sync: true` y el broker Kafka está caído, el método del producer lanzará una excepción. El servicio debe tener un mecanismo de circuit breaker (6.3 + Resilience4j, 5.x) para evitar que los hilos se queden bloqueados esperando al broker indefinidamente.

---

← [6.2 Bindings y binders](sc-stream-bindings.md) | [Índice (README.md)](README.md) | [6.4 Kafka Streams binder](sc-stream-binder-kafka-streams.md) →
