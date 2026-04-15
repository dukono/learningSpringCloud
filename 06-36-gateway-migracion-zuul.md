# 6.8.4 Migración Zuul [LEGACY] → Gateway: equivalencias de extensión

← [6.8.3 GlobalFilter reactivo](./06-35-gateway-filter-reactivo.md) | [Índice](./README.md) | [6.8.5 Antipatrones →](./06-37-gateway-antipatrones.md)

---

Netflix Zuul 1 [LEGACY] fue el API Gateway estándar de Spring Cloud hasta que Spring Cloud Gateway lo sustituyó en 2019. Zuul 1 usa un modelo bloqueante basado en Servlet (hilos Java tradicionales), mientras que Spring Cloud Gateway es reactivo y no bloqueante. La arquitectura interna es completamente distinta, pero los conceptos —filtros pre, post, error, enrutamiento, contexto de la petición— tienen equivalencias directas que permiten migrar un proyecto Zuul a Gateway de forma sistemática.

> [ADVERTENCIA] Zuul 1 y Spring Cloud Gateway son incompatibles en el mismo módulo. `spring-cloud-starter-netflix-zuul` incluye `spring-boot-starter-web` (Servlet); `spring-cloud-starter-gateway` incluye WebFlux. Intentar añadir ambas dependencias provoca un fallo en el arranque con el error `Spring MVC found on classpath, which is incompatible with Spring Cloud Gateway`.

## Mapa de conceptos: Zuul → Gateway

Cada concepto de Zuul 1 tiene un equivalente directo en Spring Cloud Gateway, aunque la API cambia por completo al pasar de un modelo Servlet bloqueante a WebFlux reactivo. La tabla muestra las correspondencias para filtros, contexto de petición, propiedades de enrutamiento y operaciones de modificación de headers.

| Concepto en Zuul [LEGACY] | Equivalente en Spring Cloud Gateway |
|---|---|
| `ZuulFilter` (PRE) | `GlobalFilter` o `GatewayFilter` con lógica antes de `chain.filter()` |
| `ZuulFilter` (POST) | `GlobalFilter` con lógica en el operador `then()` tras `chain.filter()` |
| `ZuulFilter` (ERROR) | `@ControllerAdvice` + `DefaultErrorWebExceptionHandler` |
| `ZuulFilter` (ROUTE) | `RouteLocator` / `RouteDefinitionLocator` |
| `RequestContext` | `ServerWebExchange` y `exchange.getAttributes()` |
| `RequestContext.setSendZuulResponse(false)` | `exchange.getResponse().setComplete()` (cortar la cadena) |
| `RequestContext.setResponseStatusCode(code)` | `exchange.getResponse().setStatusCode(HttpStatus.XXX)` |
| `RequestContext.addZuulRequestHeader(k, v)` | `exchange.mutate().request(r -> r.header(k, v))` |
| `RequestContext.addZuulResponseHeader(k, v)` | `exchange.getResponse().getHeaders().add(k, v)` |
| `zuul.routes.<id>.path` | `predicates: - Path=<pattern>` |
| `zuul.routes.<id>.url` | `uri: http://host:port` |
| `zuul.routes.<id>.serviceId` | `uri: lb://nombre-servicio` |
| `zuul.prefix` | Filtro `PrefixPath` o `RewritePath` |
| `zuul.stripPrefix` | Filtro `StripPrefix` |
| `zuul.sensitiveHeaders` | Filtro `RemoveRequestHeader` |
| `zuul.retryable` | Filtro `Retry` |
| `@EnableZuulProxy` | Sin equivalente de anotación; el Gateway se habilita con la dependencia |

## Migración de un ZuulFilter PRE

Un filtro PRE de Zuul típico valida un header y rechaza la petición si no está presente.

**Zuul [LEGACY]:**

```java
import com.netflix.zuul.ZuulFilter;
import com.netflix.zuul.context.RequestContext;
import org.springframework.stereotype.Component;
import javax.servlet.http.HttpServletRequest;

@Component
public class ApiKeyPreFilter extends ZuulFilter {

    @Override
    public String filterType() { return "pre"; }

    @Override
    public int filterOrder() { return 1; }

    @Override
    public boolean shouldFilter() { return true; }

    @Override
    public Object run() {
        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletRequest request = ctx.getRequest();
        String apiKey = request.getHeader("X-Api-Key");

        if (apiKey == null || apiKey.isBlank()) {
            ctx.setSendZuulResponse(false);           // cortar la cadena
            ctx.setResponseStatusCode(401);
            ctx.setResponseBody("API key requerida");
        }
        return null;
    }
}
```

**Gateway equivalente:**

```java
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.core.Ordered;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

@Component
public class ApiKeyGlobalFilter implements GlobalFilter, Ordered {

    @Override
    public int getOrder() { return -200; }   // equivalente a filterOrder

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String apiKey = exchange.getRequest().getHeaders().getFirst("X-Api-Key");

        if (apiKey == null || apiKey.isBlank()) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();   // cortar la cadena
        }

        return chain.filter(exchange);   // continuar si el header está presente
    }
}
```

## Migración de un ZuulFilter POST

Un filtro POST de Zuul añade un header a la respuesta.

**Zuul [LEGACY]:**

```java
@Component
public class CorrelationPostFilter extends ZuulFilter {

    @Override
    public String filterType() { return "post"; }

    @Override
    public int filterOrder() { return 10; }

    @Override
    public boolean shouldFilter() { return true; }

    @Override
    public Object run() {
        RequestContext ctx = RequestContext.getCurrentContext();
        ctx.addZuulResponseHeader("X-Gateway-Version", "1.0");
        return null;
    }
}
```

