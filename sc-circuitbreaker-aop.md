# 5.10 Anotaciones AOP de Resilience4j y orden de decoradores combinados

← [5.9 TimeLimiter: acotación de tiempos de ejecución asíncronos](sc-circuitbreaker-timelimiter.md) | [Índice](README.md) | [5.11 Métricas de Resilience4j con Micrometer y endpoints Actuator](sc-circuitbreaker-metricas.md) →

## Introducción

Resilience4j ofrece un conjunto de anotaciones AOP que permiten decorar métodos Spring con comportamiento de resiliencia sin escribir código de envolvimiento explícito. La comodidad de las anotaciones tiene un precio: cuando se aplican varias anotaciones al mismo método, el orden en que se aplican los aspectos determina el comportamiento del sistema, y ese orden no es intuitivo.

El orden canónico documentado por Resilience4j es `Bulkhead > CircuitBreaker > RateLimiter > TimeLimiter > Retry`. Esto significa que Bulkhead es el decorador más exterior (se evalúa primero) y Retry es el más interior (se evalúa al final, dentro de todos los demás). Una consecuencia práctica: si Retry está dentro del Circuit Breaker, cada reintento cuenta como una llamada para el CB, lo que puede hacer que el circuito abra más rápido de lo esperado. Si el orden se invierte (Retry fuera del CB), los reintentos no cuentan para el CB pero el CB puede rechazar los reintentos si ya está OPEN.

La limitación de las anotaciones AOP en llamadas internas (self-invocation) también es un punto crítico: cuando un método de una clase llama a otro método de la misma clase, Spring AOP no intercepta la llamada interna porque el proxy no está involucrado.

> [ADVERTENCIA] La limitación de self-invocation es inherente al mecanismo de proxy de Spring. No hay forma de sortearla con anotaciones; si se necesita que la llamada interna esté decorada, usar la API programática (`CircuitBreakerFactory.create()`) o inyectar el propio bean (`@Autowired Self self`).

## Representación visual

El orden de aplicación de los aspectos define una cadena de decoradores en forma de cebolla:

```
Llamada al método decorado desde el exterior:
──────────────────────────────────────────────────────────────

  ┌─── @Bulkhead (más exterior, se evalúa primero) ──────────┐
  │  ┌─── @CircuitBreaker ──────────────────────────────────┐ │
  │  │  ┌─── @RateLimiter ───────────────────────────────┐  │ │
  │  │  │  ┌─── @TimeLimiter ────────────────────────┐   │  │ │
  │  │  │  │  ┌─── @Retry (más interior) ─────────┐  │   │  │ │
  │  │  │  │  │                                   │  │   │  │ │
  │  │  │  │  │    MÉTODO REAL                    │  │   │  │ │
  │  │  │  │  └───────────────────────────────────┘  │   │  │ │
  │  │  │  └────────────────────────────────────────┘   │  │ │
  │  │  └──────────────────────────────────────────────┘  │ │
  │  └─────────────────────────────────────────────────────┘ │
  └──────────────────────────────────────────────────────────┘

Consecuencia del orden:
  1. Bulkhead comprueba si hay recursos disponibles
  2. CB comprueba si el circuito está cerrado
  3. RateLimiter comprueba si hay permisos disponibles
  4. TimeLimiter inicia el temporizador
  5. Retry ejecuta el método; si falla, repite desde el paso 4
     (cada reintento es observable por CB, RL y TL en el camino de vuelta)
```

| Anotación           | Aspect Order por defecto | Comportamiento cuando falla                         |
|---------------------|--------------------------|-----------------------------------------------------|
| `@Bulkhead`         | 2147483647 - 1 (más alto) | Lanza `BulkheadFullException`                      |
| `@CircuitBreaker`   | 2147483647 - 2           | Lanza `CallNotPermittedException` si OPEN           |
| `@RateLimiter`      | 2147483647 - 3           | Lanza `RequestNotPermitted`                         |
| `@TimeLimiter`      | 2147483647 - 4           | Lanza `TimeoutException`                            |
| `@Retry`            | 2147483647 - 5 (más bajo) | Reintenta; lanza excepción original tras agotar intentos |

## Ejemplo central

