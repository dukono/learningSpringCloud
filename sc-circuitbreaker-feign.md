# 5.13 Integración del Circuit Breaker con OpenFeign, RestClient y WebClient

← [5.12 Health check y eventos CircuitBreakerEvent en producción](sc-circuitbreaker-eventos.md) | [Índice](README.md) | [5.14 Testing del Circuit Breaker, Retry y Bulkhead](sc-circuitbreaker-testing.md) →

## Introducción

La integración del Circuit Breaker con los clientes HTTP de Spring Cloud es el escenario más común en producción: los microservicios se comunican entre sí vía HTTP y cada llamada necesita protección ante fallos del servicio remoto. Esta integración puede ser automática (en el caso de OpenFeign) o manual (en el caso de RestClient y WebClient), pero en ambos casos el Circuit Breaker es el mismo Resilience4j subyacente.

La integración con OpenFeign es la más transparente: basta con habilitar una propiedad y declarar un bean `FallbackFactory<T>` para que cada método del cliente Feign quede automáticamente protegido por un Circuit Breaker individual. La integración con `RestClient` y `WebClient` es explícita y requiere envolver cada llamada con `circuitBreaker.run()`, lo que ofrece más control sobre el fallback y la gestión de excepciones.

> [PREREQUISITO] Este fichero asume conocimiento de la API de Spring Cloud OpenFeign (Módulo 4) para la parte de integración Feign. Las propiedades `spring.cloud.openfeign.*` se describen en el módulo 4; aquí solo se documenta la parte de integración con Circuit Breaker.

## Representación visual

Los dos modos de integración del Circuit Breaker con clientes HTTP:

```
Integración con OpenFeign (automática):
─────────────────────────────────────────────────────────────────
  @FeignClient(name = "payment-service", fallback = PaymentFallback.class)
  interface PaymentClient { ... }

  Resilience4j genera automáticamente un CB por método:
  PaymentClient#processPayment(String,Long)   → CB: "payment-service#processPayment(String,Long)"
  PaymentClient#getPaymentStatus(String)      → CB: "payment-service#getPaymentStatus(String)"

  Propiedad activadora:
  spring.cloud.openfeign.circuitbreaker.enabled=true

Integración con RestClient (manual):
─────────────────────────────────────────────────────────────────
  CircuitBreaker cb = factory.create("paymentService");
  Result result = cb.run(
      () -> restClient.post().uri(...).body(req).retrieve().body(Result.class),
      ex -> fallback(ex)
  );

Integración con WebClient + ReactiveCircuitBreaker:
─────────────────────────────────────────────────────────────────
  Mono<Result> result = reactiveCircuitBreaker.create("paymentService")
      .run(
          webClient.get().uri(...).retrieve().bodyToMono(Result.class),
          t -> Mono.just(fallback)
      );
```

| Característica                          | OpenFeign + CB                             | RestClient + CB (manual)               | WebClient + ReactiveCircuitBreaker       |
|-----------------------------------------|--------------------------------------------|----------------------------------------|------------------------------------------|
| Activación                              | `feign.circuitbreaker.enabled=true`        | Explícita (`cb.run(...)`)              | Explícita (`reactiveCircuitBreaker.run`) |
| Nombre del CB                           | Generado automáticamente                   | Definido por el desarrollador          | Definido por el desarrollador            |
| Modelo de programación                  | Declarativo                                | Imperativo blocking                    | Reactivo non-blocking                    |
| Fallback                                | `FallbackFactory<T>` o clase fallback      | Lambda en `run(supplier, fallback)`    | Lambda en `run(mono, fallback)`          |
| Visibilidad de la excepción en fallback | Con `FallbackFactory`                      | Siempre disponible                     | Siempre disponible                       |

## Ejemplo central

Microservicio de pedidos que usa los tres tipos de integración: Feign para el servicio de inventario, RestClient para el servicio de precios y WebClient reactivo para el servicio de notificaciones.

```xml
<!-- pom.xml -->
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
    </dependency>
    <!-- Starter reactivo para WebClient -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-circuitbreaker-reactor-resilience4j</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-openfeign</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-webflux</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-aop</artifactId>
    </dependency>
</dependencies>
```

