# 6.4 Kafka Streams binder

← [6.3 Binder Kafka — configuración y propiedades](sc-stream-binder-kafka.md) | [Índice (README.md)](README.md) | [6.5 Binder RabbitMQ — configuración y propiedades](sc-stream-binder-rabbit.md) →

## Introducción

El binder de Kafka Streams es un sub-binder independiente del binder Kafka estándar, aunque comparte el prefijo de configuración `spring.cloud.stream.kafka.*`. Resuelve un problema diferente: mientras el binder Kafka estándar gestiona producción y consumo simple de mensajes, el binder Kafka Streams permite construir pipelines de procesamiento stateful directamente sobre la topología de Kafka Streams, incluyendo agregaciones, joins entre streams y tablas, y ventanas temporales. Sin este binder, un desarrollador tendría que configurar manualmente `StreamsBuilder`, `KafkaStreams` y gestionar el ciclo de vida de los state stores.

La distinción es crítica: el binder Kafka estándar es para casos de uso event-driven simples (consumir, transformar, producir); el binder Kafka Streams es para casos que requieren estado local en el consumidor (contadores, joins temporales, detección de patrones). Mezclar los dos en un mismo binding es un error frecuente que genera comportamientos inesperados.

> [PREREQUISITO] El binder Kafka estándar (6.3) y el concepto de consumer group. El binder Kafka Streams comparte la misma capa de bindings pero tiene un ciclo de vida y API completamente distintos.

> [CONCEPTO] La construcción de Topology y la DSL completa de Kafka Streams quedan fuera del alcance de este módulo. Este binder expone KStream, KTable y GlobalKTable como tipos de parámetro en los @Bean; para la API nativa de Kafka Streams ver https://kafka.apache.org/documentation/streams/

## Representación visual

La topología de Kafka Streams se diferencia del modelo funcional estándar en que los tipos de parámetro son nativos de Kafka Streams (`KStream`, `KTable`, `GlobalKTable`), y el binder gestiona internamente el `StreamsBuilder` y el ciclo de vida de `KafkaStreams`.

```
┌─────────────────────────────────────────────────────────┐
│  Binder Kafka Streams                                   │
│                                                         │
│   KStream<K,V>   ──►  stream de registros individuales  │
│                       sin estado (por defecto)           │
│                                                         │
│   KTable<K,V>    ──►  tabla materializada del último    │
│                       valor por clave (changelog topic) │
│                                                         │
│   GlobalKTable<K,V> ► tabla replicada en TODAS las      │
│                       instancias del cluster            │
└──────────┬─────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────┐
│  @Bean Function<KStream<String,OrderEvent>,             │
│                 KStream<String,OrderResult>>            │
│                                                         │
│  @Bean BiFunction<KStream<String,OrderEvent>,           │
│                   KTable<String,Customer>,              │
│                   KStream<String,EnrichedOrder>>        │
└─────────────────────────────────────────────────────────┘
```

La diferencia entre `KTable` y `GlobalKTable` determina el comportamiento del join: con `KTable`, el join solo ocurre con los registros de la misma partición; con `GlobalKTable`, el join funciona independientemente de la partición del stream de entrada.

## Ejemplo central

El siguiente ejemplo implementa un servicio que enriquece pedidos con datos de clientes usando un join entre un `KStream` de pedidos y una `KTable` de clientes.

**pom.xml**
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-stream-binder-kafka-streams</artifactId>
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
      definition: enrichOrders
    stream:
      bindings:
        # BiFunction tiene dos entradas: in-0 y in-1
        enrichOrders-in-0:
          destination: orders-raw
          group: orders-enrichment
        enrichOrders-in-1:
          destination: customers
          group: orders-enrichment
        enrichOrders-out-0:
          destination: orders-enriched
      kafka:
        streams:
          binder:
            brokers: localhost:9092
            application-id: orders-enrichment-app  # ID de la aplicación Kafka Streams (state store)
            configuration:
              default.key.serde: org.apache.kafka.common.serialization.Serdes$StringSerde
              default.value.serde: org.springframework.kafka.support.serializer.JsonSerde
              commit.interval.ms: 1000
```

**OrderEvent.java**
```java
package com.example.kstreams;

public record OrderEvent(String orderId, String customerId, double amount) {}
```

**Customer.java**
```java
package com.example.kstreams;

public record Customer(String customerId, String name, String tier) {}
```

**EnrichedOrder.java**
```java
package com.example.kstreams;

public record EnrichedOrder(String orderId, String customerId, String customerName,
                             String customerTier, double amount) {}
```

**OrderEnrichmentConfig.java**
```java
package com.example.kstreams;

import org.apache.kafka.streams.kstream.KStream;
import org.apache.kafka.streams.kstream.KTable;
import org.apache.kafka.streams.kstream.Joined;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.function.BiFunction;

@Configuration
public class OrderEnrichmentConfig {

