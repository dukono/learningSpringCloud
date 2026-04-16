# 3.4.3 Filtros de response (cabeceras, status y body)

← [3.4.2 Filtros de path y parámetros](sc-gateway-filtros-path.md) | [Índice](README.md) | [3.4.4 Rate Limiting con RedisRateLimiter](sc-gateway-ratelimiting.md) →

---

## Introducción

Los filtros de response actúan en la fase POST del procesamiento: después de que el backend ha respondido y antes de que el gateway devuelva la respuesta al cliente. Sin ellos, el cliente recibiría la respuesta del backend sin transformar: con cabeceras internas expuestas (ej: `X-Powered-By`, IPs internas en `Location`), status codes no ajustados al contrato del API, o body en un formato que el cliente no espera. El desarrollador necesita transformar la respuesta en Gateway para adaptar cabeceras, status code y body antes de devolverlos al cliente. Este fichero cubre ocho filtros: `AddResponseHeader`, `RemoveResponseHeader`, `SetResponseHeader`, `DedupeResponseHeader`, `RewriteLocationResponseHeader`, `SetStatus`, `RedirectTo` y `ModifyResponseBody`.

> [ADVERTENCIA] Al igual que `ModifyRequestBody`, `ModifyResponseBody` almacena el body completo en memoria. Si el backend devuelve payloads grandes sin que `spring.cloud.gateway.httpclient.codec.max-in-memory-size` esté correctamente configurado, se lanza `DataBufferLimitException`. Ver [3.7 Timeouts y configuración del cliente HTTP](sc-gateway-timeouts.md).

## Diagrama: posición de los filtros de response en la cadena

El siguiente diagrama muestra que los filtros de response se ejecutan en orden inverso (DESC) al retornar la respuesta al cliente.

```
Cliente  ←  Gateway  ←  Backend
                │
     ┌──────────┴─────────────────────────────────┐
     │  POST-FILTERS (orden DESC)                  │
     │  8. ModifyResponseBody (buffer body)        │
     │  7. DedupeResponseHeader (deduplicación)    │
     │  6. RewriteLocationResponseHeader           │
     │  5. AddResponseHeader / SetResponseHeader   │
     │  4. RemoveResponseHeader                    │
     │  3. SetStatus (override del status code)    │
     │  2. RedirectTo (302/301 + Location header)  │
     └──────────┬─────────────────────────────────┘
                │ response transformada
                ▼
            Cliente
```

## Ejemplo central

El siguiente ejemplo muestra la configuración YAML con todos los filtros de response, seguido de la variante programática para `ModifyResponseBody`.

```yaml
# application.yml
spring:
  cloud:
    gateway:
      routes:

        # ── AddResponseHeaderGatewayFilterFactory ─────────────────────
        # Añade una cabecera a la respuesta. Si ya existe, crea multi-value.
        - id: add-response-header-route
          uri: lb://catalog-service
          predicates:
            - Path=/api/catalog/**
          filters:
            - AddResponseHeader=X-Cache, MISS
            - AddResponseHeader=X-Gateway-Route, catalog-route

        # ── SetResponseHeaderGatewayFilterFactory ─────────────────────
        # Sobreescribe una cabecera de response. Si no existe, la crea.
        - id: set-response-header-route
          uri: lb://user-service
          predicates:
            - Path=/api/users/**
          filters:
            - SetResponseHeader=Content-Type, application/json;charset=UTF-8
            - SetResponseHeader=Cache-Control, no-cache, no-store

        # ── RemoveResponseHeaderGatewayFilterFactory ───────────────────
        # Elimina cabeceras que no deben exponerse al cliente externo.
        - id: remove-response-header-route
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
          filters:
            - RemoveResponseHeader=X-Powered-By
            - RemoveResponseHeader=Server
            - RemoveResponseHeader=X-AspNet-Version

        # ── DedupeResponseHeaderGatewayFilterFactory ───────────────────
        # Elimina duplicados en cabeceras multi-value.
        # Estrategias: RETAIN_FIRST (default), RETAIN_LAST, RETAIN_UNIQUE
        # Necesario cuando backend y Gateway ambos añaden la misma cabecera.
        - id: dedupe-header-route
          uri: lb://api-service
          predicates:
            - Path=/api/**
          filters:
            - DedupeResponseHeader=Access-Control-Allow-Credentials RETAIN_FIRST
            - DedupeResponseHeader=Access-Control-Allow-Origin RETAIN_UNIQUE

        # ── RewriteLocationResponseHeaderGatewayFilterFactory ──────────
        # Reescribe el header Location para ocultar la URL interna del backend.
        # stripVersionMode: NEVER_STRIP, AS_IN_REQUEST, ALWAYS_STRIP
        # locationHeaderName: cabecera a reescribir (default: Location)
        # hostValue: host a sustituir (default: el host del request del cliente)
        - id: rewrite-location-route
          uri: lb://resource-service
          predicates:
            - Path=/api/resources/**
          filters:
            - RewriteLocationResponseHeader=AS_IN_REQUEST, Location, , ^http?://[^/]*/(?<remaining>.*)
            # Transforma: http://resource-service:8080/resources/123
            #          en: https://api.example.com/api/resources/123

        # ── SetStatusGatewayFilterFactory ─────────────────────────────
        # Sobreescribe el status code de la response.
        # Acepta código numérico o nombre HttpStatus (OK, NOT_FOUND, etc.)
        - id: set-status-route
          uri: lb://legacy-service
          predicates:
            - Path=/api/legacy/**
          filters:
            - SetStatus=200   # el legacy devuelve 204; forzamos 200 para clientes que fallan con 204

        # ── RedirectToGatewayFilterFactory ─────────────────────────────
        # Redirige a la URL indicada con el status 301 o 302.
        # No hace proxy: retorna al cliente directamente con el Location header.
        - id: redirect-route
          uri: http://ignore-this-uri
          predicates:
            - Path=/old-api/**
          filters:
            - RedirectTo=301, https://api.example.com/api/v2/
```

