# 3.4.1 Filtros de cabeceras del request

← [3.3 Predicados de enrutamiento built-in](sc-gateway-predicados.md) | [Índice](README.md) | [3.4.2 Filtros de path y parámetros](sc-gateway-filtros-path.md) →

---

## Introducción

Los filtros de cabeceras del request permiten añadir, eliminar, sobrescribir o transformar las cabeceras HTTP antes de que la petición llegue al servicio backend. Sin ellos, el gateway se limita a redirigir peticiones sin adaptar el contrato HTTP: el backend recibiría cabeceras del cliente que no espera (ej: `Origin`, `X-Real-IP`) o le faltarían cabeceras que requiere (ej: `X-Tenant-Id`, `X-Request-Id`). Este fichero cubre los filtros que actúan sobre el request entrante: `AddRequestHeader`, `RemoveRequestHeader`, `SetRequestHeader`, `SetRequestHostHeader`, `PreserveHostHeader`, `ModifyRequestBody` y `RequestSizeGatewayFilterFactory`.

> [ADVERTENCIA] `ModifyRequestBody` almacena el body completo en memoria antes de pasarlo al backend. Si `spring.cloud.gateway.httpclient.codec.max-in-memory-size` es inferior al tamaño del payload se lanza `DataBufferLimitException` en tiempo de ejecución. La configuración de este límite y su interacción con el pool de conexiones Netty se documenta en [3.7 Timeouts y configuración del cliente HTTP](sc-gateway-timeouts.md).

## Diagrama: posición de los filtros de request en la cadena

El siguiente diagrama muestra en qué punto de la cadena actúan los filtros de cabeceras y body del request, antes del envío al backend.

```
Cliente  →  Gateway  →  Backend
            │
     ┌──────┴────────────────────────────────┐
     │  PRE-FILTERS (orden ASC)              │
     │  1. AddRequestHeader                  │
     │  2. RemoveRequestHeader               │
     │  3. SetRequestHeader                  │
     │  4. SetRequestHostHeader              │
     │  5. PreserveHostHeader (flag)         │
     │  6. RequestSizeGatewayFilter (guard)  │
     │  7. ModifyRequestBody (buffer body)   │
     └──────┬────────────────────────────────┘
            │ request modificado
            ▼
         Backend
            │ response
     ┌──────┴──────────────────────────┐
     │  POST-FILTERS (orden DESC)      │
     └──────┬──────────────────────────┘
            │
         Cliente
```

## Ejemplo central

El siguiente ejemplo muestra una ruta que aplica todos los filtros cubiertos en este fichero, con comentarios sobre el efecto de cada uno.

```java
package com.example.gateway;

import org.springframework.cloud.gateway.filter.factory.rewrite.RewriteFunction;
import org.springframework.cloud.gateway.route.RouteLocator;
import org.springframework.cloud.gateway.route.builder.RouteLocatorBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.MediaType;
import reactor.core.publisher.Mono;

@Configuration
public class RequestHeadersRoutesConfig {

    @Bean
    public RouteLocator requestHeaderRoutes(RouteLocatorBuilder builder) {
        return builder.routes()

            // ── AddRequestHeader ──────────────────────────────────────
            // Añade X-Correlation-Id a cada petición hacia order-service.
            // Si el header ya existe en el request del cliente, se añade
            // un valor adicional (multi-value), NO se sobreescribe.
            .route("add-header-route", r -> r
                .path("/api/orders/**")
                .filters(f -> f
                    .addRequestHeader("X-Correlation-Id", "gw-${spring.application.name}")
                    .addRequestHeader("X-Gateway-Timestamp", String.valueOf(System.currentTimeMillis())))
                .uri("lb://order-service"))

            // ── SetRequestHeader ──────────────────────────────────────
            // Sobreescribe (no añade) el valor del header.
            // Si el header ya existe en el request del cliente, su valor
            // es REEMPLAZADO por el definido aquí.
            .route("set-header-route", r -> r
                .path("/api/catalog/**")
                .filters(f -> f
                    .setRequestHeader("X-Source", "gateway")
                    .setRequestHeader("Accept", "application/json"))
                .uri("lb://catalog-service"))

            // ── RemoveRequestHeader ────────────────────────────────────
            // Elimina headers que no deben llegar al backend.
            // Útil para limpiar cabeceras internas del cliente o proxies.
            .route("remove-header-route", r -> r
                .path("/api/users/**")
                .filters(f -> f
                    .removeRequestHeader("Cookie")
                    .removeRequestHeader("X-Forwarded-Host"))
                .uri("lb://user-service"))

            // ── SetRequestHostHeader ───────────────────────────────────
            // Sobreescribe la cabecera Host de la petición.
            // Necesario cuando el backend valida el Host header y
            // Gateway lo transforma al resolver lb://.
            .route("set-host-route", r -> r
                .path("/api/legacy/**")
                .filters(f -> f
                    .setRequestHostHeader("legacy.internal.example.com"))
                .uri("https://legacy.internal.example.com"))

            // ── PreserveHostHeader ─────────────────────────────────────
            // Flag que indica a Gateway que NO sobreescriba el Host header
            // con el host del backend. Se activa como atributo del exchange.
            .route("preserve-host-route", r -> r
                .path("/api/proxy/**")
                .filters(f -> f.preserveHostHeader())
                .uri("lb://proxy-service"))

            // ── RequestSizeGatewayFilter ───────────────────────────────
            // Rechaza con 413 si el body supera el límite (en bytes o con unidad).
            // Actúa ANTES de leer el body; no consume memoria para peticiones grandes.
            .route("size-limit-route", r -> r
                .path("/api/uploads/**")
                .filters(f -> f.requestSize(5 * 1024 * 1024L)) // 5 MB
                .uri("lb://upload-service"))

            .build();
    }
}
```

