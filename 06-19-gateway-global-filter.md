# 6.4.2 GlobalFilter y Ordered: filtro global y exclusión de rutas públicas

← [6.4.1 AbstractGatewayFilterFactory](./06-18-gateway-filter-factory.md) | [Índice](./README.md) | [6.4.3 Filtros con lógica de negocio →](./06-20-gateway-filter-negocio.md)

---

Un `GlobalFilter` es un filtro que se aplica automáticamente a todas las rutas del Gateway, sin necesidad de declararlo en el YAML de cada ruta. Es el mecanismo adecuado para lógica transversal: correlación de peticiones con un identificador único, autenticación centralizada, métricas de latencia, logging de acceso o propagación de contexto de seguridad. La diferencia con los filtros de `AbstractGatewayFilterFactory` es de alcance: donde la factory crea filtros por ruta, el `GlobalFilter` actúa sobre todo el tráfico del Gateway.

## Diagrama: orden de ejecución de múltiples GlobalFilters

Cuando coexisten varios `GlobalFilter` en el mismo Gateway, el valor de `getOrder()` de cada uno determina su posición en el pipeline: los valores más bajos actúan primero sobre la petición entrante y últimos sobre la respuesta saliente. Los filtros de ruta se insertan entre los globales, después de que `RoutePredicateHandlerMapping` haya seleccionado la ruta.

```
Petición entrante
       │
       ▼
┌──────────────────────────────────────────────────────────┐
│              Pipeline del Gateway                        │
│                                                          │
│  GlobalFilter order=-2    ← CorrelationIdFilter          │
│  GlobalFilter order=-1    ← JwtValidationFilter          │
│  GlobalFilter order=0     ← MetricsFilter (defecto)      │
│                                                          │
│  ←── RoutePredicateHandlerMapping (selecciona ruta) ───  │
│                                                          │
│  GatewayFilter de ruta #1 ← StripPrefix                  │
│  GatewayFilter de ruta #2 ← AddRequestHeader             │
│                                                          │
│  GlobalFilter order=MAX-1 ← NettyRoutingFilter (llama    │
│                              al servicio destino)        │
└──────────────────────────────────────────────────────────┘
       │
       ▼
  Servicio destino
       │
       ▼
  Respuesta (recorre el pipeline en orden inverso)
```

Los filtros con `order` menor se ejecutan primero en el pre-filter y últimos en el post-filter. Los filtros built-in del framework tienen órdenes en el rango de `Integer.MIN_VALUE` a `Integer.MAX_VALUE`, reservando los extremos para routing y write de respuesta.

## Implementación completa: CorrelationIdFilter

`CorrelationIdFilter` garantiza que cada petición que atraviesa el Gateway lleve un identificador de correlación único. Si la petición ya incluye el header `X-Correlation-Id`, lo reutiliza; si no, genera un UUID nuevo. En ambos casos propaga el identificador al servicio destino en el pre-filter y lo devuelve al cliente en la respuesta mediante el post-filter.

```java
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.core.Ordered;
import org.springframework.http.server.reactive.ServerHttpRequest;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

import java.util.UUID;

@Component
public class CorrelationIdFilter implements GlobalFilter, Ordered {

    private static final String CORRELATION_ID_HEADER = "X-Correlation-Id";

    @Override
    public int getOrder() {
        // Valor negativo: ejecuta antes de los filtros de ruta (order 0)
        // y antes del routing al servicio destino
        return -1;
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String correlationId = exchange.getRequest().getHeaders()
            .getFirst(CORRELATION_ID_HEADER);

        if (correlationId == null || correlationId.isBlank()) {
            correlationId = UUID.randomUUID().toString();
        }

        final String finalCorrelationId = correlationId;

        // Añade el header a la petición hacia el servicio (usando mutate, ya que
        // ServerHttpRequest es inmutable)
        ServerHttpRequest mutatedRequest = exchange.getRequest().mutate()
            .header(CORRELATION_ID_HEADER, finalCorrelationId)
            .build();

        ServerWebExchange mutatedExchange = exchange.mutate()
            .request(mutatedRequest)
            .build();

        // Añade el header a la respuesta hacia el cliente en el post-filter
        return chain.filter(mutatedExchange).then(Mono.fromRunnable(() ->
            mutatedExchange.getResponse().getHeaders()
                .add(CORRELATION_ID_HEADER, finalCorrelationId)
        ));
    }
}
```

> [CONCEPTO] `ServerHttpRequest` y `ServerWebExchange` son inmutables en WebFlux. Para modificarlos se usa el patrón builder vía `.mutate()`. El `mutatedExchange` que se pasa a `chain.filter()` tiene el request modificado; el exchange original que recibió el filtro no se altera. Todos los filtros de la cadena que vengan después reciben el exchange mutado.

