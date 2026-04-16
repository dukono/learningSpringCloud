# 12.7 Integración con Spring Cloud Stream: binding de funciones a canales de mensajería

← [12.6 Compilación a imagen nativa con GraalVM AOT](sc-function-native.md) | [Índice (README.md)](README.md) | [12.8 Puntos de extensión avanzados: FunctionAroundWrapper y FunctionRegistration](sc-function-extension.md) →

---

## Introducción

El problema que cubre este fichero es la unión entre el modelo funcional puro de Spring Cloud Function y la mensajería asíncrona de Spring Cloud Stream. Sin esta integración, el desarrollador debe escribir un `@StreamListener` o una función `@Bean` de Stream con acoplamiento explícito a los canales y a los tipos de mensaje del binder. La integración permite que un bean `Function<Order,Invoice>` se convierta automáticamente en un consumer-processor de Kafka o RabbitMQ sin modificar una línea de código de negocio.

El mecanismo de binding automático funciona por convención de nombres: si la función se llama `processOrder`, Spring Cloud Stream genera automáticamente los bindings `processOrder-in-0` (input channel, consumidor de Kafka/RabbitMQ) y `processOrder-out-0` (output channel, productor). Esta convención es el contrato que une ambos módulos. Cuando el binding automático no es suficiente (nombres de topic no convencionales, múltiples inputs o outputs), se usa `spring.cloud.stream.function.bindings` para el mapeo explícito.

> **[PREREQUISITO]** Este fichero presupone conocimiento del módulo Spring Cloud Stream completo (módulo 6 del índice general, capítulo `sc-stream-*`), en especial `sc-stream-modelo-funcional.md` y `sc-stream-bindings.md`. Spring Cloud Function es el proveedor de la lógica de función; Spring Cloud Stream gestiona el binder, el particionado, los consumer groups y los reintentos.

> **[CONCEPTO]** Spring Cloud Stream delega internamente en Spring Cloud Function para resolver los beans funcionales. Cuando ambos módulos están en el classpath, `FunctionCatalog` es compartido. El particionado, los consumer groups y el manejo de errores con DLQ son responsabilidad del binder de Spring Cloud Stream; ver módulo 6 para su configuración.

## Representación visual

El binding de funciones a canales sigue la convención `<nombre>-in-<índice>` y `<nombre>-out-<índice>`. El índice permite múltiples inputs y outputs para una misma función reactiva.

```
  Kafka (binder)
  ┌──────────────────────┐    ┌─────────────────────────┐
  │  Topic: orders       │    │  Topic: invoices         │
  │  (input)             │    │  (output)                │
  └──────────┬───────────┘    └────────────▲────────────┘
             │                             │
             │  processOrder-in-0          │  processOrder-out-0
             ▼                             │
  ┌──────────────────────────────────────────────────────┐
  │  Spring Cloud Stream Binder (Kafka)                  │
  │                                                      │
  │  Desserializa Order ──────► Function<Order,Invoice>  │
  │                              .apply(order)           │
  │                                 │                    │
  │  Serializa Invoice ◄────────────┘                    │
  └──────────────────────────────────────────────────────┘
             │
  Mapeo explícito (opcional):
  spring.cloud.stream.function.bindings.processOrder-in-0=ordersInput
  spring.cloud.stream.function.bindings.processOrder-out-0=invoicesOutput
```

## Ejemplo central

El ejemplo muestra una función que consume pedidos de Kafka, los transforma en facturas y los publica en otro topic. Incluye la configuración de binding automático y el mapeo explícito de topics con nombre no convencional.

### Dependencias Maven

```xml
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

<dependencies>
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-function-context</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-stream-kafka</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>
  </dependency>
  <dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
  </dependency>
</dependencies>
```

### application.yml

```yaml
spring:
  cloud:
    function:
      # Activa processOrder como función de Stream; genera bindings automáticos
      definition: processOrder
    stream:
      # Mapeo explícito de binding a nombre de topic real
      function:
        bindings:
          processOrder-in-0: ordersInput
          processOrder-out-0: invoicesOutput
      bindings:
        # Canal de entrada: consume del topic Kafka "domain.orders"
        ordersInput:
          destination: domain.orders
          group: invoice-service
          consumer:
            max-attempts: 3
        # Canal de salida: publica en el topic Kafka "domain.invoices"
        invoicesOutput:
          destination: domain.invoices
          producer:
            partition-count: 3
      kafka:
        binder:
          brokers: localhost:9092
        bindings:
          ordersInput:
            consumer:
              enable-dlq: true
              dlq-name: domain.orders.dlq
```

### Código Java completo

