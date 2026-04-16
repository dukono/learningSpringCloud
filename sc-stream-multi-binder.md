# 6.11 Multi-binder

← [6.10 StreamBridge — producción de mensajes programática](sc-stream-streambridge.md) | [Índice (README.md)](README.md) | [6.12 Actuator y monitorización de bindings](sc-stream-actuator.md) →

## Introducción

El multi-binder de Spring Cloud Stream resuelve el problema de un microservicio que necesita conectarse a múltiples sistemas de mensajería simultáneamente. Esto ocurre en escenarios de migración (el servicio lee de RabbitMQ legacy y escribe en Kafka nuevo), en integraciones con sistemas externos heterogéneos (Kafka para eventos internos, RabbitMQ para notificaciones a terceros), o en arquitecturas donde diferentes dominios usan diferentes brokers por razones de compliance o coste.

Sin soporte de multi-binder, el desarrollador tendría que implementar clientes manuales para cada sistema de mensajería y gestionar su ciclo de vida fuera de Spring Cloud Stream. Con multi-binder, cada binding se asigna a un binder con nombre, y cada binder tiene su propia configuración de conexión aislada del resto.

> [PREREQUISITO] Binder Kafka (6.3) y Binder RabbitMQ (6.5). El multi-binder combina ambos en un mismo servicio.

## Representación visual

La configuración de multi-binder introduce un nivel adicional en la jerarquía de configuración: la sección `spring.cloud.stream.binders` define los binders disponibles, y cada binding declara explícitamente a qué binder pertenece.

```
spring.cloud.stream.binders
├── kafka-primary         (type: kafka)
│   └── environment.*     (brokers, SASL, etc.)
└── rabbit-notifications  (type: rabbit)
    └── environment.*     (host, port, virtual-host, etc.)

spring.cloud.stream.bindings
├── processOrder-in-0
│   └── binder: kafka-primary        ──► conecta al binder kafka-primary
├── processOrder-out-0
│   └── binder: kafka-primary
└── sendNotification-out-0
    └── binder: rabbit-notifications ──► conecta al binder rabbit-notifications

Resultado:
  processOrder lee/escribe de/a Kafka (kafka-primary)
  sendNotification escribe a RabbitMQ (rabbit-notifications)
```

## Ejemplo central

El siguiente ejemplo implementa un servicio que consume pedidos de Kafka, los procesa y produce tanto en Kafka (evento de confirmación) como en RabbitMQ (notificación a un sistema externo).

**pom.xml**
```xml
<dependencies>
    <!-- Ambos binders en el classpath -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-stream-binder-kafka</artifactId>
    </dependency>
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
      definition: processOrder;sendNotification
    stream:
      # Definición de binders disponibles
      binders:
        kafka-primary:
          type: kafka                         # tipo de binder (kafka, rabbit)
          environment:
            spring:
              cloud:
                stream:
                  kafka:
                    binder:
                      brokers: kafka-host:9092
                      replication-factor: 3
                      auto-create-topics: false
        rabbit-notifications:
          type: rabbit                        # binder de tipo RabbitMQ
          environment:
            spring:
              rabbitmq:
                host: rabbit-host
                port: 5672
                username: appuser
                password: "${RABBIT_PASSWORD}"
                virtual-host: /notifications
      # Asignación de bindings a binders específicos
      bindings:
        processOrder-in-0:
          destination: orders-raw
          group: order-processors
          binder: kafka-primary              # asignar a kafka-primary
        processOrder-out-0:
          destination: orders-confirmed
          binder: kafka-primary
        sendNotification-in-0:
          destination: orders-confirmed      # consume de Kafka lo que processOrder produce
          group: notification-sender
          binder: kafka-primary
        sendNotification-out-0:
          destination: order-notifications   # produce a RabbitMQ
          binder: rabbit-notifications
      kafka:
        binder:
          # Estas propiedades aplican al binder Kafka por defecto (si hubiera uno sin 'environment')
          brokers: kafka-host:9092
```

**OrderEvent.java**
```java
package com.example.multibinder;

public record OrderEvent(String orderId, String customerId, double amount) {}
```

**OrderConfirmation.java**
```java
package com.example.multibinder;

public record OrderConfirmation(String orderId, String status, long timestamp) {}
```

**Notification.java**
```java
package com.example.multibinder;

public record Notification(String orderId, String message, String channel) {}
```

