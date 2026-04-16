# 3.12 Spring Cloud Gateway MVC

← [3.11 Filtros y predicados personalizados (extension points)](sc-gateway-extension-points.md) | [Índice](README.md) | [3.13 Actuator, administración y observabilidad](sc-gateway-actuator.md) →

---

## Introducción

Spring Cloud Gateway MVC es la variante del gateway basada en el stack servlet de Spring MVC, introducida en Spring Cloud 2023.x como alternativa al gateway reactivo para equipos con proyectos que ya usan Spring MVC y no pueden migrar a WebFlux. La diferencia fundamental no es de funcionalidad, sino de modelo de ejecución: el gateway reactivo corre en un event loop Netty con threading no bloqueante; el gateway MVC corre en un pool de threads servlet, y en Spring Boot 4 con Java 21 puede aprovechar virtual threads. El desarrollador necesita configurar Spring Cloud Gateway MVC para integrar el enrutamiento en un stack servlet cuando el proyecto no puede usar WebFlux.

> [ADVERTENCIA] `spring-cloud-starter-gateway` (reactivo) y `spring-cloud-starter-gateway-mvc` son mutuamente excluyentes: no pueden coexistir en el mismo proyecto. Spring Cloud 2025.1.1 (Oakwood) / Spring Boot 4.0.x pueden haber modificado el API de Gateway MVC respecto a 2024.0.x; verificar las Release Notes antes de publicar.

## Diagrama: comparación de modelos de threading

El siguiente diagrama muestra la diferencia arquitectural entre el gateway reactivo y el MVC en cuanto al modelo de threading.

```
Gateway Reactivo (WebFlux + Reactor Netty)
─────────────────────────────────────────
  N peticiones concurrentes
       │
       ▼
  Reactor Netty event loop
  (típicamente: 2 × CPUs worker threads)
       │
       ├── Request 1 → FilterChain (non-blocking, Mono/Flux) → proxy
       ├── Request 2 → FilterChain (non-blocking) → proxy
       └── Request N → FilterChain (non-blocking) → proxy
  Sin bloquear el thread; cada I/O es asíncrono

Gateway MVC (Servlet + Virtual Threads, Spring Boot 4 / Java 21)
──────────────────────────────────────────────────────────────────
  N peticiones concurrentes
       │
       ▼
  Servlet container (Tomcat / Jetty)
  spring.threads.virtual.enabled=true
       │
       ├── Request 1 → Virtual Thread 1 → FilterChain → RestClient proxy
       ├── Request 2 → Virtual Thread 2 → FilterChain → RestClient proxy
       └── Request N → Virtual Thread N → FilterChain → RestClient proxy
  Virtual threads: ligeros (MB vs KB de stack), bloqueantes OK
```

## Ejemplo central

El siguiente ejemplo muestra la configuración mínima de Spring Cloud Gateway MVC con rutas YAML y la variante programática con `RouterFunction`.

### Dependencia Maven (MVC — excluyente con el gateway reactivo)

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway-mvc</artifactId>
    <!-- NO añadir spring-cloud-starter-gateway (reactivo) en el mismo proyecto -->
