# Parte 6.2 — API Gateway: Routes, Predicates y Configuración

← [Concepto y Arquitectura](./06-01-gateway-concepto.md) | [Volver al índice](./README.md) | Siguiente: [Filtros →](./06-03-gateway-filtros.md)

---

## 6.3 Conceptos: Route, Predicate, Filter

Spring Cloud Gateway tiene tres conceptos fundamentales:

### Route (Ruta)

Una **Route** es la unidad básica de configuración. Define:
- Un **ID** único
- La **URI destino** (a dónde enrutar)
- Un conjunto de **Predicates** (condiciones que deben cumplirse)
- Un conjunto de **Filters** (transformaciones a aplicar)

```
Route: si la petición cumple los Predicates, aplicar los Filters y enrutar a la URI
```

### Predicate

Un **Predicate** es una condición que debe cumplir la petición para que la ruta aplique. En YAML, varios predicates en la misma ruta se evalúan como **AND lógico** (todos deben cumplirse). Para combinarlos con **OR** es necesario usar el Java DSL:

| Predicate | Ejemplo | Descripción |
|---|---|---|
| `Path` | `Path=/pedidos/**` | Por ruta URL |
| `Method` | `Method=GET,POST` | Por método HTTP |
| `Header` | `Header=X-Request-Id, \d+` | Por header (con regex) |
| `Query` | `Query=version, v2` | Por query parameter |
| `Host` | `Host=**.miempresa.com` | Por hostname |
| `After` | `After=2024-01-01T00:00:00Z` | Solo después de una fecha/hora (ISO-8601) |
| `Before` | `Before=2024-12-31T23:59:59Z` | Solo antes de una fecha/hora |
| `Between` | `Between=2024-01-01T00:00:00Z, 2024-12-31T23:59:59Z` | Ventana temporal |
| `Cookie` | `Cookie=sessionId, abc123` | Por valor de cookie (con regex) |
| `RemoteAddr` | `RemoteAddr=192.168.1.0/24` | Por IP o CIDR del cliente |
| `XForwardedRemoteAddr` | `XForwardedRemoteAddr=192.168.1.0/24` | Como RemoteAddr pero lee `X-Forwarded-For` (cuando hay proxy/LB delante) |
| `Weight` | `Weight=group1, 80` | Porcentaje de tráfico del grupo (A/B testing) |

> En YAML no existe sintaxis para OR. Si se necesita lógica OR, la opción más limpia es declarar dos rutas separadas que apunten al mismo `uri`.

```yaml
# Equivalente en YAML usando dos rutas (OR implícito)
routes:
  - id: pedidos-desde-red-interna
    uri: lb://pedidos-service
    predicates:
      - Path=/api/pedidos/**
      - RemoteAddr=10.0.0.0/8

  - id: pedidos-con-api-key
    uri: lb://pedidos-service
    predicates:
      - Path=/api/pedidos/**
      - Header=X-Internal-Token, .+
```

```yaml
# Ejemplo: Weight para canary deployment — 80% a v1, 20% a v2
spring:
  cloud:
    gateway:
      routes:
        - id: servicio-v1
          uri: lb://servicio-v1
          predicates:
            - Path=/api/**
            - Weight=servicio-group, 80   # 80% del tráfico

        - id: servicio-v2
          uri: lb://servicio-v2
          predicates:
            - Path=/api/**
            - Weight=servicio-group, 20   # 20% del tráfico (canary)
```

### Filter

Un **Filter** transforma la petición antes de enviarla al servicio (**pre**) o la respuesta antes de devolverla al cliente (**post**):

| Filter | Tipo | Descripción |
|---|---|---|
| `AddRequestHeader` | Pre | Añade header a la petición |
| `AddResponseHeader` | Post | Añade header a la respuesta |
| `SetRequestHeader` | Pre | Reemplaza un header (si existe, lo sobreescribe) |
| `SetResponseHeader` | Post | Reemplaza un header en la respuesta |
| `RemoveRequestHeader` | Pre | Elimina un header de la petición |
| `RemoveResponseHeader` | Post | Elimina un header de la respuesta |
| `AddRequestHeadersIfNotPresent` | Pre | Añade headers solo si no existen ya |
| `MapRequestHeader` | Pre | Copia el valor de un header a otro nombre |
| `DedupeResponseHeader` | Post | Elimina duplicados de un header |
| `RewriteResponseHeader` | Post | Modifica el valor de un header de respuesta con regex |
| `RewriteLocationResponseHeader` | Post | Reescribe el header `Location` en respuestas de redirección |
| `RewritePath` | Pre | Reescribe la URL con regex |
| `StripPrefix` | Pre | Elimina segmentos del path |
| `SetPath` | Pre | Reemplaza el path completo |
| `CircuitBreaker` | Pre | Aplica circuit breaker |
| `RequestRateLimiter` | Pre | Limita peticiones |
| `Retry` | Pre | Reintenta en caso de error |
| `SetStatus` | Post | Cambia el HTTP status code |
| `RedirectTo` | Pre | Redirige a otra URL |
| `RequestSize` | Pre | Rechaza peticiones que superen un tamaño máximo |
| `SecureHeaders` | Post | Añade headers de seguridad estándar (HSTS, CSP, etc.) |
| `SaveSession` | Pre | Fuerza guardar la sesión antes de enrutar |
| `ModifyRequestBody` | Pre | Modifica el body de la petición (solo Java DSL) |
| `ModifyResponseBody` | Post | Modifica el body de la respuesta (solo Java DSL) |

