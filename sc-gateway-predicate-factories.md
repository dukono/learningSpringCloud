# 5.3 Predicate Factories built-in

← [5.2 Configuración de rutas](sc-gateway-configuracion-rutas.md) | [Índice](README.md) | [5.4 GatewayFilter Factories built-in](sc-gateway-filter-factories-builtin.md) →

---

## Introducción

Las Predicate Factories son los mecanismos de matching que determinan si una petición HTTP entrante activa una ruta concreta. Spring Cloud Gateway incluye un catálogo de Predicate Factories built-in que cubren los casos de uso más comunes: matching por path, método HTTP, headers, parámetros de query, host, cookie, rangos temporales, IP de origen y pesos de tráfico. Cuando una ruta define múltiples predicates, todos deben evaluarse como verdaderos (AND implícito) para que la ruta se active.

> [CONCEPTO] Un **Predicate Factory** es un componente que, dado su configuración (el texto después de `=` en YAML), produce un `Predicate<ServerWebExchange>` que evalúa la petición en tiempo de ejecución. El nombre del shortcut en YAML (e.g., `Path`, `Method`) corresponde al nombre de la clase menos el sufijo `RoutePredicateFactory`.

## Mapa de Predicate Factories

El siguiente diagrama muestra todas las Predicate Factories built-in organizadas por categoría de matching:

```
Predicate Factories built-in
│
├── URL / PATH
│   └── Path=/api/orders/**, /api/items/**
│
├── MÉTODO HTTP
│   └── Method=GET,POST,PUT,DELETE
│
├── HEADERS HTTP
│   ├── Header=X-Request-Id, \d+       (nombre, regex del valor)
│   └── Host=**.example.com, **.test.com
│
├── PARÁMETROS / COOKIES
│   ├── Query=color, gree.            (nombre, regex del valor — opcional)
│   └── Cookie=sessionId, abc.        (nombre, regex del valor)
│
├── TEMPORAL
│   ├── Before=2025-01-01T00:00:00+01:00[Europe/Madrid]
│   ├── After=2024-06-01T00:00:00+01:00[Europe/Madrid]
│   └── Between=2024-06-01T00:00:00+01:00[Europe/Madrid], 2025-01-01T00:00:00+01:00[Europe/Madrid]
│
├── DIRECCIÓN IP
│   ├── RemoteAddr=192.168.1.0/24, 10.0.0.1
│   └── XForwardedRemoteAddr=192.168.1.0/24
│
└── DISTRIBUCIÓN DE TRÁFICO
    └── Weight=group1, 8              (grupo, peso relativo)
```

## Ejemplo central

El siguiente ejemplo muestra todas las Predicate Factories built-in con su sintaxis YAML y el equivalente Java DSL para las más relevantes:

