# 6.7 Particionado de mensajes

← [6.6 Serialización y deserialización (SerDes)](sc-stream-serdes.md) | [Índice (README.md)](README.md) | [6.8 Gestión de errores y DLQ](sc-stream-error-handling.md) →

## Introducción

El particionado de mensajes en Spring Cloud Stream resuelve el problema del procesamiento ordenado y con estado en un sistema distribuido con múltiples instancias. Sin particionado, cuando un servicio escala a varias instancias, los mensajes del mismo cliente o pedido pueden ser procesados por instancias diferentes en orden arbitrario, corrompiendo el estado de cualquier agregación. Con particionado, todos los mensajes de un mismo valor de clave (por ejemplo, el mismo `customerId`) son siempre enrutados a la misma instancia del consumidor, garantizando el orden y la consistencia del estado local.

Esta garantía es especialmente importante en arquitecturas con state stores locales (Kafka Streams), en sagas de coreografía donde el orden de los eventos importa, y en consumers que mantienen caché local por cliente. El particionado opera en la capa del producer: es el producer quien decide en qué partición deposita el mensaje según una expresión SpEL evaluada sobre el payload o las cabeceras.

> [PREREQUISITO] Serialización (6.6). La `partitionKeyExpression` se evalúa sobre el mensaje ya serializado (el payload es el objeto Java en el handler, no los bytes). El orden de operaciones es: serialización → evaluación de partitionKeyExpression → envío al broker.

## Representación visual

El diagrama siguiente muestra el flujo de particionado desde el producer hasta los consumers, con las propiedades que lo controlan en cada paso.

```
Producer (1 instancia)
    │
    │  partitionKeyExpression: "payload.customerId"
    │  partitionCount: 3
    ▼
┌──────────────────────────────────────┐
│  Spring Cloud Stream Partitioner     │
│                                      │
│  key = hash(customerId) % 3          │
│                                      │
│  customerId=A  → partition 0          │
│  customerId=B  → partition 1          │
│  customerId=C  → partition 2          │
└──────────┬───────────────────────────┘
           │
           ▼
    Topic: orders (3 partitions)
    ┌──────────────────────────────┐
    │  Partition 0: [A, A, A, ...]  │  ──► Consumer instancia 0 (instanceIndex=0)
    │  Partition 1: [B, B, B, ...]  │  ──► Consumer instancia 1 (instanceIndex=1)
    │  Partition 2: [C, C, C, ...]  │  ──► Consumer instancia 2 (instanceIndex=2)
    └──────────────────────────────┘

Configuración en el consumer:
    partitioned: true
    instanceCount: 3
    instanceIndex: 0/1/2  (configurable por instancia)
```

La propiedad `instanceIndex` debe configurarse de forma diferente en cada instancia del servicio. En Kubernetes, esto se hace mediante variables de entorno inyectadas desde el nombre del pod.

## Ejemplo central

El siguiente ejemplo configura un servicio con particionado por `customerId`: el producer garantiza que todos los mensajes del mismo cliente van a la misma partición, y los consumers se configuran para recibir solo los mensajes de su partición asignada.

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

**application.yml (servicio producer)**
```yaml
spring:
  cloud:
    function:
      definition: routeOrder
    stream:
      bindings:
        routeOrder-in-0:
          destination: orders-input
          group: router-group
        routeOrder-out-0:
          destination: orders-partitioned
          producer:
            partition-count: 3                              # número de particiones del topic destino
            partition-key-expression: "payload.customerId"  # expresión SpEL sobre el payload Java
            # Alternativa con extractor por nombre de bean:
            # partition-key-extractor-name: customerIdExtractor
      kafka:
        binder:
          brokers: localhost:9092
          auto-create-topics: true
```

**application.yml (servicio consumer — instancia 0 de 3)**
```yaml
spring:
  cloud:
    function:
      definition: processOrder
    stream:
      instance-count: 3       # total de instancias del consumer
      instance-index: 0       # índice de ESTA instancia (0-based)
      bindings:
        processOrder-in-0:
          destination: orders-partitioned
          group: order-processors
          consumer:
            partitioned: true  # habilitar consumer particionado
      kafka:
        binder:
          brokers: localhost:9092
```

**OrderEvent.java**
```java
package com.example.partitioning;

public record OrderEvent(String orderId, String customerId, double amount, String type) {}
```

