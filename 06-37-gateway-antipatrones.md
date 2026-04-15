# 6.8.5 Antipatrones del Gateway programático

← [6.8.4 Migración Zuul](./06-36-gateway-migracion-zuul.md) | [Índice](./README.md) | [9 — Trazabilidad y Observabilidad →](./09-observabilidad.md)

---

La extensión programática del Gateway —filtros con acceso a repositorios, rutas dinámicas desde base de datos, lógica de negocio por petición— concentra los errores de diseño más difíciles de detectar en producción. A diferencia de un bug de lógica normal, los antipatrones reactivos y de ciclo de vida suelen pasar inadvertidos en tests locales y sólo se manifiestan bajo carga o cuando se reinicia el Gateway con estado inconsistente.

## Antipatrón 1: operación bloqueante en el event loop

> [ADVERTENCIA] Llamar a cualquier API síncrona bloqueante (JDBC, `RestTemplate`, `Thread.sleep()`, bloqueo de un `CompletableFuture`) desde dentro de un `GlobalFilter.filter()` o `RouteDefinitionLocator.getRouteDefinitions()` bloquea el hilo del event loop de Netty. Con un pool de 8 threads de event loop, basta con que 8 peticiones concurrentes ejecuten un bloqueo de 50 ms para que el Gateway deje de responder a todas las demás peticiones hasta que esos bloqueos liberen el hilo.

Manifestación en producción: el Gateway empieza a acumular peticiones en cola, el tiempo de respuesta p99 crece de forma sostenida, y los logs muestran `io.netty.util.internal.OutOfDirectMemoryError` o `reactor.netty.internal.shaded.reactor.pool.PoolAcquireTimeoutException`.

**Código problemático:**

```java
@Override
public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
    String apiKey = exchange.getRequest().getHeaders().getFirst("X-Api-Key");

    // MAL: acceso JDBC síncrono dentro del event loop
    Optional<Plan> plan = planJdbcRepository.findByApiKey(apiKey);

    if (plan.isEmpty()) {
        exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
        return exchange.getResponse().setComplete();
    }
    return chain.filter(exchange);
}
```

**Corrección:**

```java
@Override
public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
    String apiKey = exchange.getRequest().getHeaders().getFirst("X-Api-Key");

    // BIEN: repositorio R2DBC o Redis reactivo; nunca bloquea el event loop
    return planReactiveRepository.findByApiKey(apiKey)
        .flatMap(plan -> chain.filter(exchange))
        .switchIfEmpty(Mono.defer(() -> {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }));
}
```

Si es inevitable usar una API síncrona (p. ej. una librería sin versión reactiva), descargar la llamada a un scheduler de IO:

```java
return Mono.fromCallable(() -> planJdbcRepository.findByApiKey(apiKey))
    .subscribeOn(Schedulers.boundedElastic())   // thread pool separado del event loop
    .flatMap(opt -> opt.isPresent()
        ? chain.filter(exchange)
        : Mono.defer(() -> {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }));
```

## Antipatrón 2: RouteDefinitionLocator sin caché ni invalidación controlada

> [ADVERTENCIA] Un `RouteDefinitionLocator` que consulta la base de datos en cada llamada a `getRouteDefinitions()` puede ejecutar cientos de consultas por segundo, ya que `CachingRouteLocator` llama al método durante el proceso de resolución de cada petición entrante cuando su caché interna está vacía. Si además se publica `RefreshRoutesEvent` con demasiada frecuencia (por ejemplo, en respuesta a cada escritura individual en una operación batch), se produce un ciclo de invalidación-recarga que mantiene la caché vacía de forma permanente.

**Código problemático:**

```java
// MAL: consulta a BD en cada resolución de rutas, sin ningún control de frecuencia
@Override
public Flux<RouteDefinition> getRouteDefinitions() {
    return routeRepository.findAll()
        .map(this::toRouteDefinition);
    // Sin caché, sin filtro de enabled, sin control de errores
}
```

```java
// MAL: RefreshRoutesEvent por cada ruta en un batch de 50 rutas
routes.forEach(route -> {
    routeRepository.save(route).subscribe();
    publisher.publishEvent(new RefreshRoutesEvent(this)); // 50 eventos
});
```

**Corrección:**

