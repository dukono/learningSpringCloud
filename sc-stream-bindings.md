# 6.2 Bindings y binders

← [6.1 Modelo de programación funcional](sc-stream-modelo-funcional.md) | [Índice (README.md)](README.md) | [6.3 Binder Kafka — configuración y propiedades](sc-stream-binder-kafka.md) →

## Introducción

Spring Cloud Stream introduce dos abstracciones fundamentales que desacoplan el código de la aplicación del sistema de mensajería concreto: el **binding** y el **binder**. El problema que resuelven es el acoplamiento tecnológico: sin estas abstracciones, cambiar de Kafka a RabbitMQ requeriría reescribir la lógica de producción y consumo de mensajes. Con el modelo de bindings, el código de la aplicación trabaja siempre con canales lógicos; el binder traduce esos canales a los conceptos nativos del broker (topics en Kafka, exchanges y queues en RabbitMQ).

Un **binding** es el canal lógico que conecta una función (`Function`, `Consumer` o `Supplier`) con el broker. Tiene un nombre determinista basado en la convención `<functionName>-in-<index>` para entradas y `<functionName>-out-<index>` para salidas. Un **binder** es la implementación concreta que traduce ese canal lógico a la API del broker (el binder de Kafka usa el cliente Kafka nativo; el binder de RabbitMQ usa el cliente AMQP de Spring).

Este conocimiento es prerequisito obligatorio antes de configurar cualquier binder específico (6.3, 6.4, 6.5). Sin entender la capa de binding y el naming convention, la configuración de propiedades por binder resulta incomprensible.

> [PREREQUISITO] Haber leído 6.1 (Modelo de programación funcional). El concepto de binding nace del modelo funcional.

## Representación visual

La separación en capas permite sustituir el binder sin tocar la lógica de negocio. El diagrama siguiente muestra las tres capas y sus responsabilidades.

```
┌─────────────────────────────────────────────────────────┐
│                    APLICACIÓN                            │
│                                                         │
│   @Bean Function<OrderEvent, Confirmation>              │
│   processOrder() { ... }                                │
└──────────────┬──────────────────────────┬───────────────┘
               │                          │
               ▼ input binding            ▼ output binding
┌─────────────────────┐       ┌─────────────────────────┐
│ processOrder-in-0   │       │ processOrder-out-0       │
│ destination: orders │       │ destination: confirmations│
│ group: processors   │       │ contentType: applic/json │
└──────────┬──────────┘       └───────────┬─────────────┘
           │                              │
           ▼                              ▼
┌─────────────────────────────────────────────────────────┐
│                       BINDER                            │
│                                                         │
│   Binder Kafka         │    Binder RabbitMQ             │
│   (spring-cloud-stream │    (spring-cloud-stream        │
│   -binder-kafka)       │    -binder-rabbit)             │
└──────────┬─────────────┴──────────────┬─────────────────┘
           │                            │
           ▼                            ▼
    Kafka (topics)              RabbitMQ (exchanges/queues)
```

El índice del binding (`-in-0`, `-out-0`) permite declarar múltiples entradas o salidas para una misma función. `-in-1` sería una segunda entrada independiente para la misma función.

## Ejemplo central

El siguiente proyecto muestra cómo configurar bindings con sus propiedades principales en un servicio que consume pedidos, los procesa y emite resultados a un topic diferente.

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
      # Declaración explícita: obligatoria cuando hay múltiples beans funcionales
      definition: processOrder;auditEvent
    stream:
      # Valores por defecto aplicados a todos los bindings (override individual posible)
      default:
        contentType: application/json
      bindings:
        # Binding de entrada para processOrder
        processOrder-in-0:
          destination: orders-raw        # topic Kafka al que se suscribe
          group: order-processors        # consumer group — mensajes compartidos entre instancias
          contentType: application/json  # override del default
          consumer:
            max-attempts: 3              # reintentos antes de enviar a DLQ
            back-off-initial-interval: 1000
        # Binding de salida para processOrder
        processOrder-out-0:
          destination: orders-confirmed
          producer:
            required-groups: audit-group # garantiza que el topic se crea con este grupo
        # Binding de entrada para auditEvent (Consumer sin salida)
        auditEvent-in-0:
          destination: orders-confirmed
          group: audit-service
      kafka:
        binder:
          brokers: localhost:9092
```

**OrderEvent.java**
```java
package com.example.bindings;

public record OrderEvent(String orderId, double amount, String currency) {}
```

**OrderConfirmation.java**
```java
package com.example.bindings;

public record OrderConfirmation(String orderId, String status) {}
```

**BindingsConfig.java**
```java
package com.example.bindings;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.Message;
import org.springframework.messaging.support.MessageBuilder;

import java.util.function.Consumer;
import java.util.function.Function;

@Configuration
public class BindingsConfig {

    /**
     * processOrder: lee de orders-raw (processOrder-in-0)
     *               escribe en orders-confirmed (processOrder-out-0)
     */
    @Bean
    public Function<Message<OrderEvent>, Message<OrderConfirmation>> processOrder() {
        return message -> {
            OrderEvent order = message.getPayload();
            String status = order.amount() > 0 ? "ACCEPTED" : "REJECTED";
            OrderConfirmation confirmation = new OrderConfirmation(order.orderId(), status);
            return MessageBuilder
                .withPayload(confirmation)
                .copyHeaders(message.getHeaders())
                .build();
        };
    }

