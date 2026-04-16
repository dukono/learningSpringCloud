# 3.4.2 Filtros de path y parámetros

← [3.4.1 Filtros de cabeceras del request](sc-gateway-filtros-request-headers.md) | [Índice](README.md) | [3.4.3 Filtros de response (cabeceras, status y body)](sc-gateway-filtros-response.md) →

---

## Introducción

Los filtros de path y parámetros reescriben la URL del request antes de enviarlo al backend. Sin ellos, el backend recibiría el path tal como lo envió el cliente, incluyendo prefijos de API gateway (`/api/v1/`) que los servicios individuales no conocen ni esperan. El desarrollador necesita reescribir paths y parámetros del request en Gateway para normalizar URLs entre el gateway y los servicios backend. Los seis filtros de este grupo cubren todas las transformaciones de URL habituales: eliminación de prefijos, reescritura con regex, establecimiento de path fijo, y manipulación de query parameters.

> [ADVERTENCIA] Los filtros de path actúan sobre el path de la URL que se envía al backend, pero **no** modifican las variables de URI template capturadas por `PathRoutePredicateFactory`. Si el filtro `RewritePath` cambia la estructura del path, las variables capturadas por el predicado `Path` mantienen sus valores originales en el exchange.

## Diagrama: transformaciones de path disponibles

El siguiente diagrama muestra qué hace cada filtro sobre un ejemplo de URL real.

```
URL original del cliente: /api/v2/catalog/electronics/items?page=1&debug=true

┌─────────────────────────────────────────────────────────────────────┐
│ StripPrefix=2       → /catalog/electronics/items?page=1&debug=true  │
│                       (elimina los 2 primeros segmentos)             │
├─────────────────────────────────────────────────────────────────────┤
│ PrefixPath=/internal → /internal/api/v2/catalog/...                 │
│                         (añade prefijo al path completo)             │
├─────────────────────────────────────────────────────────────────────┤
│ SetPath=/items/{cat} → /items/electronics  (path fijo con variable)  │
├─────────────────────────────────────────────────────────────────────┤
│ RewritePath=/api/v2/(?<rest>.*), /$\{rest}                           │
│              → /catalog/electronics/items?page=1&debug=true          │
├─────────────────────────────────────────────────────────────────────┤
│ AddRequestParameter=source,gateway → ?page=1&debug=true&source=gw   │
├─────────────────────────────────────────────────────────────────────┤
│ RemoveRequestParameter=debug       → ?page=1                         │
└─────────────────────────────────────────────────────────────────────┘
```

## Ejemplo central

El siguiente ejemplo muestra una configuración YAML con los seis filtros y una variante programática para `RewritePath`, que es el filtro más utilizado y el que tiene más casos edge.

```yaml
# application.yml
spring:
  cloud:
    gateway:
      routes:

        # ── StripPrefixGatewayFilterFactory ───────────────────────────
        # Elimina N segmentos del inicio del path.
        # /api/orders/123 con StripPrefix=1 → /orders/123
        - id: strip-prefix-route
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
          filters:
            - StripPrefix=1   # elimina /api

        # ── PrefixPathGatewayFilterFactory ────────────────────────────
        # Añade un prefijo fijo al path.
        # /users/123 con PrefixPath=/v2 → /v2/users/123
        - id: prefix-path-route
          uri: lb://user-service
          predicates:
            - Path=/users/**
          filters:
            - PrefixPath=/v2

        # ── SetPathGatewayFilterFactory ───────────────────────────────
        # Establece el path a un valor fijo, con soporte de variables
        # URI template capturadas por el predicado Path.
        # /catalog/electronics → /internal/categories/electronics
        - id: set-path-route
          uri: lb://catalog-service
          predicates:
            - Path=/catalog/{category}
          filters:
            - SetPath=/internal/categories/{category}

        # ── RewritePathGatewayFilterFactory ───────────────────────────
        # Reescritura con regex y grupos de captura.
        # Nota: en YAML la barra invertida de la variable debe escaparse como $\
        # /api/v1/catalog/items/42 → /items/42
        - id: rewrite-path-route
          uri: lb://catalog-service
          predicates:
            - Path=/api/v1/catalog/**
          filters:
            - RewritePath=/api/v1/catalog/(?<segment>.*), /$\{segment}

        # ── AddRequestParameterGatewayFilterFactory ───────────────────
        # Añade un query parameter al request hacia el backend.
        # Si el parámetro ya existe, se añade como valor adicional.
        - id: add-param-route
          uri: lb://search-service
          predicates:
            - Path=/search/**
          filters:
            - AddRequestParameter=source, gateway
            - AddRequestParameter=version, 2

        # ── RemoveRequestParameterGatewayFilterFactory ────────────────
        # Elimina un query parameter antes de enviar al backend.
        # Útil para limpiar parámetros de tracking o debug del cliente.
        - id: remove-param-route
          uri: lb://analytics-service
          predicates:
            - Path=/analytics/**
          filters:
            - RemoveRequestParameter=utm_source
            - RemoveRequestParameter=utm_medium
            - RemoveRequestParameter=debug
```

