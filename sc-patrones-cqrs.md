# 13.1 CQRS con Spring Cloud Stream y Gateway

← [12.10 Testing y verificación de Spring Cloud Function](sc-function-testing.md) | [Índice (README.md)](README.md) | [13.2 Saga de coreografía con Spring Cloud Stream](sc-patrones-saga-coreografia.md) →

---

CQRS (Command Query Responsibility Segregation) separa los modelos de escritura (Commands) y lectura (Queries) de un agregado de dominio. En microservicios Spring Cloud, el patrón se implementa con Spring Cloud Stream para propagar eventos de escritura hacia el modelo de lectura, y Spring Cloud Gateway para enrutar los requests de Command a los servicios de escritura y los de Query a los servicios de lectura optimizados. El beneficio principal es la capacidad de escalar y optimizar independientemente cada lado: el write model usa una base de datos transaccional (PostgreSQL, MySQL), mientras el read model puede usar una base de datos de lectura rápida (Redis, Elasticsearch, MongoDB) proyectada a partir de los eventos.

> [PREREQUISITO] Requiere `spring-cloud-starter-stream-kafka` (o RabbitMQ) para la propagación de eventos entre write y read model. `spring-cloud-starter-gateway` para el routing de Commands y Queries.

## Arquitectura CQRS con Spring Cloud

```
┌─────────────────────────────────────────────────────────────────────┐
│                        API Gateway                                   │
│  POST /orders/*  →  Order Command Service (write model)              │
│  GET  /orders/*  →  Order Query Service  (read model)                │
└─────────────────────────────────────────────────────────────────────┘
         │                              ↑
         │ (Command ejecutado)          │ (Query sobre proyección)
         ▼                              │
┌──────────────────────┐    Kafka       │
│ Order Command Service│ ─────event──→ │ Order Query Service
│ (PostgreSQL)         │               │ (Redis/Elasticsearch)
└──────────────────────┘               └──────────────────────
```

## Ejemplo central: CQRS con Stream y Gateway

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

<!-- Order Command Service -->
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-stream-kafka</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
</dependencies>
```

### Command Service — escribe en PostgreSQL y publica evento

```java
package com.example.cqrs;

import org.springframework.cloud.stream.function.StreamBridge;
import org.springframework.messaging.support.MessageBuilder;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class OrderCommandService {

    private final OrderRepository orderRepository;
    private final StreamBridge streamBridge;

    public OrderCommandService(OrderRepository orderRepository,
                                StreamBridge streamBridge) {
        this.orderRepository = orderRepository;
        this.streamBridge = streamBridge;
    }

    @Transactional
    public Order createOrder(CreateOrderCommand command) {
        // 1. Aplicar el comando en el write model (PostgreSQL)
        Order order = new Order(command.productId(), command.quantity(), command.userId());
        Order saved = orderRepository.save(order);

        // 2. Publicar evento para actualizar el read model
        OrderCreatedEvent event = new OrderCreatedEvent(
            saved.getId(), saved.getProductId(), saved.getQuantity(),
            saved.getUserId(), saved.getCreatedAt()
        );
        streamBridge.send("order-events-out-0", 
            MessageBuilder.withPayload(event)
                .setHeader("eventType", "ORDER_CREATED")
                .build());

        return saved;
    }
}
```

### Query Service — proyección en Redis

```java
package com.example.cqrs;

import org.springframework.context.annotation.Bean;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.function.Consumer;

@Service
public class OrderProjectionService {

    private final RedisTemplate<String, OrderView> redisTemplate;

