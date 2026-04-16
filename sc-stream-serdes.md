# 6.6 Serialización y deserialización (SerDes)

← [6.5 Binder RabbitMQ — configuración y propiedades](sc-stream-binder-rabbit.md) | [Índice (README.md)](README.md) | [6.7 Particionado de mensajes](sc-stream-particionado.md) →

## Introducción

La serialización y deserialización de mensajes es uno de los puntos más frecuentes de fallos silenciosos en sistemas de mensajería distribuida. Spring Cloud Stream resuelve el problema mediante una cadena de `MessageConverter` que convierte automáticamente el payload Java a bytes (serialización) y de bytes a POJO (deserialización) basándose en el `contentType` del binding. Sin este mecanismo, el desarrollador tendría que serializar y deserializar manualmente en cada handler funcional, y los errores de tipo generarían `ClassCastException` en runtime sin mensajes claros.

La cadena de conversión opera en dos direcciones: cuando el handler produce un mensaje, el framework serializa el payload al `contentType` del binding de salida; cuando el handler recibe un mensaje, el framework deserializa el payload desde el formato del mensaje entrante al tipo declarado en la firma del bean funcional. El `contentType` del binding actúa como contrato entre productores y consumidores.

> [PREREQUISITO] El modelo de bindings (6.2). La configuración de SerDes es una propiedad del binding (`contentType`), no del binder.

## Representación visual

La cadena de conversión de Spring Cloud Stream sigue un orden determinista. El diagrama siguiente muestra el flujo de conversión en cada dirección.

```
SERIALIZACIÓN (handler produce → broker):
──────────────────────────────────────────
Java POJO (OrderEvent)
    │
    ▼  MessageConverter chain (en orden de registro)
    │  1. JsonMessageConverter (si contentType=application/json)
    │  2. ByteArrayMessageConverter (si contentType=application/octet-stream)
    │  3. StringMessageConverter (si contentType=text/plain)
    ▼
byte[] (payload del mensaje en el broker)

DESERIALIZACIÓN (broker → handler recibe):
───────────────────────────────────────────
byte[] (payload del mensaje en el broker)
    │
    ▼  MessageConverter chain (contenido del header content-type del mensaje)
    │  1. JsonMessageConverter → Object (mapeado al tipo de la firma del @Bean)
    │  2. ByteArrayMessageConverter → byte[]
    │  3. StringMessageConverter → String
    ▼
Java POJO (OrderEvent) — según tipo de parámetro del @Bean funcional

NATIVE ENCODING (modo sin conversión, binder Kafka):
─────────────────────────────────────────────────────
byte[] / String  ──►  Kafka Producer  ──►  Topic (sin pasar por MessageConverter)
                      (key.serializer / value.serializer nativos)
```

## Ejemplo central

El siguiente ejemplo muestra la configuración de tres modos de serialización: JSON automático con Jackson, Avro con Schema Registry, y native encoding para Kafka.

**pom.xml**
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-stream-binder-kafka</artifactId>
    </dependency>
    <!-- Para soporte de Avro con Schema Registry -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-stream-schema-registry-client</artifactId>
    </dependency>
    <dependency>
        <groupId>org.apache.avro</groupId>
        <artifactId>avro</artifactId>
        <version>1.11.3</version>
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
      definition: processJson;processNative
    stream:
      # Modo 1: JSON automático con Jackson (por defecto)
      bindings:
        processJson-in-0:
          destination: orders-json
          group: json-processors
          contentType: application/json      # Jackson ObjectMapper convierte automáticamente
        processJson-out-0:
          destination: orders-processed
          contentType: application/json
        # Modo 2: Native encoding para Kafka (sin MessageConverter)
        processNative-in-0:
          destination: orders-native
          group: native-processors
          # Sin contentType — el binder Kafka usa sus propios SerDes nativos
        processNative-out-0:
          destination: orders-native-out
      kafka:
        binder:
          brokers: localhost:9092
          # Para native encoding, configurar los SerDes en el nivel de binder
          configuration:
            value.deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
            value.serializer: org.springframework.kafka.support.serializer.JsonSerializer
            spring.json.trusted.packages: "com.example.serdes"
        bindings:
          processNative-in-0:
            consumer:
              # Desactivar la conversión de Spring Cloud Stream para usar los SerDes nativos de Kafka
              use-native-decoding: true
          processNative-out-0:
            producer:
              use-native-encoding: true
    # Modo 3: Avro con Schema Registry (Confluent)
    schema-registry-client:
      endpoint: http://localhost:8081
```

**OrderEvent.java**
```java
package com.example.serdes;

import com.fasterxml.jackson.annotation.JsonCreator;
import com.fasterxml.jackson.annotation.JsonProperty;

public class OrderEvent {
    private final String orderId;
    private final String customerId;
    private final double amount;

    @JsonCreator
    public OrderEvent(
        @JsonProperty("orderId") String orderId,
        @JsonProperty("customerId") String customerId,
        @JsonProperty("amount") double amount) {
        this.orderId = orderId;
        this.customerId = customerId;
        this.amount = amount;
    }