    /**
     * auditEvent: consume orders-confirmed (auditEvent-in-0) sin producir salida.
     */
    @Bean
    public Consumer<Message<OrderConfirmation>> auditEvent() {
        return message -> {
            System.out.printf("[AUDIT] Order %s -> %s%n",
                message.getPayload().orderId(),
                message.getPayload().status());
        };
    }
}
```

**BindingsApplication.java**
```java
package com.example.bindings;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class BindingsApplication {
    public static void main(String[] args) {
        SpringApplication.run(BindingsApplication.class, args);
    }
}
```

## Tabla de elementos clave

La configuración de bindings se articula en tres niveles: propiedades comunes a todos los bindings (`default`), propiedades de binding específico, y sub-propiedades de producer o consumer. La tabla siguiente cubre las más relevantes para un profesional senior.

| Parámetro | Tipo | Default | Descripción |
|-----------|------|---------|-------------|
| `spring.cloud.stream.bindings.<name>.destination` | `String` | nombre del binding | Nombre del topic (Kafka) o exchange (RabbitMQ) al que se conecta el binding. |
| `spring.cloud.stream.bindings.<name>.group` | `String` | `null` | Consumer group. Null implica subscripción anónima (fan-out): cada instancia recibe todos los mensajes. Con group definido, las instancias compiten por los mensajes. |
| `spring.cloud.stream.bindings.<name>.contentType` | `String` | `application/json` | MIME type para la serialización automática del payload. Soporte nativo: `application/json`, `text/plain`, `application/octet-stream`. |
| `spring.cloud.stream.default.contentType` | `String` | `application/json` | Valor por defecto para todos los bindings; se puede sobreescribir por binding individual. |
| `spring.cloud.stream.bindings.<name>.consumer.max-attempts` | `int` | `3` | Número máximo de reintentos antes de enviar el mensaje al canal de errores o DLQ. |
| `spring.cloud.stream.bindings.<name>.consumer.back-off-initial-interval` | `long` (ms) | `1000` | Intervalo inicial de backoff exponencial entre reintentos. |
| `spring.cloud.stream.bindings.<name>.consumer.back-off-max-interval` | `long` (ms) | `10000` | Intervalo máximo de backoff. El backoff no superará este valor aunque el multiplicador lo calcule mayor. |
| `spring.cloud.stream.bindings.<name>.consumer.back-off-multiplier` | `double` | `2.0` | Factor multiplicador del intervalo entre reintentos sucesivos. |
| `spring.cloud.stream.bindings.<name>.producer.required-groups` | `String[]` | `[]` | Grupos de consumidores que deben existir antes de que el producer envíe mensajes. Útil para garantizar que no se pierden mensajes en arranques. |
| `spring.cloud.stream.bindings.<name>.binder` | `String` | binder por defecto | Nombre del binder concreto a usar para este binding. Relevante en configuraciones multi-binder (6.11). |

## Buenas y malas prácticas

**Hacer:**

- Configurar siempre `group` en los bindings de entrada. Un binding sin group crea una suscripción anónima única por instancia: si el servicio escala a 3 instancias, cada una recibirá todos los mensajes (fan-out incontrolado), triplicando el procesamiento.
- Usar `spring.cloud.stream.default.contentType` para fijar el tipo de serialización global y sobreescribir solo en los bindings que requieran un formato diferente. Esto evita inconsistencias de serialización difíciles de diagnosticar.
- Nombrar los `destination` con nombres de dominio explícitos (`orders-raw`, `orders-confirmed`) en lugar de usar el nombre del binding por defecto. El nombre del binding cambia si se refactoriza el nombre del bean; el nombre del topic en el broker no debe cambiar con el código.

**Evitar:**

- Depender de la detección automática del bean funcional (sin `spring.cloud.function.definition`). En aplicaciones con múltiples dependencias que aportan beans al contexto de Spring, la detección automática puede enlazar el binding a un bean incorrecto sin lanzar error visible.
- Configurar `max-attempts: 1` como forma de "desactivar reintentos". Esta configuración no desactiva el error channel; simplemente envía el mensaje al DLQ (o lo descarta si no hay DLQ) en el primer fallo, lo que puede causar pérdida de datos en fallos transitorios de red.
- Reutilizar el mismo nombre de `destination` para un binding de entrada y uno de salida del mismo servicio sin consumer group. Esto crea un ciclo de retroalimentación: los mensajes que produce el servicio son consumidos por él mismo, generando un bucle infinito en producción.

## Comparación: binding con group vs sin group

El comportamiento del sistema difiere radicalmente según se defina o no el consumer group. Esta es una de las decisiones de configuración con mayor impacto en la semántica del sistema.

| Característica | Sin group (`group: null`) | Con group (`group: my-service`) |
|----------------|--------------------------|----------------------------------|
| Nombre de queue en Kafka | Grupo aleatorio por instancia | Nombre fijo del grupo |
| Comportamiento con 3 instancias | Cada instancia recibe todos los mensajes | Las 3 instancias comparten los mensajes (load balancing) |
| Mensajes en parada del servicio | Se pierden (suscripción anónima eliminada) | Se retienen hasta que el servicio vuelva (offset guardado) |
| Caso de uso | Fan-out: broadcast a todos los consumidores | Competing consumers: procesamiento distribuido |
| `auto.offset.reset` efectivo | `latest` (mensajes históricos perdidos) | Configurable (`earliest` o `latest`) |

> [EXAMEN] En entrevistas senior es habitual preguntar: "¿Qué ocurre con los mensajes si un servicio sin consumer group se detiene durante 5 minutos?" Respuesta correcta: se pierden. La suscripción anónima de Kafka (grupo efímero) solo retiene mensajes mientras hay al menos un consumidor activo. Con group definido, el offset queda guardado en Kafka y el servicio recupera los mensajes atrasados al reiniciarse.

---

← [6.1 Modelo de programación funcional](sc-stream-modelo-funcional.md) | [Índice (README.md)](README.md) | [6.3 Binder Kafka — configuración y propiedades](sc-stream-binder-kafka.md) →