**MultiBiniderConfig.java**
```java
package com.example.multibinder;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.Message;
import org.springframework.messaging.support.MessageBuilder;

import java.time.Instant;
import java.util.function.Function;

@Configuration
public class MultiBiniderConfig {

    /**
     * processOrder: lee de Kafka (kafka-primary), escribe en Kafka (kafka-primary).
     * Bindings: processOrder-in-0 y processOrder-out-0, ambos asignados a kafka-primary.
     */
    @Bean
    public Function<Message<OrderEvent>, Message<OrderConfirmation>> processOrder() {
        return message -> {
            OrderEvent order = message.getPayload();
            OrderConfirmation confirmation = new OrderConfirmation(
                order.orderId(),
                order.amount() > 0 ? "CONFIRMED" : "REJECTED",
                Instant.now().toEpochMilli()
            );
            return MessageBuilder
                .withPayload(confirmation)
                .setHeader("X-Source-Binder", "kafka-primary")
                .build();
        };
    }

    /**
     * sendNotification: lee de Kafka (kafka-primary), escribe en RabbitMQ (rabbit-notifications).
     * Bindings: sendNotification-in-0 (kafka-primary) y sendNotification-out-0 (rabbit-notifications).
     */
    @Bean
    public Function<Message<OrderConfirmation>, Message<Notification>> sendNotification() {
        return message -> {
            OrderConfirmation confirmation = message.getPayload();
            String notificationMsg = String.format(
                "Order %s has been %s",
                confirmation.orderId(),
                confirmation.status()
            );
            Notification notification = new Notification(
                confirmation.orderId(),
                notificationMsg,
                "EMAIL"
            );
            return MessageBuilder
                .withPayload(notification)
                .setHeader("notification-type", "ORDER_STATUS")
                .build();
        };
    }
}
```

**MultiBiniderApplication.java**
```java
package com.example.multibinder;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class MultiBiniderApplication {
    public static void main(String[] args) {
        SpringApplication.run(MultiBiniderApplication.class, args);
    }
}
```

## Tabla de elementos clave

La configuración de multi-binder se articula principalmente en la sección `spring.cloud.stream.binders`. La tabla siguiente cubre las propiedades más relevantes.

| Parámetro | Tipo | Default | Descripción |
|-----------|------|---------|-------------|
| `spring.cloud.stream.binders.<name>.type` | `String` | (ninguno) | Tipo de binder: `kafka` o `rabbit`. Determina qué implementación de binder se instancia. |
| `spring.cloud.stream.binders.<name>.environment.*` | `Map<String,Object>` | `{}` | Configuración aislada del binder. Las propiedades aquí no afectan a otros binders del mismo tipo. Permite dos instancias del mismo tipo de binder (dos clusters Kafka distintos). |
| `spring.cloud.stream.bindings.<name>.binder` | `String` | binder por defecto | Nombre del binder al que se asigna este binding. Debe coincidir con una clave en `spring.cloud.stream.binders`. |
| `spring.cloud.stream.binders.<name>.default-candidate` | `boolean` | `true` | Si `true`, este binder es candidato a ser el binder por defecto. Con múltiples binders del mismo tipo, poner `false` en los no-primarios evita ambigüedad. |
| `spring.cloud.stream.binders.<name>.inherit-environment` | `boolean` | `true` | Si `true`, el binder hereda el entorno de la aplicación principal. Poner `false` para aislamiento completo de propiedades. |

## Buenas y malas prácticas

**Hacer:**

- Nombrar los binders con identificadores de dominio (`kafka-primary`, `rabbit-notifications`) en lugar de nombres técnicos (`kafka1`, `rabbit1`). Los nombres de dominio hacen que la asignación de bindings sea legible sin necesidad de consultar la configuración del binder.
- Configurar `default-candidate: false` en los binders secundarios cuando hay dos binders del mismo tipo. Sin esta configuración, Spring Cloud Stream no sabe cuál es el binder por defecto para los bindings que no declaran un `binder` explícito, y lanzará una excepción ambigua en el arranque.
- Usar `inherit-environment: false` para binders que conectan a infraestructura con configuraciones muy diferentes (credenciales, TLS). El aislamiento evita que propiedades de un binder contaminen la configuración del otro.

**Evitar:**

- Incluir ambos starters (`spring-cloud-stream-binder-kafka` y `spring-cloud-stream-binder-rabbit`) sin configurar multi-binder explícitamente. Si ambos están en el classpath sin sección `binders`, Spring Cloud Stream detecta dos binders disponibles y no puede determinar cuál usar para cada binding, fallando en el arranque.
- Crear un multi-binder con dos binders del mismo tipo sin diferenciación en la sección `environment`. Si los dos binders Kafka apuntan al mismo clúster y tienen la misma configuración, no hay razón para tenerlos separados; usar un solo binder simplifica la configuración.

---

← [6.10 StreamBridge — producción de mensajes programática](sc-stream-streambridge.md) | [Índice (README.md)](README.md) | [6.12 Actuator y monitorización de bindings](sc-stream-actuator.md) →