```yaml
# application.yml
spring:
  application:
    name: order-service
  cloud:
    openfeign:
      circuitbreaker:
        enabled: true
        # Usar IDs alfanuméricos simplificados (sin parámetros en el nombre)
        alphanumeric-ids:
          enabled: false
        # Agrupar todos los métodos de un Feign client en el mismo CB
        group:
          enabled: false

resilience4j:
  circuitbreaker:
    instances:
      # CB para métodos de InventoryClient (nombre generado por Feign)
      inventory-service#getStock(String):
        sliding-window-size: 10
        failure-rate-threshold: 50
        wait-duration-in-open-state: 30s
        minimum-number-of-calls: 5
      # CB para RestClient (nombre definido manualmente)
      priceService:
        sliding-window-size: 10
        failure-rate-threshold: 50
        wait-duration-in-open-state: 20s
        minimum-number-of-calls: 5
      # CB para WebClient reactivo
      notificationService:
        sliding-window-size: 10
        failure-rate-threshold: 60
        wait-duration-in-open-state: 15s
        minimum-number-of-calls: 3
```

```java
// InventoryClient.java — @FeignClient con Circuit Breaker automático
package com.example.orders.client;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;

@FeignClient(
        name = "inventory-service",
        fallbackFactory = InventoryClientFallbackFactory.class
)
public interface InventoryClient {

    @GetMapping("/api/stock/{productId}")
    StockInfo getStock(@PathVariable String productId);
}
```

```java
// InventoryClientFallbackFactory.java — acceso a la excepción que causó el fallback
package com.example.orders.client;

import org.springframework.cloud.openfeign.FallbackFactory;
import org.springframework.stereotype.Component;

@Component
public class InventoryClientFallbackFactory implements FallbackFactory<InventoryClient> {

    @Override
    public InventoryClient create(Throwable cause) {
        // El FallbackFactory recibe la excepción; el Fallback simple no
        return productId -> {
            if (cause instanceof io.github.resilience4j.circuitbreaker.CallNotPermittedException) {
                return new StockInfo(productId, 0, false, true);  // modo degradado
            }
            return new StockInfo(productId, -1, false, false);    // error genuino
        };
    }
}
```

```java
// StockInfo y otros modelos
package com.example.orders.client;

public record StockInfo(String productId, int quantity, boolean available, boolean degraded) {}
public record PriceInfo(String productId, double price, String currency) {}
public record NotificationResult(String status, String recipient) {}
```

```java
// PriceService.java — RestClient con CB manual
package com.example.orders.service;

import com.example.orders.client.PriceInfo;
import org.springframework.cloud.client.circuitbreaker.CircuitBreaker;
import org.springframework.cloud.client.circuitbreaker.CircuitBreakerFactory;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestClient;

@Service
public class PriceService {

    private final CircuitBreaker circuitBreaker;
    private final RestClient restClient;

    public PriceService(
            CircuitBreakerFactory<?, ?> circuitBreakerFactory,
            RestClient.Builder builder) {
        // Reutilizar la instancia del CB (no crear en cada llamada)
        this.circuitBreaker = circuitBreakerFactory.create("priceService");
        this.restClient = builder.baseUrl("http://price-service").build();
    }

    public PriceInfo getPrice(String productId) {
        return circuitBreaker.run(
                () -> restClient.get()
                        .uri("/api/prices/{id}", productId)
                        .retrieve()
                        .body(PriceInfo.class),
                ex -> new PriceInfo(productId, 0.0, "EUR")   // precio por defecto
        );
    }
}
```

```java
// NotificationService.java — WebClient con ReactiveCircuitBreaker
package com.example.orders.service;

import com.example.orders.client.NotificationResult;
import org.springframework.cloud.client.circuitbreaker.ReactiveCircuitBreaker;
import org.springframework.cloud.client.circuitbreaker.ReactiveCircuitBreakerFactory;
import org.springframework.stereotype.Service;
import org.springframework.web.reactive.function.client.WebClient;
import reactor.core.publisher.Mono;

@Service
public class NotificationService {

    private final ReactiveCircuitBreaker reactiveCircuitBreaker;
    private final WebClient webClient;

    public NotificationService(
            ReactiveCircuitBreakerFactory<?, ?> reactiveCircuitBreakerFactory,
            WebClient.Builder builder) {
        this.reactiveCircuitBreaker = reactiveCircuitBreakerFactory.create("notificationService");
        this.webClient = builder.baseUrl("http://notification-service").build();
    }

    public Mono<NotificationResult> sendOrderConfirmation(String orderId, String userId) {
        return reactiveCircuitBreaker.run(
                webClient.post()
                        .uri("/api/notifications/order-confirmation")
                        .bodyValue(new OrderNotificationRequest(orderId, userId))
                        .retrieve()
                        .bodyToMono(NotificationResult.class),
                throwable -> Mono.just(new NotificationResult("SKIPPED", userId))
        );
    }
}
```

