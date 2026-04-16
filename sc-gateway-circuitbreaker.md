# 3.4.5 Circuit Breaker en Gateway con fallback

← [3.4.4 Rate Limiting con RedisRateLimiter](sc-gateway-ratelimiting.md) | [Índice](README.md) | [3.4.6 Retry: política de reintentos automáticos](sc-gateway-retry.md) →

---

## Introducción

Cuando un servicio backend falla o responde con alta latencia, sin protección el gateway mantiene las conexiones Netty abiertas esperando respuesta, agotando el pool de conexiones y produciendo un fallo en cascada que afecta a todas las rutas. `CircuitBreakerGatewayFilterFactory` resuelve este problema: envuelve la llamada al backend en un circuit breaker Resilience4j y, cuando el circuito está abierto o la llamada falla, redirige la petición a un `fallbackUri` local en lugar de propagar el error al cliente. El desarrollador necesita configurar el Circuit Breaker en Gateway para gestionar fallos y timeouts de servicios backend con respuesta de fallback.

> [PREREQUISITO] Añadir `spring-cloud-starter-circuitbreaker-reactor-resilience4j`. La lógica de estados del circuit breaker (CLOSED, OPEN, HALF_OPEN) y la configuración avanzada de Resilience4j se cubren en [5 Spring Cloud Circuit Breaker](sc-circuitbreaker-estados.md).

## Diagrama: flujo del circuit breaker en Gateway

El siguiente diagrama muestra los tres caminos posibles cuando el filtro CircuitBreaker está activo.

```
Petición → Gateway → CircuitBreakerGatewayFilter
                              │
               ┌──────────────┴──────────────────────┐
               │                                     │
         Circuito CLOSED                      Circuito OPEN
         (estado normal)                      (backend en fallo)
               │                                     │
               ▼                                     ▼
         Llama al backend               No llama al backend
               │                                     │
       ┌───────┴──────────┐                fallbackUri
       │                  │
  Respuesta OK      Fallo / timeout / statusCode en lista
       │                  │
   Responde         ┌─────┴─────────────────────────────┐
   al cliente       │                                   │
                    │ fallbackUri configurado?           │ sin fallback
                    │         │                         │
                    ▼         ▼                         ▼
              fallback     forward:/         503 Service Unavailable
              externo      controlador       (respuesta de error Gateway)
                           local
```

## Ejemplo central

El siguiente ejemplo muestra una configuración completa con fallback local y acceso al error original desde el controlador de fallback.

### Dependencia Maven

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-circuitbreaker-reactor-resilience4j</artifactId>
</dependency>
```

### Configuración YAML de la ruta con circuit breaker

```yaml
# application.yml
spring:
  cloud:
    gateway:
      routes:
        - id: order-service-route
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
          filters:
            - name: CircuitBreaker
              args:
                name: orderServiceCB          # nombre del CircuitBreaker en Resilience4j
                fallbackUri: forward:/fallback/orders
                # statusCodes: códigos HTTP del backend que se consideran fallo
                # (además de excepciones de conexión)
                statusCodes:
                  - 500
                  - 502
                  - 503
                  - 504

        - id: catalog-service-route
          uri: lb://catalog-service
          predicates:
            - Path=/api/catalog/**
          filters:
            - name: CircuitBreaker
              args:
                name: catalogServiceCB
                # fallbackUri con URI externo (fuera del gateway)
                fallbackUri: https://static-catalog.cdn.example.com/catalog-fallback
                statusCodes:
                  - 500
                  - 503

# Configuración de Resilience4j para los circuit breakers usados en Gateway
resilience4j:
  circuitbreaker:
    configs:
      default:
        slidingWindowSize: 10
        failureRateThreshold: 50
        waitDurationInOpenState: 10s
        permittedNumberOfCallsInHalfOpenState: 3
    instances:
      orderServiceCB:
        baseConfig: default
        # TimeLimiter.timeout es la configuración crítica para peticiones lentas:
        # si el backend tarda más de este tiempo, cuenta como fallo
      catalogServiceCB:
        baseConfig: default

  timelimiter:
    configs:
      default:
        timeoutDuration: 3s    # timeout para todos los CB; aplica a orderServiceCB y catalogServiceCB
    instances:
      orderServiceCB:
        timeoutDuration: 5s    # override: órdenes pueden tardar hasta 5s
```

### Controlador de fallback local

```java
package com.example.gateway;

import org.springframework.cloud.gateway.support.ServerWebExchangeUtils;
import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

import java.util.Map;

@RestController
public class FallbackController {

    /**
     * Fallback para el servicio de órdenes.
     * La excepción original está disponible en el atributo del exchange
     * CIRCUIT_BREAKER_EXECUTION_EXCEPTION_ATTR.
     */
    @RequestMapping("/fallback/orders")
    public Mono<Map<String, Object>> ordersFallback(ServerWebExchange exchange) {
        // Obtener la causa del fallo (timeout, excepción de conexión, etc.)
        Throwable cause = exchange.getAttribute(
            ServerWebExchangeUtils.CIRCUIT_BREAKER_EXECUTION_EXCEPTION_ATTR);

        String message = (cause != null)
            ? "Servicio de órdenes no disponible: " + cause.getMessage()
            : "Servicio de órdenes temporalmente no disponible";

        // Establecer el status de la respuesta de fallback
        exchange.getResponse().setStatusCode(HttpStatus.SERVICE_UNAVAILABLE);

        return Mono.just(Map.of(
            "error", "SERVICE_UNAVAILABLE",
            "message", message,
            "service", "order-service",
            "fallback", true
        ));
    }
}
```

### Variante programática con RouteLocatorBuilder

```java
package com.example.gateway;