### RewritePath con RouteLocatorBuilder (variante programática)

```java
package com.example.gateway;

import org.springframework.cloud.gateway.route.RouteLocator;
import org.springframework.cloud.gateway.route.builder.RouteLocatorBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class PathRewriteConfig {

    @Bean
    public RouteLocator pathRewriteRoutes(RouteLocatorBuilder builder) {
        return builder.routes()
            // RewritePath equivalente al YAML anterior
            .route("rewrite-path-programmatic", r -> r
                .path("/api/v1/catalog/**")
                .filters(f -> f
                    // En Java, la variable se referencia con ${segment}
                    .rewritePath("/api/v1/catalog/(?<segment>.*)", "/${segment}"))
                .uri("lb://catalog-service"))

            // Combinación habitual: StripPrefix + AddRequestParameter
            .route("strip-and-param", r -> r
                .path("/public/reports/**")
                .filters(f -> f
                    .stripPrefix(1)                              // /reports/**
                    .addRequestParameter("internal", "true")
                    .removeRequestParameter("auth_token"))       // no reenviar tokens al backend
                .uri("lb://report-service"))
            .build();
    }
}
```

## Tabla de elementos clave

La siguiente tabla recoge los seis filtros con sus parámetros y diferencias de comportamiento relevantes para producción.

| Filtro | Parámetros | Efecto sobre el path | Nota |
|--------|-----------|----------------------|------|
| `StripPrefix` | N (entero) | Elimina los primeros N segmentos del path | Si N > número de segmentos, el path queda vacío `/` |
| `PrefixPath` | prefijo (string) | Añade el prefijo al inicio del path | El prefijo debe empezar con `/` |
| `SetPath` | template con `{var}` | Reemplaza el path completo; las variables deben estar capturadas por el predicado `Path` | Si la variable no existe en el exchange, se deja vacía |
| `RewritePath` | regexp, reemplazo | Reescribe el path con grupos de captura regex | En YAML: `$\{group}`; en Java: `${group}` |
| `AddRequestParameter` | nombre, valor | Añade el parámetro al query string | Si ya existe, crea multi-value |
| `RemoveRequestParameter` | nombre | Elimina el parámetro del query string | Case-sensitive en el nombre del parámetro |

> [EXAMEN] En `RewritePath`, la sintaxis en YAML requiere `$\{group}` (con barra invertida para escapar el `$` en Spring expression context), mientras que en el DSL Java se usa `${group}` directamente. Esta diferencia es fuente frecuente de errores de configuración.

## Buenas y malas prácticas

**Hacer:**
- Preferir `StripPrefix` sobre `RewritePath` cuando la transformación es solo eliminar un prefijo fijo: `StripPrefix` es más simple, más legible y no tiene casos edge de regex.
- Usar `RewritePath` con grupos nombrados (`(?<nombre>.*)`) en lugar de grupos posicionales (`(.*)`): los nombres hacen el patrón legible y facilitan el mantenimiento cuando el path tiene múltiples segmentos variables.
- Combinar `StripPrefix` + `PrefixPath` para transformar prefijos de versión (`/v1/orders` → `/orders`): más claro que un `RewritePath` equivalente.
- Usar `RemoveRequestParameter` para eliminar parámetros de tracking (UTM, `fbclid`, etc.) antes de enviar al backend: reduce el volumen de logs del backend y evita que parámetros de marketing contaminen la lógica de negocio.

**Evitar:**
- Usar `SetPath` con variables que no están capturadas por el predicado `Path` de la misma ruta: si la variable no existe en el exchange, el path resultante tendrá segmentos vacíos o un comportamiento indefinido.
- Mezclar `StripPrefix` y `RewritePath` en la misma ruta para transformar el mismo path: el orden de aplicación de los filtros puede producir doble transformación difícil de depurar; elegir uno de los dos.
- Olvidar el prefijo `/` en `PrefixPath`: un `PrefixPath=v2` (sin `/`) genera paths inválidos como `v2/orders`, que Netty no puede resolver correctamente.

---

← [3.4.1 Filtros de cabeceras del request](sc-gateway-filtros-request-headers.md) | [Índice](README.md) | [3.4.3 Filtros de response (cabeceras, status y body)](sc-gateway-filtros-response.md) →