### ModifyResponseBody — transformación reactiva del body de response

```java
package com.example.gateway;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.cloud.gateway.filter.factory.rewrite.RewriteFunction;
import org.springframework.cloud.gateway.route.RouteLocator;
import org.springframework.cloud.gateway.route.builder.RouteLocatorBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.MediaType;
import reactor.core.publisher.Mono;

@Configuration
public class ResponseBodyConfig {

    @Bean
    public RouteLocator modifyResponseBodyRoute(RouteLocatorBuilder builder, ObjectMapper mapper) {
        // Transforma el body JSON de response: añade campo "source":"gateway"
        RewriteFunction<String, String> addSourceField = (exchange, body) -> {
            if (body == null || body.isBlank()) return Mono.just(body != null ? body : "");
            try {
                var node = mapper.readTree(body);
                if (node.isObject()) {
                    ((com.fasterxml.jackson.databind.node.ObjectNode) node)
                        .put("source", "gateway");
                }
                return Mono.just(mapper.writeValueAsString(node));
            } catch (Exception e) {
                return Mono.just(body);
            }
        };

        return builder.routes()
            .route("modify-response-body-route", r -> r
                .path("/api/products/**")
                .filters(f -> f
                    .modifyResponseBody(String.class, String.class,
                        MediaType.APPLICATION_JSON_VALUE, addSourceField))
                .uri("lb://product-service"))
            .build();
    }
}
```

## Tabla de elementos clave

La siguiente tabla describe los ocho filtros de response con sus diferencias de comportamiento.

| Filtro | Parámetros | Efecto | Nota |
|--------|-----------|--------|------|
| `AddResponseHeader` | nombre, valor | Añade la cabecera; si existe, crea multi-value | Puede generar duplicados si el backend también la añade; usar `DedupeResponseHeader` |
| `SetResponseHeader` | nombre, valor | Sobreescribe la cabecera; si no existe, la crea | Reemplaza cualquier valor del backend |
| `RemoveResponseHeader` | nombre | Elimina la cabecera de la response | Case-insensitive |
| `DedupeResponseHeader` | nombre(s) estrategia | Elimina valores duplicados de cabeceras multi-value | Estrategias: `RETAIN_FIRST`, `RETAIN_LAST`, `RETAIN_UNIQUE` |
| `RewriteLocationResponseHeader` | stripVersionMode, locationHeaderName, hostValue, protocols | Reescribe el header `Location` para ocultar URLs internas | Necesario cuando el backend devuelve URLs absolutas con host interno |
| `SetStatus` | código HTTP (int o nombre) | Sobreescribe el status code devuelto al cliente | El status original del backend queda en el atributo `ORIGINAL_RESPONSE_STATUS_CODE` |
| `RedirectTo` | código (301/302), URL | Devuelve una redirección HTTP al cliente; no hace proxy | El campo `uri` de la ruta es ignorado para el proxy |
| `ModifyResponseBody` | inClass, outClass, contentType, RewriteFunction | Transforma el body completo en memoria | Requiere configurar `codec.max-in-memory-size` para payloads > 256 KB |

> [EXAMEN] `DedupeResponseHeader` con estrategia `RETAIN_UNIQUE` es distinto de `RETAIN_FIRST`: `RETAIN_UNIQUE` mantiene todos los valores distintos (elimina solo duplicados exactos), mientras que `RETAIN_FIRST` solo mantiene el primero de todos los valores, incluso si son distintos.

## Buenas y malas prácticas

**Hacer:**
- Usar `RemoveResponseHeader` para eliminar cabeceras que exponen tecnología interna (`Server`, `X-Powered-By`, `X-AspNet-Version`): reduce la superficie de información para atacantes.
- Usar `DedupeResponseHeader=Access-Control-Allow-Origin RETAIN_UNIQUE` cuando Spring Security y Gateway ambos añaden cabeceras CORS: evita el error del navegador "Multiple values in CORS header".
- Usar `RewriteLocationResponseHeader` en APIs que devuelven `201 Created` con `Location` apuntando a URLs internas del backend: el cliente externo recibe una URL válida del gateway, no la IP interna del pod.
- En `ModifyResponseBody`, validar que el body no es `null` o vacío antes de procesarlo: los responses 204 No Content tienen body null; procesarlo sin validación lanza NullPointerException en la lambda.

**Evitar:**
- Usar `SetStatus=200` para enmascarar errores del backend (4xx, 5xx): los clientes que confían en el status code para el manejo de errores recibirán respuestas incorrectas, dificultando el debugging.
- Usar `ModifyResponseBody` en rutas de alto volumen con payloads grandes sin ajustar `max-in-memory-size` y el pool de conexiones: almacenar el body completo en memoria en miles de peticiones concurrentes puede agotar el heap del gateway.
- Usar `AddResponseHeader` para añadir `Access-Control-Allow-Origin` manualmente cuando ya existe configuración CORS en Gateway (`spring.cloud.gateway.globalcors`): el resultado son cabeceras duplicadas que el navegador rechaza.

---

← [3.4.2 Filtros de path y parámetros](sc-gateway-filtros-path.md) | [Índice](README.md) | [3.4.4 Rate Limiting con RedisRateLimiter](sc-gateway-ratelimiting.md) →
