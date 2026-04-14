# 6.8 Extensión Avanzada: RouteLocator y Filtros Programáticos

← [6.7 Testing del Gateway](./06-07-gateway-testing.md) | [Índice](./README.md) | [7 — Circuit Breaker →](./07-circuit-breaker.md)

---

## Límites del enfoque estándar

El YAML + filtros predefinidos + `AbstractGatewayFilterFactory` cubre el 95 % de los proyectos. Pero hay escenarios donde esa capa no alcanza: las rutas no están definidas en tiempo de despliegue sino que se crean y eliminan dinámicamente (plataformas multitenant, API marketplaces), la lógica de enrutamiento depende de datos en una base de datos que cambia en runtime, o los filtros necesitan acceder a beans de Spring con lógica de negocio compleja que no puede expresarse en una clase de configuración estática.

| Técnica / Concepto | Para qué sirve |
|---|---|
| `RouteDefinitionLocator` | Cargar definiciones de rutas desde BD u otra fuente dinámica |
| `RouteDefinitionWriter` | Crear, actualizar y eliminar rutas en runtime sin reiniciar |
| `AbstractGatewayFilterFactory` avanzado | Filtros con inyección de dependencias complejas, acceso a repositorios |
| `RoutePredicateHandlerMapping` personalizado | Lógica de resolución de ruta completamente custom |

---

## 6.8.1 RouteDefinitionLocator — rutas dinámicas desde base de datos

Las rutas en YAML son estáticas: requieren reiniciar el Gateway para que un cambio sea efectivo. En una plataforma donde cada equipo puede registrar su servicio a través del Gateway sin un deploy del equipo de plataforma, la fuente de rutas debe ser una base de datos. `RouteDefinitionLocator` lee las definiciones de rutas desde cualquier fuente reactiva y el endpoint `POST /actuator/gateway/refresh` las recarga en caliente.

```java
import org.springframework.cloud.gateway.route.RouteDefinition;
import org.springframework.cloud.gateway.route.RouteDefinitionLocator;
import org.springframework.cloud.gateway.handler.predicate.PredicateDefinition;
import org.springframework.cloud.gateway.filter.FilterDefinition;
import org.springframework.stereotype.Component;
import org.springframework.beans.factory.annotation.Autowired;
import reactor.core.publisher.Flux;
import java.net.URI;
import java.util.List;
import java.util.Map;

@Component
public class DatabaseRouteDefinitionLocator implements RouteDefinitionLocator {

    @Autowired
    private RouteRepository routeRepository;   // repositorio Spring Data R2DBC o Mongo reactivo

    @Override
    public Flux<RouteDefinition> getRouteDefinitions() {
        return routeRepository.findAllActive()
            .map(this::toRouteDefinition);
    }

    private RouteDefinition toRouteDefinition(RouteEntity entity) {
        RouteDefinition def = new RouteDefinition();
        def.setId(entity.getId());
        def.setUri(URI.create(entity.getUri()));
        def.setOrder(entity.getOrden());

        // Predicate de Path
        PredicateDefinition pathPredicate = new PredicateDefinition();
        pathPredicate.setName("Path");
        pathPredicate.setArgs(Map.of("pattern", entity.getPath()));
        def.setPredicates(List.of(pathPredicate));

        // Filtro StripPrefix
        FilterDefinition stripFilter = new FilterDefinition();
        stripFilter.setName("StripPrefix");
        stripFilter.setArgs(Map.of("parts", "1"));
        def.setFilters(List.of(stripFilter));

        return def;
    }
}
```

```java
// Entidad de la base de datos
public class RouteEntity {
    private String id;
    private String uri;       // "lb://servicio-nombre"
    private String path;      // "/api/servicio/**"
    private int orden;
    private boolean activa;
    // getters y setters
}
```