    public OrderProjectionService(RedisTemplate<String, OrderView> redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    // Suscriptor al evento del Command Service
    @Bean
    public Consumer<OrderCreatedEvent> orderEventsIn() {
        return event -> {
            // Actualizar la proyección de lectura en Redis
            OrderView view = new OrderView(
                event.orderId(), event.productId(),
                event.quantity(), event.userId(), event.createdAt()
            );
            redisTemplate.opsForValue().set("order:" + event.orderId(), view);
            redisTemplate.opsForList().leftPush("orders:user:" + event.userId(), view);
        };
    }

    // Query: lectura desde la proyección Redis
    public OrderView getOrder(String orderId) {
        return redisTemplate.opsForValue().get("order:" + orderId);
    }

    public List<OrderView> getOrdersByUser(String userId) {
        return redisTemplate.opsForList().range("orders:user:" + userId, 0, -1);
    }
}
```

### Gateway routing para Commands y Queries

```yaml
# application.yml del API Gateway
spring:
  cloud:
    gateway:
      routes:
        # Commands → write service (POST, PUT, DELETE)
        - id: order-commands
          uri: lb://order-command-service
          predicates:
            - Path=/orders/**
            - Method=POST,PUT,DELETE,PATCH
          filters:
            - TokenRelay=

        # Queries → read service (GET)
        - id: order-queries
          uri: lb://order-query-service
          predicates:
            - Path=/orders/**
            - Method=GET
          filters:
            - TokenRelay=
```

### application.yml de bindings Stream

```yaml
# Order Command Service
spring:
  cloud:
    stream:
      bindings:
        order-events-out-0:
          destination: order-events
          contentType: application/json

# Order Query Service
spring:
  cloud:
    stream:
      bindings:
        orderEventsIn-in-0:
          destination: order-events
          group: order-query-service-group
          contentType: application/json
```

> [CONCEPTO] La consistencia eventual es una característica inherente de CQRS: entre el momento en que el Command Service escribe la orden y el momento en que el Query Service procesa el evento y actualiza la proyección, hay un retraso (típicamente milisegundos con Kafka en el mismo datacenter). Las APIs de query deben comunicar esto mediante headers como `X-Consistency: eventual` y los clientes deben diseñarse para tolerar ver datos ligeramente desactualizados.

## Tabla de elementos clave

| Componente | Descripción |
|---|---|
| Command Service | Contiene el write model; recibe Commands y persiste en BD transaccional; publica eventos al completar |
| Query Service | Contiene el read model (proyección); suscrito a eventos del Command Service; devuelve vistas optimizadas para lectura |
| `StreamBridge.send(binding, message)` | Publicación imperativa de eventos desde el Command Service sin binding declarativo |
| Predicado `Method=POST,PUT,DELETE` | Gateway routing por método HTTP; separa Commands de Queries en el routing |
| Consistencia eventual | El read model puede estar desfasado respecto al write model; las queries pueden devolver datos anteriores al último command |
| `Consumer<Event>` | Función Spring Cloud Stream que procesa eventos y actualiza la proyección del read model |

## Buenas y malas prácticas

**Hacer:**
- Incluir un identificador de correlación (`correlationId`) en el evento publicado por el Command Service; permite a los clientes implementar estrategias de "read-your-own-writes" consultando el Query Service con el `correlationId` hasta que la proyección esté actualizada.
- Diseñar las proyecciones del Query Service para ser reconstruibles desde cero reproduciendo los eventos; facilita añadir nuevas proyecciones sin migración de datos.
- Escalar el Query Service de forma independiente al Command Service; en la mayoría de sistemas el ratio de reads/writes es 10:1 o mayor.

**Evitar:**
- Mezclar lógica de write y read en el mismo servicio "para simplificar": elimina los beneficios de escalabilidad independiente y hace que las optimizaciones de lectura impacten al rendimiento de escritura y viceversa.
- Sincronizar el Command Service con el Query Service de forma síncrona (esperar a que la proyección esté actualizada antes de responder al cliente); introduce acoplamiento temporal y degrada el rendimiento del write path.
- Usar la base de datos del Command Service como fuente de queries en lugar del read model; anula el propósito de CQRS y crea contención de recursos entre reads y writes.

---

← [12.10 Testing y verificación de Spring Cloud Function](sc-function-testing.md) | [Índice (README.md)](README.md) | [13.2 Saga de coreografía con Spring Cloud Stream](sc-patrones-saga-coreografia.md) →