## Explicación de `Ordered` y rangos de prioridad

El valor devuelto por `getOrder()` determina la posición del filtro en el pipeline. Valores menores ejecutan antes en la entrada (pre-filter) y después en la salida (post-filter):

| Rango | Uso habitual |
|---|---|
| `Integer.MIN_VALUE` a `-100` | Filtros de sistema del framework. No usar en filtros personalizados |
| `-99` a `-1` | Filtros personalizados de infraestructura (correlación, autenticación) que deben ejecutarse antes del routing |
| `0` | Valor por defecto. Filtros de propósito general |
| `1` a `Integer.MAX_VALUE - 2` | Filtros post-routing, métricas de respuesta, transformación de respuesta |
| `Integer.MAX_VALUE - 1` | `NettyRoutingFilter` (hace la llamada real al servicio) |
| `Integer.MAX_VALUE` | `NettyWriteResponseFilter` (escribe la respuesta al cliente) |

> [EXAMEN] Un `GlobalFilter` con `getOrder()` mayor que `NettyRoutingFilter` (Integer.MAX_VALUE - 1) ejecuta su pre-filter DESPUÉS de que el servicio ya ha sido llamado. Para la mayoría de casos de pre-processing (auth, correlación), el orden debe ser menor que `-1` o como máximo `-1`.

> [ADVERTENCIA] El atributo `ServerWebExchangeUtils.GATEWAY_ROUTE_ATTR` solo está disponible en el exchange después de que `RoutePredicateHandlerMapping` haya seleccionado la ruta. Los filtros con `getOrder()` muy negativo (que se ejecutan antes del mapping) no pueden acceder a la ruta seleccionada porque aún no existe.

## Exclusión de rutas públicas en un GlobalFilter de autenticación

Los endpoints de health check, actuator o rutas públicas no deben pasar por la validación de autenticación. El patrón estándar es comprobar el path al inicio del filtro y saltar directamente al siguiente filtro si la ruta es pública:

```java
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.core.Ordered;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

import java.util.List;

@Component
public class JwtValidationFilter implements GlobalFilter, Ordered {

    // Paths que no requieren autenticación
    private static final List<String> PUBLIC_PATHS = List.of(
        "/actuator",
        "/health",
        "/public/"
    );

    @Override
    public int getOrder() {
        return -1;
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String path = exchange.getRequest().getURI().getPath();

        // Si el path es público, pasar al siguiente filtro sin validar
        boolean isPublic = PUBLIC_PATHS.stream().anyMatch(path::startsWith);
        if (isPublic) {
            return chain.filter(exchange);
        }

        // Validar el token JWT
        String authHeader = exchange.getRequest().getHeaders().getFirst("Authorization");
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }

        // Aquí iría la validación real del token (ver 6.6.2 para implementación completa)
        String token = authHeader.substring(7);
        if (!isValidToken(token)) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }

        return chain.filter(exchange);
    }

    private boolean isValidToken(String token) {
        // Implementación real en 06-25-gateway-jwt-filter.md
        return token != null && !token.isBlank();
    }
}
```

> [CONCEPTO] `exchange.getResponse().setComplete()` termina la respuesta sin llamar a ningún filtro más de la cadena ni al servicio destino. Devuelve la respuesta con el status que se haya establecido previamente (`UNAUTHORIZED` en el ejemplo). Es el patrón correcto para cortocircuitar el pipeline desde un GlobalFilter.

## Acceso a la ruta activa desde un GlobalFilter

Una vez que el `RoutePredicateHandlerMapping` ha seleccionado la ruta, su definición está disponible en el exchange como atributo:

```java
import org.springframework.cloud.gateway.route.Route;
import org.springframework.cloud.gateway.support.ServerWebExchangeUtils;

// Dentro del filter(), después de que RoutePredicateHandlerMapping haya actuado:
Route route = exchange.getAttribute(ServerWebExchangeUtils.GATEWAY_ROUTE_ATTR);
if (route != null) {
    String routeId = route.getId();
    // Metadatos definidos en la ruta YAML
    Object criticidad = route.getMetadata().get("nivel-criticidad");
}
```

Este acceso solo es válido en filtros con order mayor que el del `RoutePredicateHandlerMapping`. Para filtros con order muy negativo (que actúan antes del mapping), el atributo es null.

## Tabla de GlobalFilters built-in del framework

Spring Cloud Gateway registra automáticamente varios `GlobalFilter` internos que gestionan el routing, la resolución de balanceo de carga y la escritura de la respuesta. Conocer sus valores de `order` es imprescindible para posicionar correctamente los filtros personalizados sin interferir con el comportamiento central del framework.