Servicio de procesamiento de pedidos que combina las cinco anotaciones en el mismo método, con configuración explícita del orden de aspectos y tratamiento de la limitación de self-invocation.

```xml
<!-- pom.xml -->
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-aop</artifactId>
    </dependency>
</dependencies>
```

```yaml
# application.yml — configuración de instancias usadas en las anotaciones
resilience4j:
  circuitbreaker:
    instances:
      orderProcessor:
        sliding-window-size: 10
        minimum-number-of-calls: 5
        failure-rate-threshold: 50
        wait-duration-in-open-state: 30s
        automatic-transition-from-open-to-half-open-enabled: true
  retry:
    instances:
      orderProcessor:
        max-attempts: 3
        wait-duration: 500ms
        enable-exponential-backoff: true
        exponential-backoff-multiplier: 2.0
        retry-exceptions:
          - java.io.IOException
          - org.springframework.web.client.HttpServerErrorException
  bulkhead:
    instances:
      orderProcessor:
        max-concurrent-calls: 10
        max-wait-duration: 100ms
  ratelimiter:
    instances:
      orderProcessor:
        limit-for-period: 20
        limit-refresh-period: 1s
        timeout-duration: 200ms
  timelimiter:
    instances:
      orderProcessor:
        timeout-duration: 5s
        cancel-running-future: true

  # Personalización del orden de aspectos si se necesita invertir algún par
  # (aquí se deja en el orden canónico por defecto)
  bulkhead:
    bulkhead-aspect-order: 2147482647
  circuitbreaker:
    circuit-breaker-aspect-order: 2147482646
  ratelimiter:
    rate-limiter-aspect-order: 2147482645
  timelimiter:
    time-limiter-aspect-order: 2147482644
  retry:
    retry-aspect-order: 2147482643
```

```java
// OrderProcessingService.java — uso combinado de las 5 anotaciones
package com.example.orders.service;

import com.example.orders.model.Order;
import com.example.orders.model.OrderResult;
import io.github.resilience4j.bulkhead.annotation.Bulkhead;
import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import io.github.resilience4j.ratelimiter.annotation.RateLimiter;
import io.github.resilience4j.retry.annotation.Retry;
import io.github.resilience4j.timelimiter.annotation.TimeLimiter;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestClient;
import java.util.concurrent.CompletableFuture;

@Service
public class OrderProcessingService {

    private final RestClient restClient;
    // Inyección del propio bean para resolver la limitación de self-invocation
    private final OrderProcessingService self;

    public OrderProcessingService(
            RestClient.Builder builder,
            OrderProcessingService self) {
        this.restClient = builder.baseUrl("http://order-service").build();
        this.self = self;
    }

    /**
     * Método público decorado con todas las anotaciones.
     * El orden real de aplicación es el canónico:
     * Bulkhead > CB > RateLimiter > TimeLimiter > Retry
     */
    @Bulkhead(name = "orderProcessor", fallbackMethod = "fallbackOrder")
    @CircuitBreaker(name = "orderProcessor", fallbackMethod = "fallbackOrder")
    @RateLimiter(name = "orderProcessor", fallbackMethod = "fallbackOrder")
    @TimeLimiter(name = "orderProcessor", fallbackMethod = "fallbackOrder")
    @Retry(name = "orderProcessor", fallbackMethod = "fallbackOrder")
    public CompletableFuture<OrderResult> processOrder(Order order) {
        return CompletableFuture.supplyAsync(() ->
                restClient.post()
                        .uri("/api/orders")
                        .body(order)
                        .retrieve()
                        .body(OrderResult.class)
        );
    }

    /**
     * Ejemplo de cómo llamar a un método decorado desde dentro de la misma clase:
     * usar this.processOrder() NO pasa por el proxy (sin AOP).
     * Usar self.processOrder() SÍ pasa por el proxy (con AOP).
     */
    public CompletableFuture<OrderResult> processOrderWithPreCheck(Order order) {
        if (order.items() == null || order.items().isEmpty()) {
            return CompletableFuture.failedFuture(
                    new IllegalArgumentException("El pedido no tiene items"));
        }
        // self.processOrder pasa por el proxy y aplica todas las anotaciones
        return self.processOrder(order);
    }

    private CompletableFuture<OrderResult> fallbackOrder(Order order, Throwable ex) {
        String reason = switch (ex.getClass().getSimpleName()) {
            case "BulkheadFullException"      -> "BULKHEAD_FULL";
            case "CallNotPermittedException"  -> "CIRCUIT_OPEN";
            case "RequestNotPermitted"        -> "RATE_LIMIT_EXCEEDED";
            case "TimeoutException"           -> "TIMEOUT";
            default                           -> "ERROR: " + ex.getMessage();
        };
        return CompletableFuture.completedFuture(
                new OrderResult(order.orderId(), "FAILED", reason));
    }
}
```