### ModifyRequestBody — transformación reactiva del body

`ModifyRequestBody` requiere una `RewriteFunction<In,Out>` que transforma el body. El tipo de entrada y salida puede diferir.

```java
package com.example.gateway;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.cloud.gateway.filter.GatewayFilter;
import org.springframework.cloud.gateway.filter.factory.AbstractGatewayFilterFactory;
import org.springframework.cloud.gateway.filter.factory.rewrite.RewriteFunction;
import org.springframework.cloud.gateway.support.bodyinserter.BodyInserterContext;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.cloud.gateway.route.RouteLocator;
import org.springframework.cloud.gateway.route.builder.RouteLocatorBuilder;
import org.springframework.http.MediaType;
import reactor.core.publisher.Mono;

@Configuration
public class ModifyBodyConfig {

    @Bean
    public RouteLocator modifyBodyRoute(RouteLocatorBuilder builder, ObjectMapper mapper) {
        // RewriteFunction<String, String>: recibe el body como String,
        // lo transforma y devuelve un Mono<String>.
        RewriteFunction<String, String> addAuditField = (exchange, body) -> {
            // Ejemplo: añadir campo "gatewayTimestamp" al JSON del request
            if (body == null) return Mono.just("{}");
            try {
                var node = mapper.readTree(body);
                ((com.fasterxml.jackson.databind.node.ObjectNode) node)
                    .put("gatewayTimestamp", System.currentTimeMillis());
                return Mono.just(mapper.writeValueAsString(node));
            } catch (Exception e) {
                return Mono.just(body); // sin transformación si no es JSON válido
            }
        };

        return builder.routes()
            .route("modify-body-route", r -> r
                .path("/api/events/**")
                .filters(f -> f
                    .modifyRequestBody(String.class, String.class,
                        MediaType.APPLICATION_JSON_VALUE, addAuditField))
                .uri("lb://event-service"))
            .build();
    }
}
```

### Equivalente YAML

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: request-headers-yaml
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
          filters:
            - AddRequestHeader=X-Correlation-Id, gw-orders
            - SetRequestHeader=X-Source, gateway
            - RemoveRequestHeader=Cookie
            - RequestSize=5MB
```

## Tabla de elementos clave

La siguiente tabla recoge los filtros de cabeceras del request con sus parámetros y diferencias clave.

| Filtro | Parámetros | Comportamiento | Nota clave |
|--------|-----------|----------------|------------|
| `AddRequestHeader` | nombre, valor | Añade el header; si ya existe, crea valor adicional (multi-value) | No sobreescribe; usa `SetRequestHeader` si quieres reemplazar |
| `SetRequestHeader` | nombre, valor | Sobreescribe el header; si no existe, lo crea | Reemplaza cualquier valor previo del cliente |
| `RemoveRequestHeader` | nombre | Elimina el header del request | Case-insensitive en el nombre |
| `SetRequestHostHeader` | host | Sobreescribe la cabecera `Host` | Necesario ante backends que validan el Host header |
| `PreserveHostHeader` | (sin parámetros) | Flag: preserva el Host original del cliente | Alternativa a `SetRequestHostHeader` cuando el backend espera el Host del cliente |
| `RequestSize` | tamaño (bytes o `5MB`) | Rechaza con 413 si `Content-Length` supera el límite | No consume el body; actúa sobre `Content-Length` |
| `ModifyRequestBody` | inClass, outClass, contentType, rewriteFunction | Transforma el body completo en memoria | Requiere aumentar `codec.max-in-memory-size` para payloads grandes |

> [EXAMEN] `AddRequestHeader` vs `SetRequestHeader`: la diferencia es crítica cuando el cliente ya envía el header. `Add` produce multi-value (dos valores del mismo header), `Set` reemplaza. Un backend que lee solo el primer valor puede obtener el del cliente en lugar del del gateway si se usa `Add` incorrectamente.

## Buenas y malas prácticas

**Hacer:**
- Usar `RemoveRequestHeader=Cookie` en rutas hacia servicios backend stateless: evita que cookies de sesión del cliente lleguen al servicio y contaminen la lógica de autorización basada en headers.
- Usar `RequestSize` en rutas de upload antes de `ModifyRequestBody`: `RequestSize` actúa sobre `Content-Length` sin leer el body, rechazando payloads grandes con 413 antes de consumir memoria.
- Usar `SetRequestHeader` para headers de contexto de gateway (versión, tenant, correlación) que el backend debe recibir con un valor canónico, sin importar lo que envíe el cliente.
- En `ModifyRequestBody`, devolver siempre un `Mono` no vacío: si la `RewriteFunction` devuelve `Mono.empty()`, Gateway envía un body vacío al backend, no el original.

**Evitar:**
- Usar `ModifyRequestBody` para transformaciones de gran volumen sin configurar `httpclient.codec.max-in-memory-size`: el límite por defecto es 256 KB; con payloads JSON típicos de eventos (1-10 MB) se produce `DataBufferLimitException` en producción.
- Añadir cabeceras de autenticación interna (`X-Internal-Auth`) solo con `AddRequestHeader` sin ningún filtro de seguridad previo: un cliente externo puede enviar el mismo header y bypassear controles de autorización del backend.
- Usar `PreserveHostHeader` y `SetRequestHostHeader` en la misma ruta: se contradicen; `PreserveHostHeader` tiene precedencia porque se evalúa como atributo del exchange antes del envío.

---

← [3.3 Predicados de enrutamiento built-in](sc-gateway-predicados.md) | [Índice](README.md) | [3.4.2 Filtros de path y parámetros](sc-gateway-filtros-path.md) →