| Filtro | Order | Responsabilidad |
|---|---|---|
| `AdaptCachedBodyGlobalFilter` | `Integer.MIN_VALUE + 1000` | Restaura el body cacheado por `CacheRequestBody` |
| `RouteToRequestUrlFilter` | `10000` | Construye la URL final del servicio a partir de la URI de la ruta |
| `LoadBalancerClientFilter` | `10150` | Resuelve `lb://nombre-servicio` contra el registro de servicios |
| `NettyRoutingFilter` | `Integer.MAX_VALUE - 1` | Ejecuta la llamada HTTP real al servicio destino mediante Netty |
| `NettyWriteResponseFilter` | `Integer.MAX_VALUE - 1` | Escribe el body de la respuesta del servicio al cliente |
| `ForwardRoutingFilter` | `Integer.MAX_VALUE - 1` | Procesa URIs `forward:///path` (reenvío interno) |

## Habilitar/deshabilitar un GlobalFilter con @ConditionalOnProperty

`@ConditionalOnProperty` permite controlar desde el fichero de propiedades si un `GlobalFilter` se registra como bean o no, sin modificar el código. El parámetro `matchIfMissing = true` hace que el filtro esté activo por defecto cuando la propiedad no está definida, de forma que deshabilitar el filtro es un acto explícito de configuración.

```java
@Component
@ConditionalOnProperty(
    name = "gateway.filters.correlation-id.enabled",
    havingValue = "true",
    matchIfMissing = true  // activo por defecto
)
public class CorrelationIdFilter implements GlobalFilter, Ordered {
    // ...
}
```

```yaml
# Para deshabilitar en tests o entornos específicos:
gateway:
  filters:
    correlation-id:
      enabled: false
```

## Buenas y malas prácticas

Hacer:
- Documentar el valor de `getOrder()` y el motivo de ese valor concreto. En equipos grandes, un filtro mal ordenado puede interferir con la autenticación o el routing de forma difícil de diagnosticar.
- Usar `Mono.error()` para propagar excepciones en la cadena reactiva. Lanzar excepciones síncronas dentro de un filtro (`throw new RuntimeException()`) puede escapar del contexto reactivo y producir comportamientos inesperados.
- Mantener cada `GlobalFilter` con una única responsabilidad. Un filtro que hace correlación, autenticación y métricas simultáneamente es difícil de testear y de deshabilitar selectivamente.
- Proteger rutas públicas explícitamente en la lista de exclusión y añadir un test que verifique que `/actuator/health` no requiere autenticación.

Evitar:
- Usar `getOrder() = 0` sin haberlo verificado contra los filtros existentes en el proyecto. Si hay varios filtros con `order = 0`, el orden de ejecución entre ellos no está garantizado.
- Hacer llamadas bloqueantes (JDBC, REST síncrono) dentro del filtro. Usar `WebClient` reactivo o `ReactiveRedisTemplate` en su lugar.
- Modificar la respuesta después de `chain.filter(exchange).then(...)` sin usar el operador `.then()` correctamente. El post-filter debe estar en el `then()` de la cadena:

```java
// Correcto: el post-filter está en .then()
return chain.filter(exchange).then(Mono.fromRunnable(() -> {
    // acceder a exchange.getResponse() aquí
}));

// Incorrecto: el código después de chain.filter() no es el post-filter
chain.filter(exchange);  // ¡no se devuelve el Mono!
return Mono.empty();     // el filtro termina antes de que el servicio responda
```

## Comparación: GlobalFilter vs AbstractGatewayFilterFactory

La diferencia fundamental entre ambos mecanismos es el alcance: `GlobalFilter` actúa sobre todo el tráfico del Gateway sin declaración en YAML, mientras que `AbstractGatewayFilterFactory` genera un filtro que solo opera en las rutas donde se declara explícitamente y acepta parámetros distintos por ruta. Esta tabla resume los criterios que determinan cuál usar en cada situación.

| Aspecto | GlobalFilter | AbstractGatewayFilterFactory |
|---|---|---|
| Alcance | Todas las rutas del Gateway | Solo rutas que lo declaran en YAML |
| Configuración por ruta | No (lógica uniforme para todas) | Sí, mediante clase `Config` con parámetros YAML |
| Declaración en YAML | No necesaria; es un bean de Spring | Obligatoria en la lista `filters` de la ruta |
| Control del orden | `getOrder()` preciso contra filtros del framework | Posición en la lista `filters` de la ruta |
| Casos de uso típicos | Correlación, autenticación, métricas globales | Transformaciones específicas con parámetros variables |
| Habilitación selectiva | `@ConditionalOnProperty` o lógica de exclusión interna | No declararlo en las rutas que no lo necesitan |

---

← [6.4.1 AbstractGatewayFilterFactory](./06-18-gateway-filter-factory.md) | [Índice](./README.md) | [6.4.3 Filtros con lógica de negocio →](./06-20-gateway-filter-negocio.md)
