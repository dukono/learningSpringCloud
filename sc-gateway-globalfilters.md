# 3.5 GlobalFilters: filtros transversales y personalización

← [3.4.7 Filtros de seguridad y sesión](sc-gateway-filtros-seguridad.md) | [Índice](README.md) | [3.6 Integración con Service Discovery y Load Balancer](sc-gateway-discovery.md) →

---

## Introducción

Los `GlobalFilter` son los cimientos operativos de Spring Cloud Gateway: se ejecutan para cada petición, independientemente de la ruta que haya hecho match. Mientras que los `GatewayFilter` son instancias por ruta que transforman una petición concreta, los `GlobalFilter` son beans singleton que implementan lógica transversal. Sin ellos, el gateway no podría resolver `lb://` a instancias reales, no haría proxy HTTP/HTTPS, no enrutaría WebSockets, y no emitiría métricas. El desarrollador necesita personalizar los GlobalFilters de Gateway para aplicar lógica transversal a todas las rutas controlando el orden de ejecución.

> [CONCEPTO] La distinción entre `GatewayFilter` y `GlobalFilter` es arquitectural: un `GatewayFilter` existe en el contexto de una `Route` (puede acceder a ella vía `exchange.getAttribute(GATEWAY_ROUTE_ATTR)`); un `GlobalFilter` es agnóstico de la ruta y se aplica antes, durante o después del procesamiento de todas.

## Diagrama: cadena de GlobalFilters built-in con sus órdenes

El siguiente diagrama muestra los GlobalFilters del sistema y su posición en la cadena de ejecución (orden numérico, menor = mayor prioridad).

```
Petición entrante
      │
      ▼  orden: Integer.MIN_VALUE (antes de todo)
GatewayMetricsFilter        → registra inicio de la petición en Micrometer
      │
      ▼  orden: -10000
RouteToRequestUrlFilter     → construye la URL de destino desde la Route seleccionada
      │                        (resuelve variables de path, scheme, host)
      ▼  orden: -1
ReactiveLoadBalancerClientFilter → si URI scheme es lb://, resuelve a instancia real
      │                            vía Spring Cloud LoadBalancer
      ▼  orden: 0 (aprox)
WebsocketRoutingFilter      → si Upgrade: websocket, hace proxy del WebSocket
      │
      ▼  orden: 1
ForwardRoutingFilter        → si scheme forward://, hace forward interno (sin proxy)
      │
      ▼  orden: 2147483647 (LOWEST_PRECEDENCE - 1)
NettyRoutingFilter          → proxy HTTP/HTTPS vía Reactor Netty (la llamada real al backend)
      │
      ▼  orden: 2147483647 (LOWEST_PRECEDENCE)
NettyWriteResponseFilter    → escribe la respuesta Netty al cliente
      │
      ▼
   Cliente
```

## Ejemplo central

El siguiente ejemplo muestra cómo implementar un `GlobalFilter` personalizado que añade un `X-Correlation-Id` a todas las peticiones, con gestión de orden de ejecución.

```java
package com.example.gateway;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.cloud.gateway.route.Route;
import org.springframework.cloud.gateway.support.ServerWebExchangeUtils;
import org.springframework.core.Ordered;
import org.springframework.http.server.reactive.ServerHttpRequest;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

import java.util.UUID;

/**
 * GlobalFilter que añade X-Correlation-Id a todas las peticiones
 * y registra la ruta seleccionada en el log.
 *
 * Implementa Ordered para controlar la posición en la cadena:
 * orden = -5 → se ejecuta después de RouteToRequestUrlFilter (-10000)
 *              pero antes de ReactiveLoadBalancerClientFilter (-1)
 */
@Component
public class CorrelationIdGlobalFilter implements GlobalFilter, Ordered {

    private static final Logger log = LoggerFactory.getLogger(CorrelationIdGlobalFilter.class);

    @Override
    public int getOrder() {
        return -5; // entre RouteToRequestUrl y ReactiveLoadBalancerClient
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // Obtener o generar el Correlation ID
        String correlationId = exchange.getRequest().getHeaders()
            .getFirst("X-Correlation-Id");
        if (correlationId == null || correlationId.isBlank()) {
            correlationId = UUID.randomUUID().toString();
        }

        final String finalCorrelationId = correlationId;

        // Obtener la ruta seleccionada (disponible tras RouteToRequestUrlFilter)
        Route route = exchange.getAttribute(ServerWebExchangeUtils.GATEWAY_ROUTE_ATTR);
        String routeId = (route != null) ? route.getId() : "unknown";

        log.info("REQUEST correlationId={} routeId={} method={} path={}",
            finalCorrelationId, routeId,
            exchange.getRequest().getMethod(),
            exchange.getRequest().getPath().value());

        // Mutate el request para añadir el header al proxy hacia el backend
        ServerHttpRequest mutatedRequest = exchange.getRequest().mutate()
            .header("X-Correlation-Id", finalCorrelationId)
            .build();

        ServerWebExchange mutatedExchange = exchange.mutate()
            .request(mutatedRequest)
            .build();

        // Continuar la cadena y añadir el header también a la response (POST)
        return chain.filter(mutatedExchange)
            .then(Mono.fromRunnable(() -> {
                // POST processing: añadir el correlation ID a la response
                // Nota: la response puede ya estar comprometida en este punto
                // si NettyWriteResponseFilter ya escribió. Verificar antes de mutar.
                if (!mutatedExchange.getResponse().isCommitted()) {
                    mutatedExchange.getResponse().getHeaders()
                        .add("X-Correlation-Id", finalCorrelationId);
                }
            }));
    }
}
```