```java
package com.example.scfunction.stream;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;

import java.util.function.Consumer;
import java.util.function.Function;
import java.util.function.Supplier;

@SpringBootApplication
public class FunctionStreamApplication {

    public static void main(String[] args) {
        SpringApplication.run(FunctionStreamApplication.class, args);
    }

    public record Order(Long id, String item, double price) {}
    public record Invoice(Long orderId, String description, double total) {}

    // Function: Consumer de "domain.orders" → Producer de "domain.invoices"
    // Binding automático: processOrder-in-0 y processOrder-out-0
    @Bean
    public Function<Order, Invoice> processOrder() {
        return order -> {
            System.out.println("[STREAM] Procesando pedido: " + order.id());
            return new Invoice(
                order.id(),
                "Factura Stream: " + order.item(),
                order.price() * 1.21
            );
        };
    }

    // Consumer: solo consume, no produce (sink)
    // Binding automático: auditLog-in-0 (sin -out-0)
    @Bean
    public Consumer<Order> auditLog() {
        return order -> System.out.println("[AUDIT] Pedido recibido: " + order.id());
    }

    // Supplier: produce mensajes periódicamente (source)
    // Binding automático: heartbeat-out-0 (sin -in-0)
    // Se invoca cada segundo por defecto (configurable con spring.cloud.stream.poller.*)
    @Bean
    public Supplier<String> heartbeat() {
        return () -> "heartbeat-" + System.currentTimeMillis();
    }
}
```

Convención de nombres de binding para los tres contratos:

| Bean | Tipo | Binding generado |
|---|---|---|
| `processOrder` | `Function<Order,Invoice>` | `processOrder-in-0` + `processOrder-out-0` |
| `auditLog` | `Consumer<Order>` | `auditLog-in-0` (solo input) |
| `heartbeat` | `Supplier<String>` | `heartbeat-out-0` (solo output) |

> **[EXAMEN]** La propiedad `spring.cloud.stream.function.bindings` mapea el nombre lógico del binding (`processOrder-in-0`) al nombre de binding definido en `spring.cloud.stream.bindings`. Esto permite usar nombres de topic con guiones, puntos u otros caracteres que no son válidos en nombres de beans Java. Sin este mapeo, el binding se genera con el nombre del bean directamente.

## Tabla de elementos clave

La siguiente tabla recoge los conceptos de integración entre Spring Cloud Function y Spring Cloud Stream relevantes para entrevistas.

| Elemento | Descripción |
|---|---|
| `<fn>-in-0` | Convención de nombre del binding de entrada para la función `fn`; el índice permite múltiples inputs |
| `<fn>-out-0` | Convención de nombre del binding de salida para la función `fn` |
| `spring.cloud.stream.function.bindings` | Mapeo explícito de binding lógico a nombre de binding de Stream |
| `spring.cloud.function.definition` | Activa la función en el catálogo; obligatorio con múltiples beans |
| `group` | Consumer group de Kafka/RabbitMQ; garantiza que solo una instancia del grupo procesa cada mensaje |
| `enable-dlq` | Activa Dead Letter Queue en Kafka para mensajes que superan `max-attempts` |
| `spring.cloud.stream.poller.*` | Configura la frecuencia de polling para `Supplier` beans (por defecto 1 segundo) |
| Reactivo con `Flux<T>` | `Function<Flux<Order>,Flux<Invoice>>` procesa mensajes en streaming reactivo sin bloquear hilos |

## Buenas y malas prácticas

**Hacer:**
- Definir `group` en todos los bindings de entrada en producción: sin consumer group, cada instancia del servicio consume todos los mensajes del topic de forma independiente (comportamiento broadcast), duplicando el procesamiento.
- Usar `Function<Flux<Order>,Flux<Invoice>>` cuando el procesamiento requiera operaciones de ventana, agregación o batching reactivo: el modelo reactivo evita el bloqueo de hilos y escala mejor bajo alta concurrencia.
- Configurar `enable-dlq: true` en producción para mensajes que fallen repetidamente tras `max-attempts` reintentos: sin DLQ, los mensajes fallidos bloquean la partición en Kafka o se pierden silenciosamente en RabbitMQ.
- Usar `spring.cloud.stream.function.bindings` cuando los nombres de topic contengan caracteres no válidos en nombres de beans Java (puntos, guiones): el mapeo explícito evita sorpresas en la convención de nombres.

**Evitar:**
- Evitar usar el mismo bean funcional simultáneamente en un binding de Stream y en la exposición HTTP (`spring-cloud-function-web`): aunque técnicamente posible, produce confusión en el modelo de invocación y puede generar procesamiento duplicado si ambos adaptadores están activos.
- Evitar `Supplier` beans sin control de rate (sin `spring.cloud.stream.poller.fixed-delay`): por defecto el Supplier se invoca cada segundo independientemente de si hay consumidores disponibles, produciendo mensajes no procesados que acumulan lag en el topic.
- Evitar omitir la configuración del binder en `application.yml` (brokers, topic names) asumiendo valores por defecto: los defaults de Kafka son para desarrollo local y no son adecuados para producción (particiones, replication factor, timeouts).

---

← [12.6 Compilación a imagen nativa con GraalVM AOT](sc-function-native.md) | [Índice (README.md)](README.md) | [12.8 Puntos de extensión avanzados: FunctionAroundWrapper y FunctionRegistration](sc-function-extension.md) →
