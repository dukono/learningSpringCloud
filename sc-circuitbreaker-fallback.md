# 5.5 Fallback: estrategias de respuesta degradada

← [5.4 Configuración YAML avanzada del Circuit Breaker](sc-circuitbreaker-config.md) | [Índice](README.md) | [5.6 Retry: política de reintentos, backoff y anotación @Retry](sc-circuitbreaker-retry.md) →

## Introducción

El Circuit Breaker protege el sistema cortando el flujo de llamadas hacia un servicio degradado, pero el corte en sí no es suficiente: la aplicación necesita responder de forma coherente al cliente final aunque el servicio remoto no esté disponible. Aquí entra el **fallback**: la lógica de respuesta alternativa que se ejecuta cuando el circuito está abierto, cuando se agota el número de reintentos o cuando el bulkhead rechaza una llamada.

Un fallback bien diseñado es lo que diferencia una aplicación que degrada graciosamente de una que simplemente falla con HTTP 500. Las estrategias van desde devolver un valor por defecto inmediato, hasta consultar una caché local, hasta delegar en un sistema de backup. La elección depende de las garantías de consistencia que el negocio puede aceptar en modo degradado.

> [ADVERTENCIA] Un fallback que también llama a servicios externos no está aislado del fallo: si el servicio de caché remoto también falla, el fallback amplifica el problema en lugar de contenerlo. Los fallbacks deben ser locales o basados en datos ya disponibles en memoria.

## Representación visual

Las dos formas de registrar un fallback en Resilience4j son la API programática y las anotaciones AOP. Ambas comparten la misma semántica pero difieren en dónde se declara la lógica.

```
API programática (CircuitBreaker.run):
──────────────────────────────────────
  circuitBreaker.run(
      () -> remoteService.call(),       ← supplier (lógica principal)
      throwable -> fallbackValue()      ← fallback (lógica de degradación)
  )

  Throwable puede ser:
  ├── CallNotPermittedException  → circuito OPEN o HALF_OPEN saturado
  ├── BulkheadFullException      → bulkhead lleno
  ├── RequestNotPermitted        → rate limiter agotado
  └── cualquier excepción del supplier

Anotación AOP (@CircuitBreaker, @Retry, @Bulkhead, @RateLimiter):
──────────────────────────────────────────────────────────────────
  @CircuitBreaker(name = "svc", fallbackMethod = "myFallback")
  public Response callRemote(String id) { ... }

  private Response myFallback(String id, Throwable ex) {
      // misma firma que el método protegido + Throwable al final
      return Response.empty();
  }
```

| Estrategia de fallback       | Cuándo usarla                                              | Consistencia   |
|------------------------------|------------------------------------------------------------|----------------|
| Valor por defecto            | Respuesta vacía/nula aceptable; ejemplo: lista vacía       | No garantizada |
| Caché local (in-memory)      | Datos de lectura con tolerancia a versión anterior         | Eventual       |
| Caché distribuida (Redis)    | Datos críticos con caché invalidable externamente          | Configurable   |
| Respuesta estática           | Contenido degradado fijo (banner, mensaje de error)        | Garantizada    |
| Cola de reintento asíncrono  | Operaciones de escritura que no pueden perderse            | Eventual       |
| Excepción controlada         | El cliente debe conocer explícitamente el modo degradado   | N/A            |

## Ejemplo central

Un servicio de catálogo que protege tres llamadas remotas con fallbacks distintos según la criticidad: precios (caché local), reseñas (lista vacía) y stock (excepción controlada).

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
# application.yml
resilience4j:
  circuitbreaker:
    instances:
      priceService:
        sliding-window-size: 10
        failure-rate-threshold: 50
        wait-duration-in-open-state: 30s
        minimum-number-of-calls: 5
        automatic-transition-from-open-to-half-open-enabled: true
      reviewService:
        sliding-window-size: 20
        failure-rate-threshold: 60
        wait-duration-in-open-state: 15s
        minimum-number-of-calls: 5
      stockService:
        sliding-window-size: 10
        failure-rate-threshold: 40
        wait-duration-in-open-state: 60s
        minimum-number-of-calls: 3
```

```java
// ProductCatalogService.java — tres estrategias de fallback distintas
package com.example.catalog.service;

import com.example.catalog.exception.StockServiceUnavailableException;
import com.example.catalog.model.PriceInfo;
import com.example.catalog.model.Review;
import com.example.catalog.model.StockInfo;
import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import org.springframework.core.ParameterizedTypeReference;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestClient;

import java.math.BigDecimal;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

@Service
public class ProductCatalogService {

    private final RestClient restClient;

    // Caché local in-memory para precios (TTL gestionado externamente o con Caffeine)
    private final Map<String, PriceInfo> priceCache = new ConcurrentHashMap<>();

    public ProductCatalogService(RestClient.Builder builder) {
        this.restClient = builder.build();
    }

    // ─── Estrategia 1: Caché local ──────────────────────────────────────────────

    @CircuitBreaker(name = "priceService", fallbackMethod = "fallbackPrice")
    public PriceInfo getPrice(String productId) {
        PriceInfo price = restClient.get()
                .uri("http://price-service/api/prices/{id}", productId)
                .retrieve()
                .body(PriceInfo.class);
        // Actualizar caché cuando el servicio funciona
        if (price != null) {
            priceCache.put(productId, price);
        }
        return price;
    }

    // Fallback: devuelve el precio de la caché local o un precio 0 si no hay caché
    private PriceInfo fallbackPrice(String productId, Throwable ex) {
        return priceCache.getOrDefault(productId,
                new PriceInfo(productId, BigDecimal.ZERO, "EUR", true));
    }

    // ─── Estrategia 2: Colección vacía ──────────────────────────────────────────