    /**
     * BiFunction<KStream<K,V1>, KTable<K,V2>, KStream<K,R>>
     * Une pedidos (stream) con clientes (tabla) por customerId.
     *
     * Spring Cloud Stream infiere:
     *   - enrichOrders-in-0  → KStream (orders-raw)
     *   - enrichOrders-in-1  → KTable (customers)
     *   - enrichOrders-out-0 → KStream de salida (orders-enriched)
     */
    @Bean
    public BiFunction<KStream<String, OrderEvent>, KTable<String, Customer>,
                      KStream<String, EnrichedOrder>> enrichOrders() {
        return (ordersStream, customersTable) ->
            ordersStream
                .selectKey((key, order) -> order.customerId())  // re-keyed por customerId para el join
                .join(
                    customersTable,
                    (order, customer) -> new EnrichedOrder(
                        order.orderId(),
                        customer != null ? customer.customerId() : "UNKNOWN",
                        customer != null ? customer.name() : "UNKNOWN",
                        customer != null ? customer.tier() : "BASIC",
                        order.amount()
                    ),
                    Joined.as("orders-customers-join")
                );
    }
}
```

**KStreamsApplication.java**
```java
package com.example.kstreams;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class KStreamsApplication {
    public static void main(String[] args) {
        SpringApplication.run(KStreamsApplication.class, args);
    }
}
```

## Tabla de elementos clave

Las propiedades del binder Kafka Streams se configuran bajo `spring.cloud.stream.kafka.streams.binder.*` y son independientes de las del binder Kafka estándar.

| Parámetro | Tipo | Default | Descripción |
|-----------|------|---------|-------------|
| `spring.cloud.stream.kafka.streams.binder.brokers` | `String` | `localhost` | Lista de brokers Kafka para el binder Kafka Streams. Independiente del binder Kafka estándar. |
| `spring.cloud.stream.kafka.streams.binder.application-id` | `String` | nombre de la app | ID de la aplicación Kafka Streams. Determina el nombre de los state stores y los consumer groups internos. Debe ser único por aplicación en el clúster. |
| `spring.cloud.stream.kafka.streams.binder.configuration.commit.interval.ms` | `long` | `100` | Intervalo de commit del estado interno. Valores más altos mejoran el throughput pero aumentan el tiempo de recuperación ante fallos. |
| `spring.cloud.stream.kafka.streams.binder.configuration.default.key.serde` | `String` | StringSerde | SerDe por defecto para las claves. |
| `spring.cloud.stream.kafka.streams.binder.configuration.default.value.serde` | `String` | ByteArraySerde | SerDe por defecto para los valores. Configurar JsonSerde para POJOs. |
| `spring.cloud.stream.kafka.streams.binder.state-store-retry` | `boolean` | `false` | Habilitar retry en la inicialización del state store (útil en entornos con Kafka intermitente). |

## Buenas y malas prácticas

**Hacer:**

- Configurar `application-id` explícitamente. Si no se configura, Kafka Streams usa el nombre de la aplicación Spring Boot, que puede coincidir con otras instancias del mismo servicio y generar conflictos en los state stores internos.
- Usar `GlobalKTable` cuando el volumen de la tabla de referencia es pequeño (< 100MB) y se necesita un join desde cualquier partición del stream. Con `KTable` estándar, el join solo funciona si stream y tabla comparten la misma partición por clave, lo que requiere que ambos topics tengan el mismo número de particiones y la misma estrategia de particionado.
- Diseñar las funciones Kafka Streams como stateless cuando sea posible. Los state stores locales requieren restauración al reiniciar el servicio, lo que puede tardar minutos en topics con alto volumen de cambios.

**Evitar:**

- Mezclar el binder Kafka Streams con el binder Kafka estándar en el mismo bean funcional. Son implementaciones completamente separadas con ciclos de vida distintos. Si se necesitan ambos en un mismo servicio, se configura multi-binder (6.11).
- Ignorar el `commit.interval.ms`. Con el valor por defecto de 100ms, Kafka Streams hace commits 10 veces por segundo por partición. En topics con miles de particiones, esto puede saturar los logs de Kafka con commits y degradar el throughput global del clúster.

## Comparación: KStream vs KTable vs GlobalKTable

La elección del tipo de abstracción Kafka Streams determina la semántica del procesamiento y los requisitos de infraestructura.

| Tipo | Semántica | Particionado | Caso de uso |
|------|-----------|-------------|-------------|
| `KStream<K,V>` | Stream de todos los registros, sin estado | Distribuido entre instancias | Transformaciones, filtros, aggregaciones con ventana |
| `KTable<K,V>` | Última versión por clave (compacted view) | Misma partición que el stream de join | Join con tabla de referencia particionada |
| `GlobalKTable<K,V>` | Tabla completa replicada en todas las instancias | Independiente | Join con tabla de referencia pequeña y global |

---

← [6.3 Binder Kafka — configuración y propiedades](sc-stream-binder-kafka.md) | [Índice (README.md)](README.md) | [6.5 Binder RabbitMQ — configuración y propiedades](sc-stream-binder-rabbit.md) →