**Gateway equivalente:**

```java
@Component
public class VersionResponseFilter implements GlobalFilter, Ordered {

    @Override
    public int getOrder() { return Ordered.LOWEST_PRECEDENCE; }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // La lógica POST va en el then() que se ejecuta DESPUÉS de chain.filter()
        return chain.filter(exchange)
            .then(Mono.fromRunnable(() ->
                exchange.getResponse().getHeaders().add("X-Gateway-Version", "1.0")
            ));
    }
}
```

> [CONCEPTO] En Zuul [LEGACY], los filtros PRE y POST son clases separadas con `filterType()` distinto. En Spring Cloud Gateway, un único `GlobalFilter` contiene la lógica PRE antes de `chain.filter(exchange)` y la lógica POST dentro del `.then()` que se ejecuta cuando la cadena completa ha terminado. Esto permite tener todo el ciclo pre/post de una funcionalidad en un único lugar.

## Migración de rutas YAML

**Zuul [LEGACY] (application.yml):**

```yaml
zuul:
  prefix: /api
  routes:
    productos:
      path: /productos/**
      serviceId: productos-service
      stripPrefix: false
    pedidos:
      path: /pedidos/**
      url: http://pedidos-host:8081
      sensitiveHeaders: Cookie,Authorization
```

**Gateway equivalente:**

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: productos
          uri: lb://productos-service
          predicates:
            - Path=/api/productos/**
          filters:
            - RewritePath=/api/(?<segment>.*), /${segment}

        - id: pedidos
          uri: http://pedidos-host:8081
          predicates:
            - Path=/api/pedidos/**
          filters:
            - RewritePath=/api/(?<segment>.*), /${segment}
            - RemoveRequestHeader=Cookie
            - RemoveRequestHeader=Authorization
```

> [ADVERTENCIA] En Zuul [LEGACY], `stripPrefix: false` conserva el prefijo de la ruta al reenviar al servicio. En Gateway, el comportamiento por defecto es también conservar el path completo: si la petición llega a `/api/productos/123`, el microservicio recibe `/api/productos/123`. Si el microservicio no tiene ese prefijo, usa `RewritePath` para eliminarlo, no `StripPrefix`, que elimina segmentos por número y no por patrón.

## Migración de la configuración de reintentos

**Zuul [LEGACY]:**

```yaml
zuul:
  retryable: true
ribbon:
  MaxAutoRetries: 2
  MaxAutoRetriesNextServer: 1
```

**Gateway equivalente:**

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: productos
          uri: lb://productos-service
          predicates:
            - Path=/api/productos/**
          filters:
            - name: Retry
              args:
                retries: 2
                statuses: BAD_GATEWAY, SERVICE_UNAVAILABLE
                methods: GET
                backoff:
                  firstBackoff: 50ms
                  maxBackoff: 500ms
                  factor: 2
```

## Tabla de equivalencias de propiedades globales

Las propiedades de `zuul.*` que controlan timeouts de conexión, pool de conexiones y patrones ignorados tienen equivalentes en `spring.cloud.gateway.httpclient.*` y en la configuración de rutas, aunque algunas no tienen traducción directa porque Gateway usa backpressure reactivo en lugar de semáforos.

| Propiedad Zuul [LEGACY] | Propiedad Gateway |
|---|---|
| `zuul.host.connect-timeout-millis` | `spring.cloud.gateway.httpclient.connect-timeout` |
| `zuul.host.socket-timeout-millis` | `spring.cloud.gateway.httpclient.response-timeout` |
| `zuul.max.host.connections` | `spring.cloud.gateway.httpclient.pool.max-connections` |
| `zuul.semaphore.max-semaphores` | No aplica (Gateway no usa semáforos; usa backpressure reactivo) |
| `zuul.ignored-patterns` | No hay equivalente global; usar `predicates` para excluir rutas |
| `zuul.ignored-services` | `spring.cloud.gateway.discovery.locator.enabled: false` + rutas explícitas |

## Buenas y malas prácticas

Hacer:
- Migrar un filtro Zuul a la vez y verificar el comportamiento con los tests existentes antes de pasar al siguiente. Dado que el modelo bloqueante y el reactivo tienen diferencias sutiles en el manejo de errores, una migración gradual permite detectar regresiones ruta a ruta.
- Usar los tests de integración con `WebTestClient` descritos en la sección 6.7 para validar que cada ruta migrada se comporta exactamente igual que la ruta Zuul original. `WebTestClient` es el equivalente del `MockMvc` que probablemente ya tenías en los tests de Zuul.

Evitar:
- No intentar portar código que use `RequestContext.getCurrentContext()` directamente a Gateway. `RequestContext` es una ThreadLocal específica de Zuul; en Gateway no existe. Todo el estado de la petición vive en `ServerWebExchange`, que se pasa explícitamente por la cadena de filtros.
- No conservar Zuul como capa intermedia mientras se añaden rutas nuevas en Gateway. Tener dos Gateways en paralelo duplica la latencia, complica el diagnóstico de problemas y crea confusión sobre qué sistema aplica qué política. La migración debe ser completa ruta a ruta, no coexistente.

---

← [6.8.3 GlobalFilter reactivo](./06-35-gateway-filter-reactivo.md) | [Índice](./README.md) | [6.8.5 Antipatrones →](./06-37-gateway-antipatrones.md)