    @CircuitBreaker(name = "reviewService", fallbackMethod = "fallbackReviews")
    public List<Review> getReviews(String productId) {
        return restClient.get()
                .uri("http://review-service/api/reviews/product/{id}", productId)
                .retrieve()
                .body(new ParameterizedTypeReference<List<Review>>() {});
    }

    // Fallback: sin reseñas disponibles — experiencia degradada pero no bloqueante
    private List<Review> fallbackReviews(String productId, Throwable ex) {
        return List.of();
    }

    // ─── Estrategia 3: Excepción controlada ─────────────────────────────────────

    @CircuitBreaker(name = "stockService", fallbackMethod = "fallbackStock")
    public StockInfo getStock(String productId) {
        return restClient.get()
                .uri("http://stock-service/api/stock/{id}", productId)
                .retrieve()
                .body(StockInfo.class);
    }

    // Fallback: lanza excepción controlada; el controller la traduce a 503
    private StockInfo fallbackStock(String productId, Throwable ex) {
        throw new StockServiceUnavailableException(
                "El servicio de stock no está disponible temporalmente. " +
                "Causa: " + ex.getClass().getSimpleName());
    }
}
```

```java
// Modelos
package com.example.catalog.model;

import java.math.BigDecimal;

public record PriceInfo(String productId, BigDecimal amount, String currency, boolean degraded) {}
public record Review(String reviewId, String userId, int rating, String text) {}
public record StockInfo(String productId, int quantity, boolean available) {}
```

```java
// Excepción controlada para fallback de stock
package com.example.catalog.exception;

public class StockServiceUnavailableException extends RuntimeException {
    public StockServiceUnavailableException(String message) {
        super(message);
    }
}
```

```java
// GlobalExceptionHandler.java — traduce la excepción del fallback a HTTP 503
package com.example.catalog.web;

import com.example.catalog.exception.StockServiceUnavailableException;
import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.ResponseStatus;
import org.springframework.web.bind.annotation.RestControllerAdvice;
import java.util.Map;

@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(StockServiceUnavailableException.class)
    @ResponseStatus(HttpStatus.SERVICE_UNAVAILABLE)
    public Map<String, String> handleStockUnavailable(StockServiceUnavailableException ex) {
        return Map.of(
                "error", "SERVICE_UNAVAILABLE",
                "message", ex.getMessage()
        );
    }
}
```

## Tabla de elementos clave

Los aspectos más importantes del mecanismo de fallback en Resilience4j Spring Boot.

| Concepto / Atributo                          | Tipo / Valores                            | Descripción                                                                              |
|----------------------------------------------|-------------------------------------------|------------------------------------------------------------------------------------------|
| `fallbackMethod` (en anotaciones)            | nombre de método `String`                 | Nombre del método fallback en la misma clase                                             |
| Firma del método fallback                    | `T fallback(P1,..., Throwable)`           | Misma firma que el método protegido con `Throwable` opcional al final                    |
| Fallback sin parámetro `Throwable`           | `T fallback(P1,...)`                      | Válido; no recibe la excepción pero aplica para cualquier excepción                      |
| Múltiples fallbacks por tipo de excepción    | varios métodos con mismo nombre           | Resilience4j selecciona el más específico según el tipo de excepción                     |
| `CallNotPermittedException`                  | subtipo de `RuntimeException`             | Lanzada cuando el circuito está OPEN; llega al fallback                                  |
| Fallback en API programática                 | `Function<Throwable, T>`                  | Segundo argumento de `circuitBreaker.run(supplier, fallback)`                            |
| Fallback reactivo (Mono/Flux)                | el método fallback retorna `Mono<T>`      | Compatible con `@CircuitBreaker` en métodos que retornan Mono o Flux                     |
| Fallback encadenado                          | un fallback llama a otro CB               | Posible pero peligroso si el segundo servicio también está caído                         |

## Buenas y malas prácticas

**Hacer:**

- Tipar el parámetro `Throwable` del fallback como `CallNotPermittedException` cuando se quiere distinguir entre "circuito abierto" y "excepción del servicio". Resilience4j seleccionará el método fallback más específico en la jerarquía de excepciones.
- Usar caché local (Caffeine, `ConcurrentHashMap`) para fallbacks de lectura con datos que cambian lentamente. La caché debe actualizarse en el camino feliz (cuando el servicio funciona) para que el fallback tenga datos frescos.
- Hacer los fallbacks lo más simples posibles: retornar un valor inmediato, leer de memoria, o lanzar una excepción controlada. Cualquier lógica compleja en el fallback puede fallar también y enmascarar el error original.
- Incluir el tipo de excepción en la respuesta de fallback (logs o métricas) para distinguir entre "circuito abierto" y "excepción genuina" durante el análisis postmortem.

**Evitar:**

- No diseñar fallbacks que llamen a servicios externos no decorados con Circuit Breaker. Si el fallback llama a un servicio de caché distribuida que también falla, se pierde la protección del patrón.
- Evitar retornar `null` desde el fallback cuando el código cliente asume un objeto no nulo. Un `NullPointerException` en el cliente tras el fallback es peor que una excepción controlada.
- No declarar el método fallback como `public` innecesariamente. Puede ser `private` o `protected`; Spring AOP lo interceptará igualmente y reducir la visibilidad evita su uso inadvertido desde otro código.
- Evitar lógica de negocio compleja en fallbacks que requiera transacciones o acceso a base de datos propia. Si la base de datos local también está bajo carga, el fallback puede convertirse en un cuello de botella secundario.

---

← [5.4 Configuración YAML avanzada del Circuit Breaker](sc-circuitbreaker-config.md) | [Índice](README.md) | [5.6 Retry: política de reintentos, backoff y anotación @Retry](sc-circuitbreaker-retry.md) →
