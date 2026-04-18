# 5.6 GlobalFilter: interfaz, orden y filtros built-in

← [5.5 GatewayFilter Factories de resiliencia](sc-gateway-filter-factories-resiliencia.md) | [Índice](README.md) | [5.7 Integración con Service Discovery](sc-gateway-discovery-integration.md) →

---

## Introducción

Los `GlobalFilter` son filtros que se aplican automáticamente a **todas** las rutas del Gateway, sin necesidad de configurarlos en cada ruta individualmente. Representan el mecanismo para implementar comportamiento transversal: logging de acceso, propagación de headers de trazabilidad, métricas globales o autenticación a nivel de gateway. Spring Cloud Gateway incluye varios GlobalFilters built-in que realizan las funciones esenciales del proxy (resolución de instancias, envío HTTP, WebSocket). Entender su orden de ejecución es clave tanto para debugging como para implementar filtros custom sin interferir con el flujo del gateway.

> [CONCEPTO] Un `GlobalFilter` implementa las interfaces `GlobalFilter` y `Ordered`. La diferencia con `GatewayFilter` es de **scope**: un `GatewayFilter` aplica solo a la ruta donde está configurado; un `GlobalFilter` aplica a **todas** las rutas del gateway.

## Interfaz GlobalFilter y control de orden

La interfaz `GlobalFilter` define un único método reactivo. El orden de ejecución dentro de la cadena de filtros se controla implementando la interfaz `Ordered` o usando la anotación `@Order`.

```java
public interface GlobalFilter {
    Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain);
}
```

Los filtros con **número de orden menor** se ejecutan **antes** (más cerca del cliente en la fase PRE, más lejos en la fase POST). Los GlobalFilters built-in usan órdenes negativos en la fase de -10000 a 0 para ejecutarse después de los filtros de usuario que típicamente usan valores positivos o `Ordered.LOWEST_PRECEDENCE`.

> [EXAMEN] En Spring Cloud Gateway, un GlobalFilter con `getOrder()` retornando `-1` se ejecuta **antes** que uno con `getOrder()` retornando `0`. En la fase POST (código después de `chain.filter(exchange)`), el orden se invierte: el último en ejecutar PRE es el primero en ejecutar POST.

## GlobalFilters built-in y sus órdenes

Los siguientes GlobalFilters están incluidos en Spring Cloud Gateway y forman la columna vertebral del ciclo de vida de cada petición:

```
Orden de ejecución (menor número = primero en PRE):

Integer.MIN_VALUE  ← GatewayMetricsFilter (métricas Micrometer)
      │
    -400           ← RouteToRequestUrlFilter (resuelve URI final de la ruta)
      │
    -200           ← ReactiveLoadBalancerClientFilter (resuelve lb:// a instancia)
      │
    -100           ← ForwardPathFilter (ajusta path para forward://)
      │
      0            ← [Filtros custom de usuario — orden por defecto]
      │
   2147483546      ← NettyWriteResponseFilter (escribe respuesta Netty al cliente)
      │
   2147483547      ← WebsocketRoutingFilter (routing WebSocket)
      │
   2147483548      ← ForwardRoutingFilter (routing forward://)
      │
   2147483549      ← NettyRoutingFilter (routing HTTP/HTTPS via Netty — el proxy real)
```

Cada GlobalFilter built-in tiene una responsabilidad específica en el flujo:

`RouteToRequestUrlFilter` (orden -400) transforma la URI de la ruta (p. ej. `lb://order-service`) en una URL concreta, añadiendo el atributo `GATEWAY_REQUEST_URL_ATTR` al exchange.

`ReactiveLoadBalancerClientFilter` (orden -200) resuelve el esquema `lb://service-id` a una instancia real del servicio via Spring Cloud LoadBalancer, actualizando la URL en el exchange.