```java
// BIEN: la caché de CachingRouteLocator funciona sola cuando no se publica
// RefreshRoutesEvent innecesariamente. Publicar solo al final de la operación batch.

@Override
public Flux<RouteDefinition> getRouteDefinitions() {
    return routeRepository.findByEnabledTrue()   // solo rutas activas
        .map(this::toRouteDefinition)
        .onErrorResume(ex -> {
            // Si la BD falla, devolver vacío en lugar de propagar el error
            // para no romper el Gateway; registrar la incidencia
            log.error("Error leyendo rutas de BD; manteniendo caché actual", ex);
            return Flux.empty();
        });
}

// En el servicio de escritura batch: un solo evento al final
public Mono<Void> saveAll(List<RouteDefinition> routes) {
    return Flux.fromIterable(routes)
        .flatMap(def -> routeRepository.save(toEntity(def)))
        .then(Mono.fromRunnable(
            () -> publisher.publishEvent(new RefreshRoutesEvent(this))  // 1 solo evento
        ));
}
```

## Antipatrón 3: lógica de negocio compleja acumulada en GlobalFilter

> [ADVERTENCIA] Un único `GlobalFilter` que realiza autenticación JWT, consulta el plan de suscripción, verifica la lista de bloqueo, añade headers de trazabilidad y registra métricas se convierte en un punto de fallo único: un bug en cualquiera de esas responsabilidades rompe todas las peticiones. Además, el orden de ejecución de las responsabilidades queda oculto dentro del método `filter()`, lo que hace imposible probar cada responsabilidad de forma aislada.

**Código problemático:**

```java
@Component
public class MegaFilter implements GlobalFilter, Ordered {

    // 5 dependencias inyectadas
    private final JwtValidator jwtValidator;
    private final PlanRepository planRepository;
    private final BlocklistRepository blocklistRepository;
    private final MetricsRecorder metrics;
    private final AuditLogger audit;

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // 80 líneas de lógica mezclada: autenticación + cuota + bloqueo + métricas + auditoría
        // ...
    }
}
```

**Corrección — filtros pequeños y ordenados:**

```java
// Cada filtro tiene una responsabilidad y un orden explícito

@Component
public class JwtValidationFilter implements GlobalFilter, Ordered {
    @Override public int getOrder() { return -300; }
    // Solo valida el JWT y añade X-User-Id / X-User-Roles
}

@Component
public class BlocklistFilter implements GlobalFilter, Ordered {
    @Override public int getOrder() { return -200; }
    // Solo consulta la blocklist por IP o user-id
}

@Component
public class SubscriptionPlanFilter implements GlobalFilter, Ordered {
    @Override public int getOrder() { return -100; }
    // Solo resuelve el plan y añade X-Subscription-Plan
}

@Component
public class AuditFilter implements GlobalFilter, Ordered {
    @Override public int getOrder() { return Ordered.LOWEST_PRECEDENCE; }
    // Solo registra la petición y respuesta para auditoría (en el then())
}
```

```
Orden de ejecución resultante (de menor a mayor Order):
  -300: JwtValidationFilter      → autentica
  -200: BlocklistFilter          → bloquea IPs/usuarios vetados
  -100: SubscriptionPlanFilter   → resuelve cuota
     0: (rate limiter integrado)
  MAX: AuditFilter               → registra resultado
```

> [CONCEPTO] El principio de responsabilidad única aplicado a `GlobalFilter` produce filtros que se pueden activar o desactivar individualmente con `@Profile` o `@ConditionalOnProperty`, probar de forma aislada inyectando mocks de sus dependencias, y ordenar con precisión sin que un cambio en una responsabilidad afecte a las demás.

## Tabla resumen: enfoque estándar vs extensión programática

No todo caso de uso requiere código Java: la mayoría de los escenarios habituales se resuelven con YAML y los filtros predefinidos de Gateway. La extensión programática solo es necesaria cuando la lógica depende de datos en tiempo de ejecución, de rutas dinámicas o de la migración de filtros Zuul existentes.

| Caso de uso | Estándar suficiente | Requiere extensión programática |
|---|---|---|
| Reenviar `/api/v1/**` a un microservicio | YAML con `Path` predicate | No |
| Añadir un header fijo a todas las peticiones | `default-filters: AddRequestHeader` | No |
| Validar JWT con clave pública JWKS | `spring-security-oauth2-resource-server` | No |
| Límite de peticiones por IP con Redis | `RequestRateLimiter` filter integrado | No |
| Rutas que cambian sin redeploy | — | `RouteDefinitionLocator` + `RouteDefinitionWriter` |
| Lógica de acceso que consulta BD por petición | — | `GlobalFilter` con repositorio reactivo |
| Añadir rutas desde una API REST de administración | — | `RouteDefinitionWriter` + REST controller |
| Migrar un `ZuulFilter` PRE/POST existente | — | `GlobalFilter` con `then()` para POST |
| Rate limiting por plan de suscripción (no por IP) | — | `KeyResolver` personalizado + `GlobalFilter` |

---

← [6.8.4 Migración Zuul](./06-36-gateway-migracion-zuul.md) | [Índice](./README.md) | [9 — Trazabilidad y Observabilidad →](./09-observabilidad.md)