    public String getOrderId() { return orderId; }
    public String getCustomerId() { return customerId; }
    public double getAmount() { return amount; }
}
```

**SerDesConfig.java**
```java
package com.example.serdes;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.Message;
import org.springframework.messaging.support.MessageBuilder;

import java.util.function.Function;

@Configuration
public class SerDesConfig {

    /**
     * processJson: Jackson convierte automáticamente OrderEvent ↔ JSON bytes.
     * El contentType application/json del binding activa el JsonMessageConverter.
     */
    @Bean
    public Function<Message<OrderEvent>, Message<OrderEvent>> processJson() {
        return message -> {
            OrderEvent order = message.getPayload();
            // En este punto, order ya es un POJO Java desserializado automáticamente desde JSON
            System.out.printf("[JSON] Received orderId=%s amount=%.2f%n",
                order.getOrderId(), order.getAmount());
            return MessageBuilder.withPayload(order).build();
        };
    }

    /**
     * processNative: el binder Kafka usa sus propios SerDes (JsonDeserializer/JsonSerializer).
     * use-native-decoding: true desactiva el MessageConverter de Spring Cloud Stream.
     * El payload llega al handler ya deserializado por el cliente Kafka nativo.
     */
    @Bean
    public Function<Message<OrderEvent>, Message<OrderEvent>> processNative() {
        return message -> {
            OrderEvent order = message.getPayload();
            System.out.printf("[NATIVE] Received orderId=%s%n", order.getOrderId());
            return MessageBuilder.withPayload(order).build();
        };
    }
}
```

**SerDesApplication.java**
```java
package com.example.serdes;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class SerDesApplication {
    public static void main(String[] args) {
        SpringApplication.run(SerDesApplication.class, args);
    }
}
```

## Tabla de elementos clave

La serialización se controla principalmente a través de `contentType` en el binding y las propiedades de SerDes del binder Kafka. La siguiente tabla cubre los parámetros más relevantes.

| Parámetro | Tipo | Default | Descripción |
|-----------|------|---------|-------------|
| `spring.cloud.stream.bindings.<name>.contentType` | `String` | `application/json` | MIME type del payload. Determina qué `MessageConverter` del chain se activa. |
| `spring.cloud.stream.kafka.bindings.<name>.consumer.use-native-decoding` | `boolean` | `false` | Si `true`, desactiva el `MessageConverter` de Spring Cloud Stream y delega la deserialización al cliente Kafka nativo (`value.deserializer`). |
| `spring.cloud.stream.kafka.bindings.<name>.producer.use-native-encoding` | `boolean` | `false` | Si `true`, desactiva el `MessageConverter` de Spring Cloud Stream y delega la serialización al cliente Kafka nativo (`value.serializer`). |
| `spring.cloud.stream.kafka.binder.configuration.value.deserializer` | `String` | `ByteArrayDeserializer` | Clase del deserializador nativo de Kafka para el valor del mensaje. |
| `spring.cloud.stream.kafka.binder.configuration.spring.json.trusted.packages` | `String` | `*` | Packages Java de confianza para `JsonDeserializer`. Restringir en producción para evitar deserialización de clases arbitrarias. |
| `spring.cloud.schema-registry-client.endpoint` | `String` | (ninguno) | URL del Schema Registry (Confluent o Spring Cloud Schema Registry) para Avro. Ver https://docs.confluent.io |

## Buenas y malas prácticas

**Hacer:**

- Usar `contentType: application/json` con Jackson como configuración por defecto. Es el modo más sencillo, totalmente compatible con Spring Boot 4.x y no requiere configuración adicional. Jackson deserializa directamente al tipo declarado en la firma del `@Bean` funcional.
- Restringir `spring.json.trusted.packages` al package específico de los DTOs del proyecto. El default `*` permite deserializar cualquier clase Java en el classpath, lo que es un vector de ataque de deserialización insegura en entornos con mensajes no confiables.
- Usar `use-native-decoding/encoding` cuando el binder Kafka necesita interoperar con otros sistemas que no usan Spring Cloud Stream y tienen sus propios SerDes registrados. El `MessageConverter` de Spring añade una cabecera `contentType` en los mensajes que sistemas externos no esperan.

**Evitar:**

- Mezclar `contentType: application/json` en un binding con `use-native-decoding: true` en el mismo binding. Esta combinación es inconsistente: el `contentType` del binding controla el `MessageConverter`, pero con native decoding este converter está desactivado. Resulta en comportamiento impredecible según la versión del framework.
- Ignorar el `contentType` del mensaje entrante cuando difiere del `contentType` configurado en el binding. Si el producer envía `application/avro` pero el binding del consumer declara `application/json`, el `MessageConverter` lanzará una excepción de conversión que Spring Cloud Stream interpreta como error de procesamiento y dispara los reintentos, enviando finalmente el mensaje a la DLQ.

---

← [6.5 Binder RabbitMQ — configuración y propiedades](sc-stream-binder-rabbit.md) | [Índice (README.md)](README.md) | [6.7 Particionado de mensajes](sc-stream-particionado.md) →