</dependency>
<!-- Virtual threads (Java 21 / Spring Boot 4) -->
<!-- No requiere dependencia adicional; se activa con propiedad -->
```

### Configuración YAML (idéntica a la del gateway reactivo para filtros soportados)

```yaml
# application.yml
spring:
  application:
    name: api-gateway-mvc

  # Activar virtual threads (Spring Boot 4 + Java 21)
  threads:
    virtual:
      enabled: true

  cloud:
    gateway:
      mvc:
        routes:
          # La estructura de rutas es la misma que en el gateway reactivo
          - id: order-service-route
            uri: lb://order-service
            predicates:
              - Path=/api/orders/**
            filters:
              - StripPrefix=1
              - AddRequestHeader=X-Source, gateway-mvc

          - id: catalog-service-route
            uri: lb://catalog-service
            predicates:
              - Path=/api/catalog/**
              - Method=GET,HEAD
            filters:
              - StripPrefix=1
              - RewritePath=/api/catalog/(?<segment>.*), /$\{segment}
```

### Variante programática con RouterFunction

```java
package com.example.gatewaymvc;

import org.springframework.cloud.gateway.server.mvc.filter.BeforeFilterFunctions;
import org.springframework.cloud.gateway.server.mvc.handler.GatewayRouterFunctions;
import org.springframework.cloud.gateway.server.mvc.handler.ProxyExchange;
import org.springframework.cloud.gateway.server.mvc.predicate.GatewayRequestPredicates;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.function.RouterFunction;
import org.springframework.web.servlet.function.ServerResponse;

import java.net.URI;

@Configuration
public class GatewayMvcRoutesConfig {

    /**
     * Configuración programática de rutas en Spring Cloud Gateway MVC.
     * Usa RouterFunction de Spring MVC 6+, no RouteLocatorBuilder de WebFlux.
     */
    @Bean
    public RouterFunction<ServerResponse> orderServiceRoutes() {
        return GatewayRouterFunctions.route("order-service-route")
            // Predicado equivalente a Path=/api/orders/**
            .GET("/api/orders/**", GatewayRequestPredicates.path("/api/orders/**"),
                request -> ProxyExchange.builder(request)
                    .uri(URI.create("lb://order-service"))
                    .build()
                    .exchangeStrippingPrefix(1))
            .build();
    }

    /**
     * Ruta con filtros adicionales.
     * BeforeFilterFunctions provee los equivalentes MVC de los GatewayFilterFactory.
     */
    @Bean
    public RouterFunction<ServerResponse> catalogServiceRoutes() {
        return GatewayRouterFunctions.route("catalog-service-route")
            .route(GatewayRequestPredicates.path("/api/catalog/**"),
                request -> {
                    // Aplicar filtros antes del proxy
                    request = BeforeFilterFunctions.addRequestHeader(
                        "X-Source", "gateway-mvc").apply(request);
                    return ProxyExchange.builder(request)
                        .uri(URI.create("lb://catalog-service"))
                        .build()
                        .exchange();
                })
            .build();
    }
}
```

### Clase principal

```java
package com.example.gatewaymvc;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class GatewayMvcApplication {
    public static void main(String[] args) {
        SpringApplication.run(GatewayMvcApplication.class, args);
    }
}
```

## Tabla de elementos clave

La siguiente tabla compara qué GatewayFilters y características están disponibles en MVC vs reactivo.

| Característica | Gateway Reactivo | Gateway MVC |
|----------------|:----------------:|:-----------:|
| `AddRequestHeader` / `SetRequestHeader` / `RemoveRequestHeader` | Sí | Sí |
| `StripPrefix` / `PrefixPath` / `RewritePath` | Sí | Sí |
| `AddResponseHeader` / `RemoveResponseHeader` | Sí | Sí |
| `RequestRateLimiter` (RedisRateLimiter) | Sí | Limitado (verificar versión) |
| `CircuitBreaker` | Sí | Sí |
| `Retry` | Sí | Sí |
| `TokenRelay` | Sí (OAuth2 reactivo) | Sí (OAuth2 servlet) |
| `ModifyRequestBody` / `ModifyResponseBody` | Sí (RewriteFunction reactiva) | API diferente |
| `SecureHeaders` | Sí | Sí |
| WebSocket proxying | Sí (nativo) | No |
| SSE proxying | Sí (nativo Flux) | Limitado |
| Virtual threads | No (event loop) | Sí (Java 21+) |
| Modelo de programación | `RouteLocatorBuilder` / YAML | `RouterFunction` / YAML |

> [EXAMEN] La pregunta de entrevista habitual es: "¿Cuándo usarías Gateway MVC en lugar del reactivo?" La respuesta correcta no es "cuando no sabes WebFlux", sino "cuando el proyecto ya tiene stack servlet y no puede/no quiere migrar a WebFlux; o cuando se necesita integración directa con filtros servlet existentes (ej: Spring Security antiguo); o con Java 21 y virtual threads donde el modelo bloqueante simple puede ser suficiente para el throughput requerido".

## Buenas y malas prácticas

**Hacer:**
- Activar `spring.threads.virtual.enabled=true` cuando se usa Gateway MVC con Java 21: los virtual threads eliminan el coste de los threads bloqueantes sin necesidad de programación reactiva, acercando el throughput al del gateway reactivo para workloads I/O intensivos.
- Elegir Gateway MVC cuando el proyecto tiene filtros servlet existentes que necesitan ejecutarse en el mismo pipeline: la integración con la cadena de filtros servlet de Spring MVC es directa, sin necesidad de adaptadores.

**Evitar:**
- Usar Gateway MVC para proxying de WebSockets: el stack servlet no soporta WebSocket proxy de forma nativa; para WebSocket es obligatorio el gateway reactivo.
- Mezclar `spring-cloud-starter-gateway` y `spring-cloud-starter-gateway-mvc` en el mismo módulo Maven/Gradle: el classpath tendrá ambos stacks y la autoconfiguración fallará con conflictos de beans en el arranque.

## Comparación: cuándo elegir cada opción

La siguiente tabla resume los criterios de decisión entre el gateway reactivo y el MVC.

| Criterio | Reactivo (WebFlux) | MVC (Servlet) |
|----------|:------------------:|:-------------:|
| Proyecto nuevo sin restricciones | Recomendado | Alternativa |
| Stack existente Spring MVC | Migración necesaria | Sin migración |
| WebSocket proxying | Sí | No |
| SSE nativo | Sí | Limitado |
| Virtual threads (Java 21) | No aplica (event loop) | Sí |
| Throughput máximo (I/O bound) | Alto (event loop) | Alto (virtual threads) |
| Throughput con CPU bound | Medio | Medio |
| Curva de aprendizaje | Reactor/WebFlux | Spring MVC estándar |

---

← [3.11 Filtros y predicados personalizados (extension points)](sc-gateway-extension-points.md) | [Índice](README.md) | [3.13 Actuator, administración y observabilidad](sc-gateway-actuator.md) →