```yaml
# Exponer el endpoint de recarga de rutas
management:
  endpoints:
    web:
      exposure:
        include: gateway
# POST /actuator/gateway/refresh → recarga todas las rutas sin reiniciar el proceso
```

> [ADVERTENCIA] `RouteDefinitionLocator` se llama en cada ciclo de refresco, no en cada petición. Si la base de datos es lenta, el refresco tarda y durante ese tiempo el Gateway sigue usando las rutas anteriores. Añadir un caché reactivo (`.cache(Duration.ofSeconds(30))`) para evitar consultas frecuentes a la BD en sistemas con miles de rutas.

---

## 6.8.2 RouteDefinitionWriter — CRUD de rutas en runtime

`RouteDefinitionWriter` es el complemento de `RouteDefinitionLocator`: permite crear, actualizar y eliminar rutas en runtime vía API, sin necesidad de reiniciar el Gateway ni modificar la base de datos directamente.

```java
import org.springframework.cloud.gateway.route.RouteDefinition;
import org.springframework.cloud.gateway.route.RouteDefinitionWriter;
import org.springframework.stereotype.Service;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.cloud.gateway.event.RefreshRoutesEvent;
import reactor.core.publisher.Mono;

@Service
public class GatewayAdminService {

    @Autowired
    private RouteDefinitionWriter routeDefinitionWriter;

    @Autowired
    private ApplicationEventPublisher publisher;

    public Mono<Void> crearRuta(RouteDefinition definition) {
        return routeDefinitionWriter.save(Mono.just(definition))
            .then(Mono.fromRunnable(() ->
                // Publicar evento para que el Gateway recargue las rutas inmediatamente
                publisher.publishEvent(new RefreshRoutesEvent(this))
            ));
    }

    public Mono<Void> eliminarRuta(String routeId) {
        return routeDefinitionWriter.delete(Mono.just(routeId))
            .then(Mono.fromRunnable(() ->
                publisher.publishEvent(new RefreshRoutesEvent(this))
            ));
    }
}
```

```java
// Endpoint de administración expuesto en el propio Gateway
import org.springframework.web.bind.annotation.*;
import reactor.core.publisher.Mono;

@RestController
@RequestMapping("/gateway-admin/routes")
public class GatewayAdminController {

    @Autowired
    private GatewayAdminService adminService;

    @PostMapping
    public Mono<Void> crearRuta(@RequestBody RouteDefinition definition) {
        return adminService.crearRuta(definition);
    }

    @DeleteMapping("/{id}")
    public Mono<Void> eliminarRuta(@PathVariable String id) {
        return adminService.eliminarRuta(id);
    }
}
```

> [ADVERTENCIA] `RouteDefinitionWriter` opera sobre el almacén en memoria del Gateway, no sobre la base de datos del `RouteDefinitionLocator` personalizado. Si el Gateway se reinicia, las rutas creadas con `RouteDefinitionWriter` se pierden a menos que también se persistan en la BD. Siempre persistir en BD **antes** de llamar a `routeDefinitionWriter.save()`.

---

## 6.8.3 GlobalFilter con acceso a repositorio — lógica de negocio compleja

Los filtros creados con `AbstractGatewayFilterFactory` reciben su configuración desde YAML en tiempo de arranque. Cuando la lógica requiere acceso a datos que cambian en runtime (plan de suscripción del usuario, lista de IPs bloqueadas actualizada dinámicamente, cuotas por API key desde BD), el filtro necesita inyectar beans de Spring con acceso a repositorios reactivos.