```java
// Modelos
package com.example.orders.model;

import java.util.List;

public record Order(String orderId, String customerId, List<String> items) {}
public record OrderResult(String orderId, String status, String reason) {}
```

## Tabla de elementos clave

Aspectos de configuración de las anotaciones AOP de Resilience4j.

| Anotación                                  | Atributos principales                      | Tipo de retorno requerido                     |
|--------------------------------------------|--------------------------------------------|-----------------------------------------------|
| `@CircuitBreaker(name, fallbackMethod)`    | `name`, `fallbackMethod`                   | Cualquiera                                    |
| `@Retry(name, fallbackMethod)`             | `name`, `fallbackMethod`                   | Cualquiera                                    |
| `@Bulkhead(name, type, fallbackMethod)`    | `name`, `type` (SEMAPHORE/THREADPOOL), `fallbackMethod` | SEMAPHORE: cualquiera; THREADPOOL: `CompletableFuture` |
| `@RateLimiter(name, fallbackMethod)`       | `name`, `fallbackMethod`                   | Cualquiera                                    |
| `@TimeLimiter(name, fallbackMethod)`       | `name`, `fallbackMethod`                   | `CompletableFuture` (obligatorio)             |
| Configuración de aspect order YAML         | `resilience4j.[component].aspect-order`   | Integer                                       |
| Self-invocation workaround                 | Inyección del bean en el constructor       | —                                             |

## Buenas y malas prácticas

**Hacer:**

- Anotar solo el método más externo de la llamada a servicios externos. Decorar métodos intermedios que a su vez llaman a los externos duplica los aspectos y produce comportamientos imprevisibles.
- Documentar en un comentario el orden de aplicación real cuando se combinan varias anotaciones. El orden canónico (`Bulkhead > CB > RL > TL > Retry`) no es obvio para quien lee el código por primera vez.
- Usar la inyección del propio bean en el constructor para resolver self-invocation de forma limpia. Declarar el bean en el constructor con `@Autowired` o con inyección de constructor evita dependencias circulares con Spring.
- Revisar que todos los métodos `fallbackMethod` tengan exactamente la misma firma que el método protegido más el parámetro `Throwable` al final. Resilience4j lanza `FallbackExecutionException` en runtime si las firmas no coinciden.

**Evitar:**

- No combinar `@Retry` con operaciones no idempotentes (pagos, creación de recursos) solo porque la anotación es conveniente. Las anotaciones ocultan el riesgo de duplicación; la API programática hace el contrato más explícito.
- Evitar el orden invertido Retry > CircuitBreaker (Retry exterior al CB) sin documentarlo. En ese orden, los reintentos pasan antes de que el CB evalúe el fallo, lo que puede generar llamadas adicionales a un servicio ya caído.
- No usar `@TimeLimiter` en métodos síncronos. La anotación solo funciona con `CompletableFuture`; en métodos síncronos no produce ningún error en tiempo de compilación pero tampoco aplica ningún timeout en tiempo de ejecución, lo que crea una falsa sensación de seguridad.
- Evitar tener más de dos o tres anotaciones por método. A partir de tres es una señal de que el método tiene demasiadas responsabilidades o de que la arquitectura del servicio necesita revisión.

---

← [5.9 TimeLimiter: acotación de tiempos de ejecución asíncronos](sc-circuitbreaker-timelimiter.md) | [Índice](README.md) | [5.11 Métricas de Resilience4j con Micrometer y endpoints Actuator](sc-circuitbreaker-metricas.md) →
