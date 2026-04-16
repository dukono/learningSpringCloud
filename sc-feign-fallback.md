# 4.9 Fallback y Circuit Breaker — fallback estático, FallbackFactory y feign.circuitbreaker.enabled

<- [4.8 Manejo de errores HTTP](sc-feign-errores.md) | [Índice](README.md) | [4.10 Retryer](sc-feign-retryer.md) ->

---

## Introducción

Un cliente Feign sin fallback propaga cualquier excepción directamente al llamador. Si el servicio remoto está caído o supera el timeout, el error llega hasta el controlador HTTP del consumidor y se convierte en un 500 para el usuario final. En arquitecturas de microservicios, este comportamiento hace que la disponibilidad de un servicio dependa de la disponibilidad de todos sus dependencias: un fallo en cadena que puede derribar el sistema completo.

El fallback de Feign permite declarar una respuesta degradada para cuando el cliente no puede completar la llamada. Feign delegará en la implementación de fallback en lugar de propagar la excepción, devolviendo datos por defecto, caché o un mensaje de error controlado. El fallback no suaviza el problema; lo hace visible y manejable.

El fallback de Feign requiere que el Circuit Breaker esté activo (`feign.circuitbreaker.enabled=true`). Sin Circuit Breaker, el atributo `fallback` en `@FeignClient` se ignora silenciosamente. La configuración de umbrales del Circuit Breaker (número de fallos, tiempo de espera en estado OPEN) se realiza en el módulo Resilience4j; este fichero solo documenta la configuración del lado Feign.

> [ADVERTENCIA] `feign.circuitbreaker.enabled=true` activa la integración con Spring Cloud Circuit Breaker, que delega en Resilience4j si `spring-cloud-starter-circuitbreaker-resilience4j` está en el classpath. Sin esa dependencia, el Circuit Breaker de Feign no tiene implementación y la activación no produce efecto.

---

## Diagrama: flujo de llamada con fallback activo

El siguiente diagrama muestra en qué punto del pipeline de Feign se activa el fallback.

```
 Llamada al método de la interfaz @FeignClient
         │
         ▼
 Spring Cloud Circuit Breaker (Resilience4j)
         │
         ├─ Estado CLOSED → permite la llamada
         │       │
         │       ▼
         │   Feign proxy → HTTP → Servicio remoto
         │       │
         │       ├─ Éxito → devuelve resultado al llamador
         │       └─ Fallo (excepción / timeout)
         │               │
         │               ▼
         │           ¿fallback declarado?
         │             Sí → FallbackFactory.create(causa).método()
         │             No → excepción se propaga al llamador
         │
         ├─ Estado OPEN → cortocircuito inmediato
         │               │
         │               ▼
         │           FallbackFactory.create(excepción).método()
         │
         └─ Estado HALF-OPEN → permite llamadas de prueba
```

---

## Ejemplo central

El siguiente ejemplo muestra los dos mecanismos de fallback: el fallback estático (clase que implementa la interfaz) y el `FallbackFactory` (que recibe la causa del fallo).

**Interfaz @FeignClient con ambos tipos de fallback declarados (solo uno puede estar activo):**

```java
package com.example.orderservice.client;

import com.example.orderservice.dto.ProductDto;
import com.example.orderservice.feign.CatalogClientFallback;
import com.example.orderservice.feign.CatalogClientFallbackFactory;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;

import java.util.Collections;
import java.util.List;

// Opción A: fallback estático
@FeignClient(
        name     = "catalog-service",
        contextId = "catalogWithFallback",
        fallback = CatalogClientFallback.class
)
public interface CatalogClient {
    @GetMapping("/api/products/{productId}")
    ProductDto getProduct(@PathVariable("productId") String productId);

    @GetMapping("/api/products")
    List<ProductDto> listProducts();
}
```

**CatalogClientFallback.java — fallback estático:**

```java
package com.example.orderservice.feign;

import com.example.orderservice.client.CatalogClient;
import com.example.orderservice.dto.ProductDto;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;

import java.util.Collections;
import java.util.List;

/**
 * Fallback estático: no tiene acceso a la causa del fallo.
 * Útil cuando la respuesta degradada es siempre la misma
 * independientemente del tipo de error.
 *
 * @Component es necesario para que Spring lo registre como bean
 * y Feign pueda inyectarlo en el proxy.
 */
@Component
public class CatalogClientFallback implements CatalogClient {

    private static final Logger log = LoggerFactory.getLogger(CatalogClientFallback.class);

    @Override
    public ProductDto getProduct(String productId) {
        log.warn("Fallback activado para getProduct({}): catalog-service no disponible", productId);
        // Devolver un DTO vacío o de caché en lugar de propagar la excepción
        return ProductDto.unavailable(productId);
    }

    @Override
    public List<ProductDto> listProducts() {
        log.warn("Fallback activado para listProducts(): devolviendo lista vacía");
        return Collections.emptyList();
    }
}
```

**CatalogClientFallbackFactory.java — FallbackFactory con acceso a la causa:**

