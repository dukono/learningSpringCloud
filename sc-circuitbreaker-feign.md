# 4.10 Integración con Feign

← [4.9 Eventos y Métricas — Observabilidad completa](sc-circuitbreaker-eventos.md) | [Índice](README.md) | [4.11 Integración con Spring Cloud Gateway](sc-circuitbreaker-gateway.md) →

---

## Introducción

La integración entre Feign y Resilience4j permite que cada método de un cliente Feign esté protegido automáticamente por un Circuit Breaker sin necesidad de anotaciones adicionales. Cuando el servicio downstream falla, el CircuitBreaker se activa y redirige al fallback definido en el cliente Feign, que puede devolver datos cacheados, valores por defecto o simplemente registrar el fallo. Esta integración es la forma más natural de añadir resiliencia a las llamadas HTTP declarativas en una arquitectura Spring Cloud.

> [PREREQUISITO] Requiere conocimiento del cliente Feign básico documentado en [3.1 Cliente declarativo](sc-feign-cliente-declarativo.md). La integración con Resilience4j solo funciona cuando `feign.circuitbreaker.enabled=true`.

## Nomenclatura automática del CircuitBreaker

Cuando se activa la integración Feign-Resilience4j, Feign crea automáticamente un CircuitBreaker por cada método del cliente. El nombre del CircuitBreaker sigue el patrón:

```
<clientName>#<methodName>(<parameterTypes>)
```

Por ejemplo, para el cliente `ProductClient` con método `getProduct(Long)`, el nombre será `ProductClient#getProduct(Long)`. Este nombre se usa para configurar el CircuitBreaker en YAML:

```yaml
resilience4j:
  circuitbreaker:
    instances:
      ProductClient#getProduct(Long):
        failure-rate-threshold: 60
        wait-duration-in-open-state: 30s
```

> [CONCEPTO] El nombre por defecto puede cambiarse implementando `CircuitBreakerNameResolver` como bean Spring. Esto es útil para usar nombres cortos y legibles en configuración YAML.

## Ejemplo central

El ejemplo muestra la configuración completa: cliente Feign con fallback simple, FallbackFactory para acceder a la excepción, y configuración de CircuitBreakerNameResolver:

```java
package com.example.client;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;

// fallback: clase que implementa el interface para respuestas degradadas simples
// fallbackFactory: para acceder a la excepción original del fallo
@FeignClient(
    name = "product-service",
    fallback = ProductClientFallback.class,
    fallbackFactory = ProductClientFallbackFactory.class  // solo uno puede estar activo
)
public interface ProductClient {
    @GetMapping("/products/{id}")
    ProductResponse getProduct(@PathVariable Long id);

    @PostMapping("/products")
    ProductResponse createProduct(@RequestBody ProductRequest request);
}
```

Implementación de fallback simple (sin acceso a la excepción):

```java
package com.example.client;

import org.springframework.stereotype.Component;

// Debe ser un bean Spring para que Feign lo inyecte
@Component
public class ProductClientFallback implements ProductClient {

    @Override
    public ProductResponse getProduct(Long id) {
        // Respuesta degradada — sin acceso a la excepción original
        return ProductResponse.notAvailable(id);
    }

    @Override
    public ProductResponse createProduct(ProductRequest request) {
        return ProductResponse.notAvailable(-1L);
    }
}
```

FallbackFactory para acceder a la excepción original (más útil en producción):

```java
package com.example.client;

import org.springframework.cloud.openfeign.FallbackFactory;
import org.springframework.stereotype.Component;

// FallbackFactory permite acceder a la causa del fallo en cada método
@Component
public class ProductClientFallbackFactory implements FallbackFactory<ProductClient> {

    @Override
    public ProductClient create(Throwable cause) {
        // cause es la excepción que disparó el fallback (puede ser FeignException,
        // CallNotPermittedException si el CB está abierto, etc.)
        return new ProductClient() {
            @Override
            public ProductResponse getProduct(Long id) {
                if (cause instanceof feign.FeignException.ServiceUnavailable) {
                    return ProductResponse.serviceDown(id, "503 from product-service");
                }
                return ProductResponse.notAvailable(id, cause.getMessage());
            }

            @Override
            public ProductResponse createProduct(ProductRequest request) {
                return ProductResponse.notAvailable(-1L, cause.getMessage());
            }
        };
    }
}
```

Configuración YAML para activar la integración y configurar el CB:

```yaml
spring:
  cloud:
    openfeign:
      circuitbreaker:
        enabled: true          # activa la integración Feign + Resilience4j

resilience4j:
  circuitbreaker:
    configs:
      default:
        sliding-window-size: 10
        minimum-number-of-calls: 5
        failure-rate-threshold: 50
        wait-duration-in-open-state: 30s
    instances:
      # Nombre largo automático basado en clientName#methodName(paramTypes)
      product-service#getProduct(Long):
        failure-rate-threshold: 60
        wait-duration-in-open-state: 60s
```

Implementación de `CircuitBreakerNameResolver` para nombres cortos:

```java
package com.example.config;

import feign.Target;
import org.springframework.cloud.openfeign.CircuitBreakerNameResolver;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import java.lang.reflect.Method;

@Configuration
public class FeignCircuitBreakerConfig {

    @Bean
    public CircuitBreakerNameResolver circuitBreakerNameResolver() {
        // Nombre simplificado: solo clientName + methodName
        return (feignClientName, target, method) ->
            feignClientName + "_" + method.getName();
    }
}
```

## Tabla de elementos clave

| Elemento | Descripción |
|----------|-------------|
| `feign.circuitbreaker.enabled=true` | Activa la integración global Feign-Resilience4j |
| `@FeignClient(fallback=X.class)` | Clase que implementa el interface para respuestas degradadas |
| `@FeignClient(fallbackFactory=X.class)` | Factory que recibe la excepción y devuelve una implementación fallback |
| Nombre automático | `<clientName>#<methodName>(<paramTypes>)` |
| `CircuitBreakerNameResolver` | Bean para personalizar el nombre del CB por método Feign |

> [EXAMEN] `fallback` y `fallbackFactory` son mutuamente excluyentes en la misma anotación `@FeignClient`. Si se definen ambos, se usa `fallbackFactory`. Para acceder a la causa del fallo, siempre usar `fallbackFactory`.

> [ADVERTENCIA] Cuando el CircuitBreaker está en estado OPEN, la excepción que llega al `FallbackFactory` es `CallNotPermittedException`, no la excepción original del downstream. Es importante distinguir esta excepción en el fallback para dar respuestas diferentes según la causa del fallo.

## Propagación de excepciones

El flujo de propagación de excepciones desde Feign hacia el CircuitBreaker:

1. El método Feign lanza `FeignException` (para errores HTTP) o una excepción de red.
2. Si la excepción está en `record-exceptions` (o no está en `ignore-exceptions`), el CircuitBreaker la registra como fallo.
3. Si el umbral se supera, el CB transiciona a OPEN.
4. En estado OPEN, se lanza `CallNotPermittedException` directamente (sin llamar al downstream).
5. La excepción (sea `FeignException` o `CallNotPermittedException`) llega al `FallbackFactory.create()`.

## Buenas y malas prácticas

**Buenas prácticas:**
- Usar `FallbackFactory` en lugar de `fallback` simple en producción para poder loggear la excepción original.
- Configurar `ignore-exceptions: [FeignException.BadRequest]` para que errores 4xx no abran el CB (son errores del cliente, no del servidor).
- Dar nombres legibles a los CBs mediante `CircuitBreakerNameResolver` para facilitar la configuración YAML.

**Malas prácticas:**
- Lanzar excepciones dentro del fallback: produce `FallbackExecutionException` que oculta el error original.
- No diferenciar en el fallback entre `CallNotPermittedException` (CB abierto) y `FeignException` (fallo real): impide métricas de calidad.

## Verificación y práctica

> [EXAMEN] 1. ¿Qué propiedad activa la integración entre Feign y Resilience4j?

> [EXAMEN] 2. ¿Cuál es el nombre automático del CircuitBreaker para `@FeignClient(name="order-service")` con método `getOrder(Long, String)`?

> [EXAMEN] 3. ¿Qué diferencia hay entre `fallback` y `fallbackFactory` en `@FeignClient`? ¿Cuál permite acceder a la excepción?

> [EXAMEN] 4. Cuando el CircuitBreaker de un cliente Feign está en estado OPEN, ¿qué excepción recibe el `FallbackFactory.create(cause)`?

> [EXAMEN] 5. ¿Cómo se configura en YAML un CircuitBreaker con nombre corto "productService" cuando se usa `CircuitBreakerNameResolver` que devuelve `clientName + "_" + methodName`?

---

← [4.9 Eventos y Métricas — Observabilidad completa](sc-circuitbreaker-eventos.md) | [Índice](README.md) | [4.11 Integración con Spring Cloud Gateway](sc-circuitbreaker-gateway.md) →