```java
// Request model para notificación
package com.example.orders.service;

public record OrderNotificationRequest(String orderId, String userId) {}
```

## Tabla de elementos clave

Propiedades de la integración OpenFeign con Circuit Breaker.

| Propiedad                                                      | Tipo    | Default  | Descripción                                                                       |
|----------------------------------------------------------------|---------|----------|-----------------------------------------------------------------------------------|
| `spring.cloud.openfeign.circuitbreaker.enabled`                | boolean | `false`  | Activa la integración automática CB en todos los clientes Feign                   |
| `spring.cloud.openfeign.circuitbreaker.alphanumeric-ids.enabled` | boolean | `false` | Usa IDs simplificados sin tipos de parámetro en el nombre del CB                 |
| `spring.cloud.openfeign.circuitbreaker.group.enabled`          | boolean | `false`  | Un solo CB para todos los métodos del cliente (en lugar de uno por método)        |
| Nombre automático del CB (sin group)                           | String  | —        | `[serviceName]#[methodName]([paramTypes])` p.ej. `inventory-service#getStock(String)` |
| Nombre automático del CB (con alphanumeric-ids)                | String  | —        | `[className]_[methodName]` sin tipos de parámetro                                |
| `FallbackFactory<T>`                                           | interfaz | —       | Factory que recibe el `Throwable` y crea el objeto fallback                       |
| `Fallback simple` (clase que implementa la interfaz)           | clase   | —        | Sin acceso a la excepción; adecuado cuando no se necesita el motivo del fallo     |

## Buenas y malas prácticas

**Hacer:**

- Preferir `FallbackFactory<T>` sobre la clase fallback simple cuando el comportamiento del fallback debe diferir según si fue `CallNotPermittedException` (circuito abierto) o una excepción de red/HTTP. La distinción ayuda a loguear con el nivel correcto.
- Almacenar la instancia del `CircuitBreaker` en un campo de la clase cuando se usa la API programática con `RestClient`. Llamar a `factory.create("name")` en cada invocación tiene coste de lookup aunque el registry haga cache.
- Verificar los nombres de los CB generados automáticamente por Feign en `/actuator/circuitbreakers` antes de definir la configuración YAML. El formato `[className]#[methodName]([paramTypes])` puede causar sorpresas con tipos genéricos.
- Usar `spring.cloud.openfeign.circuitbreaker.group.enabled: true` solo cuando todos los métodos del cliente tienen el mismo SLA. Si unos métodos son críticos y otros no, el CB por método permite afinar los umbrales independientemente.

**Evitar:**

- No activar `spring.cloud.openfeign.circuitbreaker.enabled: true` globalmente sin haber definido configuración YAML para las instancias. Los CB se crearán con los defaults de Resilience4j (`minimumNumberOfCalls: 100`) que en muchos servicios nunca llegan a evaluar.
- Evitar declarar el fallback de Feign como clase `@Component` sin la anotación correcta. Si Spring no encuentra el bean, el fallback se ignora silenciosamente y el cliente Feign no tiene protección.
- No mezclar `@CircuitBreaker` (anotación AOP) con la integración automática de Feign en el mismo cliente. La anotación AOP decoraría el método del servicio que llama al cliente Feign, añadiendo un segundo CB que puede interactuar con el CB interno de Feign de formas no previstas.

---

← [5.12 Health check y eventos CircuitBreakerEvent en producción](sc-circuitbreaker-eventos.md) | [Índice](README.md) | [5.14 Testing del Circuit Breaker, Retry y Bulkhead](sc-circuitbreaker-testing.md) →