---

## 6.4 Configuración de rutas (YAML y Java DSL)

### Configuración en YAML (más común)

YAML es la forma recomendada para la mayoría de proyectos: es declarativa, vive en el repositorio Git junto con el resto de la configuración del servicio, y puede ser actualizada sin tocar código Java. El Java DSL se reserva para los casos que YAML no puede expresar: lógica condicional en tiempo de arranque (rutas que se crean o no según variables de entorno), transformaciones de body (`ModifyRequestBody`/`ModifyResponseBody`), y predicates con lógica OR que requieren encadenamiento de lambdas. Para el 95% de los proyectos, YAML es suficiente.

```yaml
spring:
  application:
    name: gateway-service

  cloud:
    gateway:
      routes:
        # Ruta para el servicio de pedidos
        - id: pedidos-route
          uri: lb://pedidos-service       # lb:// = usa LoadBalancer (Eureka/Consul)
                                          # También puede ser una URL fija: https://api.externo.com
          predicates:
            - Path=/api/pedidos/**
          filters:
            - StripPrefix=1               # elimina /api del path antes de enrutar
            # /api/pedidos/42 → /pedidos/42

        # Ruta para el servicio de productos con múltiples predicates
        - id: productos-route
          uri: lb://productos-service
          predicates:
            - Path=/api/productos/**
            - Method=GET                  # solo peticiones GET
            - Header=X-API-Version, v2    # solo si el header tiene ese valor
          filters:
            - StripPrefix=1
            - AddRequestHeader=X-Gateway-Source, spring-cloud-gateway
            - AddResponseHeader=X-Response-From, gateway

        # Ruta con URL directa (sin Eureka)
        - id: servicio-externo
          uri: https://api.externo.com
          predicates:
            - Path=/externo/**
          filters:
            - RewritePath=/externo/(?<segment>.*), /${segment}

        # Ruta con ventana temporal (solo activa en fechas concretas)
        - id: promo-navidad
          uri: lb://promo-service
          predicates:
            - Path=/promo/**
            - Between=2024-12-01T00:00:00Z, 2024-12-31T23:59:59Z

        # Ruta restringida a IP interna
        - id: admin-route
          uri: lb://admin-service
          predicates:
            - Path=/admin/**
            - RemoteAddr=10.0.0.0/8       # solo desde red interna 10.x.x.x
```

### Configuración en Java DSL

El Java DSL es equivalente al YAML pero permite lógica dinámica (condicionales, bucles, inyección de dependencias):

```java
@Configuration
public class GatewayConfig {

    @Bean
    public RouteLocator customRoutes(RouteLocatorBuilder builder) {
        return builder.routes()
            .route("pedidos-route", r -> r
                .path("/api/pedidos/**")
                .filters(f -> f
                    .stripPrefix(1)
                    .addRequestHeader("X-Gateway-Source", "spring-cloud-gateway")
                    .circuitBreaker(config -> config
                        .setName("pedidosCircuitBreaker")
                        .setFallbackUri("forward:/fallback/pedidos"))
                )
                .uri("lb://pedidos-service")
            )
            .route("productos-route", r -> r
                .path("/api/productos/**")
                .and().method(HttpMethod.GET)
                .filters(f -> f
                    .stripPrefix(1)
                    .modifyResponseBody(String.class, String.class,
                        (exchange, body) -> Mono.just(body.toUpperCase()))  // solo disponible en DSL
                )
                .uri("lb://productos-service")
            )
            .build();
    }
}
```

> **[ADVERTENCIA]** `ModifyRequestBody` y `ModifyResponseBody` solo están disponibles en Java DSL, no en YAML. Para modificar el body en YAML hay que implementar un `GatewayFilter` personalizado.

### Orden de rutas y regla del primer match