```java
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.core.Ordered;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

@Component
public class ApiKeyPlanFilter implements GlobalFilter, Ordered {

    @Autowired
    private ApiKeyRepository apiKeyRepository;   // repositorio R2DBC o Mongo reactivo

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String apiKey = exchange.getRequest().getHeaders().getFirst("X-API-Key");

        if (apiKey == null) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }

        // Consulta reactiva a la BD — no bloquea el hilo
        return apiKeyRepository.findByKey(apiKey)
            .flatMap(plan -> {
                if (!plan.isActivo()) {
                    exchange.getResponse().setStatusCode(HttpStatus.FORBIDDEN);
                    return exchange.getResponse().setComplete();
                }
                // Propagar el plan al microservicio para que aplique lógica de negocio
                ServerWebExchange mutated = exchange.mutate()
                    .request(r -> r.header("X-Plan-Id", plan.getId())
                                   .header("X-Plan-Limite", String.valueOf(plan.getLimiteRpm())))
                    .build();
                return chain.filter(mutated);
            })
            .switchIfEmpty(Mono.defer(() -> {
                exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
                return exchange.getResponse().setComplete();
            }));
    }

    @Override
    public int getOrder() {
        return -150;   // después del JwtAuthFilter (-200), antes de los GatewayFilters
    }
}
```

> [ADVERTENCIA] Las consultas reactivas a la base de datos en un `GlobalFilter` añaden latencia a **todas** las peticiones. Añadir caché reactivo con Caffeine o Redis para las consultas más frecuentes (como la validación de API keys que se repite en cada petición del mismo cliente).

---

## 6.8.4 Caché reactivo para `RouteDefinitionLocator`

Cuando el `RouteDefinitionLocator` consulta una BD o una API externa, el resultado debe cachearse para no añadir latencia a cada resolución de ruta. Reactor Core ofrece el operador `.cache(Duration)` que convierte cualquier `Flux` en un `Flux` con caché temporal:

```java
import org.springframework.cloud.gateway.route.RouteDefinition;
import org.springframework.cloud.gateway.route.RouteDefinitionLocator;
import org.springframework.stereotype.Component;
import org.springframework.beans.factory.annotation.Autowired;
import reactor.core.publisher.Flux;
import java.time.Duration;

@Component
public class CachedDatabaseRouteDefinitionLocator implements RouteDefinitionLocator {

    @Autowired
    private RouteRepository routeRepository;

    // Flux con caché de 30 segundos: la BD solo se consulta una vez cada 30 s
    // durante ese tiempo todas las peticiones usan el resultado cacheado
    private final Flux<RouteDefinition> cachedRoutes = Flux.defer(
            () -> routeRepository.findAllActive().map(this::toRouteDefinition)
        )
        .cache(Duration.ofSeconds(30));

    @Override
    public Flux<RouteDefinition> getRouteDefinitions() {
        return cachedRoutes;
    }

    private RouteDefinition toRouteDefinition(RouteEntity entity) {
        RouteDefinition def = new RouteDefinition();
        def.setId(entity.getId());
        def.setUri(java.net.URI.create(entity.getUri()));
        // ... mismo mapeo que en 6.8.1
        return def;
    }
}
```

Para invalidar el caché cuando una ruta cambia en la BD, publicar un `RefreshRoutesEvent`:

```java
import org.springframework.cloud.gateway.event.RefreshRoutesEvent;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.stereotype.Service;
import org.springframework.beans.factory.annotation.Autowired;

@Service
public class RouteInvalidationService {

    @Autowired
    private ApplicationEventPublisher publisher;

    public void invalidarCache() {
        publisher.publishEvent(new RefreshRoutesEvent(this));
    }
}
```

---

## 6.8.5 Migración Zuul → Gateway: equivalencias de extensión

Para proyectos que migran de Zuul [LEGACY] a Spring Cloud Gateway, la equivalencia de los puntos de extensión es directa:

| Zuul [LEGACY] | Spring Cloud Gateway | Notas |
|---|---|---|
| `ZuulFilter` con `filterType=pre` | `GlobalFilter` o `GatewayFilter` (pre) | Mismo concepto, API reactiva |
| `ZuulFilter` con `filterType=post` | `GlobalFilter` o `GatewayFilter` (post con `.then()`) | La fase post se expresa con `chain.filter().then(...)` |
| `ZuulFilter.filterOrder()` | `getOrder()` en `Ordered` | Misma semántica: menor número = mayor prioridad en pre |
| `RequestContext.getCurrentContext()` | `ServerWebExchange.getAttributes()` | Mapa de atributos por petición |
| `ZuulProperties.ZuulRoute` | `RouteDefinition` | Estructura de datos de una ruta |
| `SimpleRouteLocator` | `RouteDefinitionLocator` | Fuente de rutas |

```java
// Zuul [LEGACY] — pre filter
public class MiZuulFilter extends ZuulFilter {
    @Override public String filterType() { return "pre"; }
    @Override public int filterOrder() { return 5; }
    @Override public boolean shouldFilter() { return true; }
    @Override public Object run() {
        RequestContext ctx = RequestContext.getCurrentContext();
        ctx.addZuulRequestHeader("X-Source", "gateway");
        return null;
    }
}

// Gateway equivalente
@Component
public class MiGlobalFilter implements GlobalFilter, Ordered {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest mutated = exchange.getRequest().mutate()
            .header("X-Source", "gateway")
            .build();
        return chain.filter(exchange.mutate().request(mutated).build());
    }
    @Override public int getOrder() { return 5; }
}
```

---

## Antipatrones

> [ADVERTENCIA] **Usar `.block()` dentro de un filtro reactivo.** El pool de hilos de Reactor/Netty es muy reducido (típicamente 2× número de CPUs). Bloquear uno de esos hilos con `.block()` detiene el procesamiento de todas las demás peticiones que ese hilo maneja, y bajo carga puede causar deadlock completo del Gateway. Toda operación de E/S dentro de un filtro debe ser reactiva: `.flatMap()`, `.switchIfEmpty()`, `.then()`.

> [ADVERTENCIA] **Registrar estado mutable en campos de instancia de un filtro.** Los filtros son singletons de Spring compartidos por todas las peticiones concurrentes. Un contador, mapa o lista declarado como campo de instancia sin sincronización es una condición de carrera garantizada. El estado por petición debe vivir en atributos del exchange: `exchange.getAttributes().put("clave", valor)`.

> [ADVERTENCIA] **Crear un `RouteDefinitionLocator` que consulte la BD sin caché.** `RouteDefinitionLocator.getRouteDefinitions()` se llama en cada ciclo de resolución de rutas. Sin caché, una BD con latencia de 5 ms añadida a cada petición equivale a un máximo teórico de 200 RPS por hilo de Netty. Cachear las definiciones con una TTL apropiada al ritmo de cambio de las rutas.

---

## Enfoque estándar vs Enfoque avanzado

| Caso de uso | Estándar suficiente | Requiere extensión |
|---|---|---|
| Enrutar 10-50 servicios con rutas estáticas | ✅ YAML + `spring.cloud.gateway.routes` | — |
| Añadir headers, reescribir paths, rate limiting | ✅ Filtros predefinidos en YAML | — |
| Logging o correlation ID global | ✅ `GlobalFilter` simple sin BD | — |
| Rutas que cambian sin reiniciar el Gateway | — | ✅ `RouteDefinitionLocator` + `RouteDefinitionWriter` |
| Throttling por plan de suscripción desde BD | — | ✅ `GlobalFilter` con `ApiKeyRepository` reactivo |
| Modificar body de petición/respuesta | — | ✅ Java DSL con `modifyRequestBody`/`modifyResponseBody` |
| Cientos de rutas creadas dinámicamente por tenants | — | ✅ `DatabaseRouteDefinitionLocator` con caché reactivo |
| Migrar lógica de `ZuulFilter` a Gateway | — | ✅ `GlobalFilter` / `GatewayFilter` con misma semántica |

---

← [6.7 Testing del Gateway](./06-07-gateway-testing.md) | [Índice](./README.md) | [7 — Circuit Breaker →](./07-circuit-breaker.md)