**PartitioningConfig.java**
```java
package com.example.partitioning;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.cloud.stream.binder.PartitionKeyExtractorStrategy;
import org.springframework.messaging.Message;

import java.util.function.Function;

@Configuration
public class PartitioningConfig {

    /**
     * Producer: recibe pedidos sin particionar y los reenvía con particionado por customerId.
     * La partitionKeyExpression "payload.customerId" extrae el customerId del POJO Java.
     */
    @Bean
    public Function<Message<OrderEvent>, Message<OrderEvent>> routeOrder() {
        return message -> {
            // No es necesario hacer nada especial — el particionado lo gestiona el framework
            // según partitionKeyExpression y partitionCount del binding de salida
            return message;
        };
    }

    /**
     * Alternativa al partitionKeyExpression: un bean que implementa PartitionKeyExtractorStrategy.
     * Se referencia en la configuración como partition-key-extractor-name: customerIdExtractor.
     * Útil cuando la lógica de extracción de la clave es compleja o requiere servicios inyectados.
     */
    @Bean
    public PartitionKeyExtractorStrategy customerIdExtractor() {
        return message -> {
            OrderEvent event = (OrderEvent) message.getPayload();
            return event.customerId();
        };
    }

    /**
     * Consumer: procesa pedidos de la partición asignada a esta instancia.
     * Configurado con partitioned: true e instanceIndex en application.yml.
     */
    @Bean
    public Function<Message<OrderEvent>, String> processOrder() {
        return message -> {
            OrderEvent order = message.getPayload();
            Integer partition = (Integer) message.getHeaders()
                .getOrDefault("kafka_receivedPartitionId", -1);
            System.out.printf("[Partition %d] Processing order %s for customer %s%n",
                partition, order.orderId(), order.customerId());
            return "processed:" + order.orderId();
        };
    }
}
```

**PartitioningApplication.java**
```java
package com.example.partitioning;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class PartitioningApplication {
    public static void main(String[] args) {
        SpringApplication.run(PartitioningApplication.class, args);
    }
}
```

## Tabla de elementos clave

El particionado se configura en el lado del producer (cómo asignar mensajes a particiones) y del consumer (cuántas instancias hay y cuál es esta). La tabla siguiente cubre ambas perspectivas.

| Parámetro | Tipo | Default | Descripción |
|-----------|------|---------|-------------|
| `spring.cloud.stream.bindings.<name>.producer.partition-count` | `int` | `1` | Número de particiones del topic destino. Debe coincidir con el número real de particiones del topic en el broker. |
| `spring.cloud.stream.bindings.<name>.producer.partition-key-expression` | `String` | (ninguno) | SpEL evaluada sobre el `Message` para extraer la clave de particionado. Ejemplo: `payload.customerId` o `headers['tenantId']`. |
| `spring.cloud.stream.bindings.<name>.producer.partition-key-extractor-name` | `String` | (ninguno) | Nombre del bean `PartitionKeyExtractorStrategy`. Alternativa programática a `partition-key-expression`. |
| `spring.cloud.stream.bindings.<name>.producer.partition-selector-name` | `String` | (ninguno) | Nombre del bean `PartitionSelectorStrategy` para selección custom de partición. Por defecto usa `key.hashCode() % partitionCount`. |
| `spring.cloud.stream.bindings.<name>.producer.partition-selector-expression` | `String` | (ninguno) | SpEL que recibe la clave de partición y retorna el índice de partición. Ejemplo: `Integer.parseInt(key.substring(0, 1)) % 3`. |
| `spring.cloud.stream.bindings.<name>.consumer.partitioned` | `boolean` | `false` | Habilita el consumo particionado en el consumer. Debe ser `true` cuando el producer usa particionado. |
| `spring.cloud.stream.instance-count` | `int` | `1` | Número total de instancias del consumer en el cluster. Usado para calcular qué particiones corresponden a esta instancia. |
| `spring.cloud.stream.instance-index` | `int` | `0` | Índice de esta instancia (0-based). Determina qué particiones asignará el binder a esta instancia. |

## Buenas y malas prácticas

**Hacer:**

- Mantener `partition-count` en la configuración del producer igual al número real de particiones del topic en Kafka. Si el topic tiene 6 particiones pero `partition-count` es 3, la mitad de las particiones del topic nunca recibirán mensajes, desperdiciando recursos y rompiendo el ordering esperado.
- Usar `partition-key-expression: "payload.<field>"` para claves de dominio simples. Para lógica compleja (hash de múltiples campos, normalización de claves), implementar `PartitionKeyExtractorStrategy` y referenciarla por nombre para mantener la expresión YAML limpia.
- En Kubernetes, inyectar `spring.cloud.stream.instance-index` desde el índice del pod (por ejemplo, `HOSTNAME=my-service-2` → `instanceIndex=2`). Esto permite el escalado automático sin cambios de configuración.

**Evitar:**

- Cambiar el número de particiones de un topic en producción sin coordinar el cambio con el producer y todos los consumers. Si se añaden particiones a un topic mientras el sistema está en producción, los mensajes de las claves que antes iban a la partición 0 pueden ir a una partición diferente (porque `hash(key) % newCount != hash(key) % oldCount`), rompiendo el ordering.
- Usar `partitioned: true` en un consumer sin configurar `instance-count` e `instance-index`. Sin estas propiedades, todos los consumers intentan recibir mensajes de todas las particiones, anulando el efecto del particionado y procesando mensajes duplicados.

---

← [6.6 Serialización y deserialización (SerDes)](sc-stream-serdes.md) | [Índice (README.md)](README.md) | [6.8 Gestión de errores y DLQ](sc-stream-error-handling.md) →