import org.springframework.cloud.gateway.route.RouteLocator;
import org.springframework.cloud.gateway.route.builder.RouteLocatorBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpStatus;

@Configuration
public class CircuitBreakerRoutesConfig {

    @Bean
    public RouteLocator circuitBreakerRoutes(RouteLocatorBuilder builder) {
        return builder.routes()
            .route("order-cb-route", r -> r
                .path("/api/orders/**")
                .filters(f -> f
                    .circuitBreaker(cb -> cb
                        .setName("orderServiceCB")
                        .setFallbackUri("forward:/fallback/orders")
                        .setStatusCodes(
                            String.valueOf(HttpStatus.INTERNAL_SERVER_ERROR.value()),
                            String.valueOf(HttpStatus.SERVICE_UNAVAILABLE.value()))))
                .uri("lb://order-service"))
            .build();
    }
}
```

## Tabla de elementos clave

La siguiente tabla recoge los parámetros de `CircuitBreakerGatewayFilterFactory` y la configuración de Resilience4j más relevante.

| Parámetro | Ubicación | Descripción |
|-----------|-----------|-------------|
| `name` | filtro `args` | Nombre del CircuitBreaker en Resilience4j; debe coincidir con el instance name |
| `fallbackUri` | filtro `args` | URI de fallback: `forward:/path` (controlador local) o URL absoluta (externo) |
| `statusCodes` | filtro `args` | Lista de códigos HTTP del backend que activan el CB como fallo |
| `resilience4j.circuitbreaker.instances.<name>.failureRateThreshold` | `application.yml` | % de fallos para abrir el circuito (default 50%) |
| `resilience4j.circuitbreaker.instances.<name>.slidingWindowSize` | `application.yml` | Número de llamadas en la ventana deslizante |
| `resilience4j.circuitbreaker.instances.<name>.waitDurationInOpenState` | `application.yml` | Tiempo en estado OPEN antes de probar HALF_OPEN |
| `resilience4j.timelimiter.instances.<name>.timeoutDuration` | `application.yml` | Timeout por llamada; si se supera, cuenta como fallo |
| `ServerWebExchangeUtils.CIRCUIT_BREAKER_EXECUTION_EXCEPTION_ATTR` | exchange attribute | Throwable con la causa del fallo, disponible en el controlador fallback |

> [EXAMEN] Los `statusCodes` en el filtro son adicionales a las excepciones de conexión: incluso sin `statusCodes`, una `ConnectTimeoutException` o `ReadTimeoutException` del cliente Netty activa el circuit breaker. Los `statusCodes` añaden la posibilidad de tratar respuestas HTTP de error del backend (ej: 500, 503) también como fallos del circuito.

## Buenas y malas prácticas

**Hacer:**
- Configurar `TimeLimiter.timeoutDuration` siempre que se use `CircuitBreakerGatewayFilter`: sin él, el circuit breaker nunca se activa por latencia alta, solo por excepciones de conexión. Una petición que tarda 30 segundos no es menos problemática que una que falla.
- Usar `forward:/` como `fallbackUri` en lugar de URIs externos: el controlador local puede acceder al exchange para construir respuestas de fallback con contexto del error original.
- Añadir `statusCodes: [500, 502, 503, 504]` en los circuit breakers de servicios críticos: sin esta configuración, una respuesta 500 del backend no activa el circuito y el gateway sigue enviando tráfico a un servicio que falla.
- Configurar circuit breakers distintos (`name` distinto) por servicio: compartir un circuit breaker entre servicios diferentes mezcla sus contadores de fallo y produce estados de circuit breaker incorrectos.

**Evitar:**
- Devolver 200 OK desde el controlador de fallback con un body de error: los clientes que confían en el status code para detección de fallos no sabrán que recibieron una respuesta degradada.
- Configurar `waitDurationInOpenState` muy corto (ej: 1s): el backend en HALF_OPEN recibirá tráfico de prueba casi inmediatamente, sin tiempo para recuperarse, lo que prolonga el ciclo de fallos.
- Usar la misma instancia de `CircuitBreaker` para el filtro de Gateway y para `@CircuitBreaker` en el mismo servicio: los contadores de Resilience4j se comparten y el comportamiento se vuelve difícil de predecir.

---

← [3.4.4 Rate Limiting con RedisRateLimiter](sc-gateway-ratelimiting.md) | [Índice](README.md) | [3.4.6 Retry: política de reintentos automáticos](sc-gateway-retry.md) →