Las rutas se evalúan **en el orden en que están declaradas**. La primera ruta cuyo predicate coincide con la petición es la que se aplica; el resto no se evalúan:

```yaml
spring:
  cloud:
    gateway:
      routes:
        # ORDEN INCORRECTO — la ruta genérica bloquea a la específica
        - id: ruta-generica
          uri: lb://servicio-general
          predicates:
            - Path=/api/**            # coincide con /api/admin/usuarios también

        - id: ruta-admin              # NUNCA se alcanza para /api/admin/**
          uri: lb://admin-service
          predicates:
            - Path=/api/admin/**

        # ORDEN CORRECTO — las rutas más específicas van primero
        - id: ruta-admin
          uri: lb://admin-service
          predicates:
            - Path=/api/admin/**      # se evalúa primero; si coincide, termina

        - id: ruta-generica
          uri: lb://servicio-general
          predicates:
            - Path=/api/**            # fallback para el resto de /api/
```

> **[ADVERTENCIA]** Este es uno de los errores más frecuentes en la configuración del Gateway: una ruta genérica declarada antes que una específica hace que la específica nunca se aplique, sin ningún mensaje de error.

El campo `order` permite fijar explícitamente la prioridad sin depender del orden de declaración en el YAML (menor número = mayor prioridad):

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: ruta-generica
          uri: lb://servicio-general
          predicates:
            - Path=/api/**
          order: 10            # se evalúa después

        - id: ruta-admin
          uri: lb://admin-service
          predicates:
            - Path=/api/admin/**
          order: 1             # se evalúa primero, independientemente de la posición en el YAML
```

> Si no se declara `order`, las rutas se evalúan en el orden de aparición en el YAML. Con `order` explícito, el orden de declaración no importa.

### URI `forward:` — reenvío interno al propio Gateway

Cuando un Circuit Breaker detecta que un microservicio está fallando, necesita devolver alguna respuesta al cliente en lugar de un error genérico. La opción de redirigir a otro microservicio crea una dependencia cruzada que complica el diagnóstico. La solución es que el propio Gateway tenga un controller de fallback y la URI de fallback apunte a ese controller con el prefijo `forward:`. El flujo de la petición nunca sale del proceso del Gateway, lo que garantiza que el fallback funciona aunque todos los microservicios estén caídos.

Además de `lb://` (LoadBalancer) y `https://` (URL directa), se puede usar `forward:` para reenviar la petición a un endpoint del propio Gateway (útil para fallbacks de Circuit Breaker):

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: pedidos-route
          uri: lb://pedidos-service
          predicates:
            - Path=/api/pedidos/**
          filters:
            - name: CircuitBreaker
              args:
                name: pedidosCircuitBreaker
                fallbackUri: forward:/fallback/pedidos   # redirige al controller de fallback del Gateway
```

```java
// Controller de fallback en el propio Gateway
@RestController
public class FallbackController {

    @GetMapping("/fallback/pedidos")
    public ResponseEntity<Map<String, String>> pedidosFallback() {
        return ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE)
            .body(Map.of("mensaje", "Servicio de pedidos temporalmente no disponible"));
    }
}
```

---

### Rutas dinámicas (desde base de datos u otra fuente)

Las rutas en YAML son estáticas: requieren reiniciar el Gateway para que un cambio sea efectivo. Esto es aceptable para arquitecturas estables, pero no para plataformas donde los equipos necesitan registrar nuevas rutas sin un deploy del Gateway. Un ejemplo típico es una plataforma de APIs interna donde cada equipo puede exponer su servicio a través del Gateway sin intervención del equipo de plataforma. La solución es implementar `RouteDefinitionLocator`, que lee las definiciones de rutas desde una base de datos (o cualquier otra fuente), y usar el endpoint `POST /actuator/gateway/refresh` para recargar las rutas en caliente sin reiniciar el proceso.

```java
// Implementar RouteDefinitionLocator para cargar rutas desde una fuente dinámica
@Component
public class DatabaseRouteDefinitionLocator implements RouteDefinitionLocator {

    private final RouteRepository routeRepository;

    @Override
    public Flux<RouteDefinition> getRouteDefinitions() {
        return routeRepository.findAll()
            .map(this::toRouteDefinition);
    }
}
```

```yaml
# Forzar recarga de rutas dinámicas sin reiniciar:
# POST /actuator/gateway/refresh
management:
  endpoints:
    web:
      exposure:
        include: gateway
```

---

← [Concepto y Arquitectura](./06-01-gateway-concepto.md) | [Volver al índice](./README.md) | Siguiente: [Filtros →](./06-03-gateway-filtros.md)