`NettyRoutingFilter` (orden `Integer.MAX_VALUE - 1`) realiza la llamada HTTP real al upstream usando el cliente Netty. Es el "proxy" efectivo.

`GatewayMetricsFilter` registra métricas de cada petición en Micrometer cuando `spring.cloud.gateway.metrics.enabled=true`.

## Ejemplo central

El siguiente ejemplo implementa dos GlobalFilters custom: uno para logging de acceso y otro para propagar un header de correlación en todas las peticiones:

```java
package com.example.gateway.filter;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.core.Ordered;
import org.springframework.http.server.reactive.ServerHttpRequest;
import org.springframework.http.server.reactive.ServerHttpResponse;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

import java.time.Instant;
import java.util.UUID;

/**
 * GlobalFilter de logging: registra método, path, status y duración de cada petición.
 * Orden 1 → ejecuta justo después de los built-in de resolución de ruta.
 */
@Component
public class AccessLogGlobalFilter implements GlobalFilter, Ordered {

    private static final Logger log = LoggerFactory.getLogger(AccessLogGlobalFilter.class);

    @Override
    public int getOrder() {
        return 1; // Después de RouteToRequestUrlFilter (-400) y ReactiveLoadBalancerClientFilter (-200)
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        long startTime = Instant.now().toEpochMilli();

        // Fase PRE: log de entrada
        log.info("[ACCESS] {} {} - requestId={}",
            request.getMethod(),
            request.getURI().getPath(),
            request.getId());

        return chain.filter(exchange)
            // Fase POST: log de salida (se ejecuta cuando se completa la respuesta)
            .doFinally(signalType -> {
                ServerHttpResponse response = exchange.getResponse();
                long duration = Instant.now().toEpochMilli() - startTime;
                log.info("[ACCESS] {} {} → {} ({}ms) - requestId={}",
                    request.getMethod(),
                    request.getURI().getPath(),
                    response.getStatusCode(),
                    duration,
                    request.getId());
            });
    }
}
```

```java
package com.example.gateway.filter;

import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.core.Ordered;
import org.springframework.http.server.reactive.ServerHttpRequest;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

import java.util.UUID;

/**
 * GlobalFilter que añade X-Correlation-Id a TODAS las peticiones.
 * Si ya viene en el request del cliente, lo mantiene.
 * Orden -1 → antes que los filtros de usuario (orden 0+) pero después de los built-in de resolución.
 */
@Component
public class CorrelationIdGlobalFilter implements GlobalFilter, Ordered {

    private static final String CORRELATION_ID_HEADER = "X-Correlation-Id";

    @Override
    public int getOrder() {
        return -1;
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();

        // Si ya existe el header, mantenerlo; si no, generar uno nuevo
        String correlationId = request.getHeaders().getFirst(CORRELATION_ID_HEADER);
        if (correlationId == null || correlationId.isBlank()) {
            correlationId = UUID.randomUUID().toString();
        }

        final String finalCorrelationId = correlationId;

        // Mutamos el request para añadir el header hacia el upstream
        ServerHttpRequest mutatedRequest = request.mutate()
            .header(CORRELATION_ID_HEADER, finalCorrelationId)
            .build();

        // Mutamos el exchange con el request modificado
        ServerWebExchange mutatedExchange = exchange.mutate()
            .request(mutatedRequest)
            .build();

        return chain.filter(mutatedExchange)
            // También añadimos el correlation ID en la respuesta al cliente
            .doFirst(() -> exchange.getResponse()
                .getHeaders()
                .add(CORRELATION_ID_HEADER, finalCorrelationId));
    }
}
```

## Tabla de GlobalFilters built-in

La siguiente tabla resume los GlobalFilters incluidos en Spring Cloud Gateway con su orden y función:

| GlobalFilter | Orden | Función |
|---|---|---|
| `GatewayMetricsFilter` | `Integer.MIN_VALUE` | Registra métricas Micrometer por ruta |
| `RouteToRequestUrlFilter` | -400 | Resuelve la URI de la ruta al exchange |
| `ReactiveLoadBalancerClientFilter` | -200 | Resuelve `lb://` a instancia real |
| `ForwardPathFilter` | -100 | Ajusta path para `forward://` |
| `NettyWriteResponseFilter` | `MAX_VALUE - 1` | Escribe cuerpo de respuesta a Netty |
| `WebsocketRoutingFilter` | `MAX_VALUE - 2` | Routing de conexiones WebSocket |
| `ForwardRoutingFilter` | `MAX_VALUE - 3` | Routing de `forward://` interno |
| `NettyRoutingFilter` | `MAX_VALUE - 4` | Proxy HTTP real via Netty |

## GlobalFilter vs GatewayFilter: comparación

La diferencia fundamental entre `GlobalFilter` y `GatewayFilter` es el **scope de aplicación**. Un `GlobalFilter` se aplica a todas las rutas sin configuración explícita; un `GatewayFilter` solo aplica a las rutas donde se referencia en la sección `filters:`.

Un segundo diferencia es la **forma de registro**: `GlobalFilter` se registra como bean Spring (`@Component` o `@Bean`) y el Gateway lo detecta automáticamente. `GatewayFilter` se obtiene de una `GatewayFilterFactory` que se registra como bean, pero la instancia del filtro se crea por ruta según la configuración YAML/DSL.

> [ADVERTENCIA] Si implementas un `GlobalFilter` que modifica el request (p. ej. añade headers), debes usar `exchange.mutate().request(...)` para crear una copia inmutable modificada. Intentar modificar directamente los headers del `ServerHttpRequest` lanza `UnsupportedOperationException` porque son inmutables.

## Buenas y malas prácticas

**Buenas prácticas:**
- Dar órdenes explícitos y documentados a los GlobalFilters custom para evitar sorpresas con los built-in.
- Usar `.doFinally()` en lugar de `.then()` para logging POST: `doFinally` se ejecuta siempre (éxito, error o cancelación).
- Implementar correlation ID como GlobalFilter para garantizar trazabilidad en todas las rutas sin configuración por ruta.
- Preferir `@Component` para GlobalFilters simples y `@Bean` en `@Configuration` cuando necesitas inyectar dependencias complejas.

**Malas prácticas:**
- Colocar lógica bloqueante en un `GlobalFilter` (consultas síncronas, `Thread.sleep`): bloquea el EventLoop de Netty para todas las peticiones.
- Usar el mismo orden que un GlobalFilter built-in: puede causar comportamiento indefinido según el orden de registro de beans.
- Olvidar llamar a `chain.filter(exchange)`: si no se llama, la cadena se rompe y el upstream nunca recibe la petición.

## Verificación y práctica

1. ¿Cuál es la diferencia entre `GlobalFilter` y `GatewayFilter`? ¿Cuándo usarías cada uno?

2. ¿Cómo controlas el orden de ejecución de un `GlobalFilter` custom? ¿Qué orden debería tener un filter de logging para ejecutarse antes de los filtros de rutas?

3. ¿Qué hace `ReactiveLoadBalancerClientFilter` y en qué orden se ejecuta respecto a `NettyRoutingFilter`?

4. En la fase POST de un GlobalFilter (código después de `chain.filter(exchange)`), ¿en qué orden se ejecutan los filtros si tenemos uno con `getOrder()=1` y otro con `getOrder()=2`?

5. ¿Por qué debes usar `exchange.mutate()` para modificar los headers de una petición en un GlobalFilter? ¿Qué ocurre si intentas modificar directamente `exchange.getRequest().getHeaders()`?

---

← [5.5 GatewayFilter Factories de resiliencia](sc-gateway-filter-factories-resiliencia.md) | [Índice](README.md) | [5.7 Integración con Service Discovery](sc-gateway-discovery-integration.md) →