```java
package com.example.orderservice.feign;

import com.example.orderservice.client.CatalogClient;
import com.example.orderservice.dto.ProductDto;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.cloud.openfeign.FallbackFactory;
import org.springframework.stereotype.Component;

import java.util.Collections;
import java.util.List;

/**
 * FallbackFactory: recibe la excepción que causó el fallo.
 * Permite diferenciar el comportamiento según el tipo de error:
 * - Si es un 404, el producto genuinamente no existe.
 * - Si es un 503, el servicio está caído.
 * - Si es un timeout, puede valer con datos de caché.
 *
 * Usar FallbackFactory en lugar de fallback estático cuando
 * la respuesta degradada depende de la causa del fallo.
 */
@Component
public class CatalogClientFallbackFactory implements FallbackFactory<CatalogClient> {

    private static final Logger log = LoggerFactory.getLogger(CatalogClientFallbackFactory.class);

    @Override
    public CatalogClient create(Throwable cause) {
        // Loggear la causa aquí; el fallback estático no tiene acceso a ella
        log.error("Fallback activado para CatalogClient. Causa: {}", cause.getMessage());

        return new CatalogClient() {
            @Override
            public ProductDto getProduct(String productId) {
                if (cause instanceof feign.FeignException.NotFound) {
                    // 404 real: el producto no existe; no es un error del servicio
                    return null;  // o lanzar ProductNotFoundException aquí
                }
                // Para cualquier otro error: devolver DTO degradado
                return ProductDto.unavailable(productId);
            }

            @Override
            public List<ProductDto> listProducts() {
                return Collections.emptyList();
            }
        };
    }
}
```

**Interfaz usando FallbackFactory en lugar de fallback estático:**

```java
@FeignClient(
        name            = "catalog-service",
        contextId       = "catalogWithFactory",
        fallbackFactory = CatalogClientFallbackFactory.class  // usa factory en lugar de fallback
)
public interface CatalogClientWithFactory {
    @GetMapping("/api/products/{productId}")
    ProductDto getProduct(@PathVariable("productId") String productId);

    @GetMapping("/api/products")
    List<ProductDto> listProducts();
}
```

**application.yml — activación del Circuit Breaker para Feign:**

```yaml
feign:
  circuitbreaker:
    enabled: true   # activa la integración con Spring Cloud Circuit Breaker

# La configuración de umbrales del Circuit Breaker se gestiona en Resilience4j:
# Ver módulo sc-circuitbreaker para configuración avanzada de estados y transiciones.
resilience4j:
  circuitbreaker:
    instances:
      catalog-service:          # nombre del circuito = name del @FeignClient
        sliding-window-size: 10
        failure-rate-threshold: 50
        wait-duration-in-open-state: 10s
```

> [CONCEPTO] Cuando `FallbackFactory` y `fallback` se declaran en el mismo `@FeignClient`, Spring Cloud usa `fallbackFactory` y el `fallback` se ignora. Los dos atributos son mutuamente excluyentes; si se declaran ambos, no hay error en arranque pero el comportamiento puede ser confuso.

---

## Tabla de elementos clave

| Elemento | Tipo | Descripción |
|---|---|---|
| `feign.circuitbreaker.enabled` | `boolean` | `true` para activar CB en todos los clientes Feign |
| `fallback` | `Class` en `@FeignClient` | Clase que implementa la interfaz; sin acceso a la causa |
| `fallbackFactory` | `Class` en `@FeignClient` | `FallbackFactory<T>`; recibe `Throwable` causa del fallo |
| `FallbackFactory<T>` | interfaz | Fábrica que crea una instancia de fallback por cada fallo |
| `@Component` en fallback | — | Obligatorio para que Spring registre el bean de fallback |
| Nombre del circuito | `String` | Por defecto: `name` del `@FeignClient`; configurable en Resilience4j |

---

## Buenas y malas prácticas

**Hacer:**
- Usar `FallbackFactory` en lugar de fallback estático cuando el comportamiento degradado depende del tipo de error. Un 404 semántico ("el producto no existe") debe tratarse diferente a un 503 ("el servicio no está disponible"). El fallback estático no puede distinguirlos.
- Loggear la causa en `FallbackFactory.create(cause)`, no dentro de los métodos del fallback. El `create` se invoca una vez por fallo; los métodos del fallback pueden invocarse múltiples veces si el llamador reintenta a través de Spring Retry. Loggear en `create` garantiza un único mensaje por evento de fallo.
- Indicar en los logs del fallback el método afectado y la causa. En producción, el fallback activo es una señal de degradación; los logs son la única evidencia de qué servicio está fallando y con qué error.

**Evitar:**
- Absorber todas las excepciones en el fallback sin registrarlas. La práctica de devolver silenciosamente un objeto vacío sin log enmascara fallos reales: el sistema parece funcionar pero está devolviendo datos incorrectos a los usuarios.
- Olvidar añadir `@Component` a la clase de fallback o `FallbackFactory`. Feign busca el bean en el `ApplicationContext`; si no está registrado, lanza `BeanCreationException` en arranque con un mensaje críptico que no menciona el fallback como causa.
- Configurar `feign.circuitbreaker.enabled=true` sin añadir la dependencia de Resilience4j al classpath. El comportamiento en ese caso es que el Circuit Breaker no tiene implementación y los fallos no se contabilizan: el fallback nunca se activa por apertura del circuito, solo por errores individuales.

---

<- [4.8 Manejo de errores HTTP](sc-feign-errores.md) | [Índice](README.md) | [4.10 Retryer](sc-feign-retryer.md) ->