### GlobalFilter con lógica de autorización interna (ejemplo avanzado)

```java
package com.example.gateway;

import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.core.Ordered;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

/**
 * GlobalFilter que verifica un header de autenticación interna
 * para todas las rutas que no son públicas.
 * Orden negativo alto: se ejecuta muy temprano en la cadena.
 */
@Component
public class InternalAuthGlobalFilter implements GlobalFilter, Ordered {

    @Override
    public int getOrder() {
        return -100; // antes de cualquier otro filtro personalizado
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String path = exchange.getRequest().getPath().value();

        // Las rutas públicas no requieren verificación
        if (path.startsWith("/api/public/") || path.startsWith("/actuator/health")) {
            return chain.filter(exchange);
        }

        String internalToken = exchange.getRequest().getHeaders()
            .getFirst("X-Internal-Service-Token");

        if ("expected-internal-token".equals(internalToken)) {
            return chain.filter(exchange);
        }

        // Cortar la cadena: devolver 401 sin llamar al backend
        exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
        return exchange.getResponse().setComplete();
    }
}
```

## Tabla de elementos clave

La siguiente tabla describe los GlobalFilters built-in con sus órdenes y función.

| GlobalFilter | Orden | Función |
|--------------|-------|---------|
| `GatewayMetricsFilter` | `Integer.MIN_VALUE` | Registra métricas Micrometer de la petición |
| `RouteToRequestUrlFilter` | `-10000` | Construye la URL final de destino desde la `Route` |
| `ReactiveLoadBalancerClientFilter` | `-1` | Resuelve `lb://` a instancia real vía Spring Cloud LoadBalancer |
| `WebsocketRoutingFilter` | `Integer.MIN_VALUE + 1` | Proxy de WebSocket si hay upgrade |
| `ForwardRoutingFilter` | `Integer.MAX_VALUE - 1` | Maneja el scheme `forward://` |
| `NettyRoutingFilter` | `Integer.MAX_VALUE - 1` | Realiza la llamada HTTP/HTTPS al backend vía Reactor Netty |
| `NettyWriteResponseFilter` | `Integer.MAX_VALUE` | Escribe la respuesta Netty al cliente |

> [EXAMEN] El orden `Integer.MAX_VALUE - 1` de `NettyRoutingFilter` significa que se ejecuta casi al final de la cadena PRE y es el primero en la cadena POST. Cualquier `GlobalFilter` con orden > `NettyRoutingFilter.ORDER` pero < `Integer.MAX_VALUE` se ejecuta como POST-filter después de que el backend ya ha respondido.

> [ADVERTENCIA] Tras `NettyWriteResponseFilter` (orden `Integer.MAX_VALUE`), la response ya está comprometida y no se pueden modificar cabeceras ni el status code. Los filtros POST-processing deben verificar `exchange.getResponse().isCommitted()` antes de intentar mutar la response.

## Buenas y malas prácticas

**Hacer:**
- Implementar `Ordered` en lugar de `@Order` para `GlobalFilter`: la interfaz `Ordered` es más explícita y evita conflictos con la ordenación de Spring beans en el contexto de autoconfiguración.
- Usar `Ordered.HIGHEST_PRECEDENCE` o valores negativos altos para filtros de auditoría y seguridad: se ejecutan antes del routing y tienen visibilidad de la petición completa original.
- Verificar `exchange.getResponse().isCommitted()` en lógica POST-filter antes de modificar headers de response: si el streaming de la respuesta ya comenzó, las modificaciones se pierden silenciosamente.
- Obtener el `GATEWAY_ROUTE_ATTR` del exchange para acceder a metadatos de la ruta en el filtro: permite construir lógica condicional basada en qué ruta hizo match.

**Evitar:**
- Crear `GlobalFilter` que hacen llamadas bloqueantes (JDBC, APIs síncronas): el event loop de Reactor Netty es single-threaded por worker; una llamada bloqueante congela el procesamiento de todas las peticiones concurrentes en ese worker.
- Usar `Integer.MIN_VALUE` como orden en filtros personalizados: puede interferir con `GatewayMetricsFilter` (que usa ese mismo orden) y producir contabilización incorrecta de métricas.
- Modificar el body de la request en un `GlobalFilter` con buffer manual: la gestión de `DataBuffer` en Reactor Netty es propensa a memory leaks si no se libera correctamente; usar `ModifyRequestBodyGatewayFilterFactory` por ruta es más seguro.

---

← [3.4.7 Filtros de seguridad y sesión](sc-gateway-filtros-seguridad.md) | [Índice](README.md) | [3.6 Integración con Service Discovery y Load Balancer](sc-gateway-discovery.md) →