```yaml
# application.yml — Catálogo completo de Predicate Factories
spring:
  cloud:
    gateway:
      routes:
        # --- PATH ---
        - id: path-example
          uri: lb://product-service
          predicates:
            # Acepta uno o más patterns separados por coma
            - Path=/api/products/**, /api/items/**

        # --- METHOD ---
        - id: method-example
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
            - Method=GET,POST

        # --- HEADER ---
        - id: header-example
          uri: lb://internal-service
          predicates:
            - Path=/internal/**
            # Header con validación de valor via regex
            - Header=X-Internal-Token, secret-\d{4}

        # --- HOST ---
        - id: host-example
          uri: lb://web-service
          predicates:
            # Ant-style pattern; el host capturado queda disponible como variable
            - Host=**.shop.example.com, **.api.example.com

        # --- QUERY PARAMETER ---
        - id: query-example
          uri: lb://search-service
          predicates:
            - Path=/search/**
            # Query solo por nombre (sin validar valor)
            - Query=q
            # Query con validación de valor via regex
            # - Query=color, gree.   (coincidiría con "green", "grey", etc.)

        # --- COOKIE ---
        - id: cookie-example
          uri: lb://user-service
          predicates:
            - Path=/profile/**
            - Cookie=session, [a-f0-9]{32}

        # --- TEMPORAL: BEFORE / AFTER / BETWEEN ---
        - id: beta-feature
          uri: lb://beta-service
          predicates:
            - Path=/beta/**
            # Disponible solo a partir de esta fecha (ZonedDateTime ISO-8601)
            - After=2024-06-01T00:00:00+01:00[Europe/Madrid]

        - id: promo-feature
          uri: lb://promo-service
          predicates:
            - Path=/promo/**
            - Between=2024-11-01T00:00:00+01:00[Europe/Madrid], 2024-11-30T23:59:59+01:00[Europe/Madrid]

        # --- REMOTE ADDR (IP directa del cliente) ---
        - id: admin-direct
          uri: lb://admin-service
          predicates:
            - Path=/admin/**
            # CIDR notation o IP exacta
            - RemoteAddr=192.168.1.0/24, 10.0.0.1/32

        # --- X-FORWARDED-REMOTE-ADDR (cliente detrás de proxy) ---
        - id: admin-proxied
          uri: lb://admin-service
          predicates:
            - Path=/admin-proxy/**
            # Usa el header X-Forwarded-For enviado por el proxy upstream
            - XForwardedRemoteAddr=192.168.1.0/24

        # --- WEIGHT (distribución de tráfico A/B) ---
        - id: service-v1
          uri: lb://service-v1
          predicates:
            - Path=/api/v1/**
            # 80% del tráfico del grupo "service-group" va a v1
            - Weight=service-group, 8

        - id: service-v2
          uri: lb://service-v2
          predicates:
            - Path=/api/v1/**
            # 20% del tráfico del grupo "service-group" va a v2 (canary)
            - Weight=service-group, 2
```

```java
package com.example.gateway.config;

import org.springframework.cloud.gateway.route.RouteLocator;
import org.springframework.cloud.gateway.route.builder.RouteLocatorBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.time.ZonedDateTime;

@Configuration
public class PredicateExamplesConfig {

    @Bean
    public RouteLocator predicateRoutes(RouteLocatorBuilder builder) {
        return builder.routes()
            // Path + Method combinados (AND implícito)
            .route("order-get-post", r -> r
                .path("/api/orders/**")
                .and()
                .method("GET", "POST")
                .uri("lb://order-service"))

            // Header con regex de valor
            .route("internal-header", r -> r
                .path("/internal/**")
                .and()
                .header("X-Internal-Token", "secret-\\d{4}")
                .uri("lb://internal-service"))

            // After temporal
            .route("beta-after", r -> r
                .path("/beta/**")
                .and()
                .after(ZonedDateTime.parse("2024-06-01T00:00:00+01:00[Europe/Madrid]"))
                .uri("lb://beta-service"))

            // Weight para canary deployment
            .route("service-v1", r -> r
                .path("/api/v1/**")
                .and()
                .weight("service-group", 8)
                .uri("lb://service-v1"))

            .route("service-v2", r -> r
                .path("/api/v1/**")
                .and()
                .weight("service-group", 2)
                .uri("lb://service-v2"))
            .build();
    }
}
```

## Tabla de Predicate Factories built-in

La siguiente tabla resume todas las Predicate Factories con sus parámetros y caso de uso principal:

| Factory | Shortcut YAML | Parámetros | Caso de uso |
|---|---|---|---|
| `PathRoutePredicateFactory` | `Path=` | Uno o más ant-patterns | Enrutamiento por URL (el más común) |
| `MethodRoutePredicateFactory` | `Method=` | Lista de métodos HTTP | Restringir métodos permitidos |
| `HeaderRoutePredicateFactory` | `Header=` | nombre, regex-valor | Routing por header de seguridad |
| `HostRoutePredicateFactory` | `Host=` | Ant-patterns del Host | Multi-tenant por dominio |
| `QueryRoutePredicateFactory` | `Query=` | nombre[, regex-valor] | Feature flags via query param |
| `CookieRoutePredicateFactory` | `Cookie=` | nombre, regex-valor | Routing por sesión/cookie |
| `BeforeRoutePredicateFactory` | `Before=` | ZonedDateTime ISO-8601 | Deshabilitar ruta en fecha |
| `AfterRoutePredicateFactory` | `After=` | ZonedDateTime ISO-8601 | Activar ruta en fecha futura |
| `BetweenRoutePredicateFactory` | `Between=` | ZonedDateTime, ZonedDateTime | Ventana temporal (promos) |
| `RemoteAddrRoutePredicateFactory` | `RemoteAddr=` | Lista CIDR/IPs | Restringir por IP directa |
| `XForwardedRemoteAddrRoutePredicateFactory` | `XForwardedRemoteAddr=` | Lista CIDR/IPs | Restringir por IP detrás de proxy |
| `WeightRoutePredicateFactory` | `Weight=` | grupo, peso-entero | Canary deployments / A/B testing |

## Combinación de predicates y orden de rutas

Cuando una ruta define múltiples predicates, todos deben evaluarse como verdaderos (AND lógico) para que la ruta se active. No existe sintaxis OR nativa entre predicates de la misma ruta; para OR se deben definir dos rutas separadas.

El orden en que las rutas se evalúan importa: Gateway evalúa las rutas en orden de definición (YAML: de arriba a abajo; Java DSL: orden de llamada a `.route()`). La primera ruta cuyos predicates coincidan gana. Las rutas más específicas deben definirse antes que las genéricas.

> [EXAMEN] El predicate `Weight` no filtra peticiones: todas las peticiones que coincidan con el `Path` serán enrutadas, distribuyéndose entre las rutas del mismo grupo según el peso relativo (no porcentaje). `Weight=group, 8` y `Weight=group, 2` reparten tráfico 80%/20%.

> [ADVERTENCIA] `RemoteAddr` usa la IP de la conexión TCP directa. Si el Gateway está detrás de un load balancer o proxy inverso, la IP directa es la del proxy, no la del cliente real. En ese caso usa `XForwardedRemoteAddr` con el header `X-Forwarded-For`.

## Buenas y malas prácticas

**Buenas prácticas:**
- Ordenar rutas de más específica a más general en el YAML para evitar enmascaramiento.
- Usar `Host` predicate para implementar virtual hosting (múltiples dominios en un gateway).
- Usar `Weight` para canary deployments controlados antes de promover una nueva versión al 100%.
- Validar predicates temporales (`Between`) con fechas en formato ISO-8601 con zona horaria explícita.

**Malas prácticas:**
- Usar `RemoteAddr` para seguridad en entornos con proxies sin complementar con `XForwardedRemoteAddr`.
- Definir una ruta con solo `Path=/**` sin otros predicates al inicio: bloquea todas las rutas siguientes.
- Confiar en el predicate `Header` para autenticación sin un filtro de seguridad adicional.

## Verificación y práctica

1. Dado el predicado `- Path=/api/orders/**, /api/items/**`, ¿qué peticiones coinciden? ¿Y si se añade `- Method=GET`?

2. ¿Cómo se combinan lógicamente múltiples predicates en una misma ruta? ¿Existe OR nativo entre predicates?

3. ¿Qué diferencia hay entre `RemoteAddr` y `XForwardedRemoteAddr`? ¿Cuándo usarías cada uno?

4. Si tienes dos rutas con el mismo `Path=/api/v1/**` y predicates `Weight=group, 8` y `Weight=group, 2`, ¿qué porcentaje de tráfico recibe cada una?

5. ¿En qué orden evalúa Spring Cloud Gateway las rutas definidas en YAML? ¿Cómo afecta este orden a una ruta `Path=/**` definida al principio?

---

← [5.2 Configuración de rutas](sc-gateway-configuracion-rutas.md) | [Índice](README.md) | [5.4 GatewayFilter Factories built-in](sc-gateway-filter-factories-builtin.md) →
