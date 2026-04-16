# 3.9 Rutas dinámicas y recarga en caliente

← [3.8 CORS y TLS/HTTPS en Gateway](sc-gateway-cors-tls.md) | [Índice](README.md) | [3.10 WebSockets y Server-Sent Events](sc-gateway-websockets.md) →

---

## Introducción

Las rutas definidas en YAML o con `@Bean RouteLocator` son estáticas: se cargan en el arranque y no cambian sin reiniciar el proceso. En arquitecturas donde el enrutamiento debe adaptarse a cambios frecuentes (activación/desactivación de features, integración de nuevos microservicios, A/B routing por configuración), reiniciar el gateway introduce downtime y riesgo. `RouteDefinitionRepository` resuelve este problema proporcionando una abstracción para persistir y modificar definiciones de rutas en tiempo de ejecución: las rutas se pueden añadir, eliminar y recargar sin reiniciar el proceso. El desarrollador necesita gestionar rutas en tiempo de ejecución en Gateway para actualizar el enrutamiento sin reiniciar el proceso.

## Diagrama: arquitectura de rutas dinámicas

El siguiente diagrama muestra la relación entre `RouteDefinitionRepository`, `RouteDefinitionLocator` y el `RouteLocator` cacheado que se recarga.

```
  ┌────────────────────────────────────────────────────────────┐
  │  Fuentes de definiciones de rutas (RouteDefinitionLocator) │
  │                                                            │
  │  PropertiesRouteDefinitionLocator  ← YAML / propiedades   │
  │  InMemoryRouteDefinitionRepository ← memoria (default)     │
  │  CustomRouteDefinitionRepository   ← tu implementación     │
  │    (Redis, base de datos, Config Server, etc.)             │
  └────────────────────────┬───────────────────────────────────┘
                           │  Flux<RouteDefinition>
                           ▼
              CompositeRouteDefinitionLocator
                           │
                           ▼
              RouteDefinitionRouteLocator (convierte RouteDefinition → Route)
                           │
                           ▼
              CachingRouteLocator  ← caché de rutas resueltas
                           │
              ┌────────────┴────────────────┐
              │  RefreshRoutesEvent         │
              │  → invalida la caché        │
              │  → recarga desde todas      │
              │    las fuentes              │
              └─────────────────────────────┘
                           │
              POST /actuator/gateway/refresh
              también publica RefreshRoutesEvent
```

## Ejemplo central

El siguiente ejemplo implementa un controlador REST que permite añadir y eliminar rutas en tiempo de ejecución, usando `RouteDefinitionWriter` como punto de acceso al repositorio.

```java
package com.example.gateway;

import org.springframework.cloud.gateway.event.RefreshRoutesEvent;
import org.springframework.cloud.gateway.route.RouteDefinition;
import org.springframework.cloud.gateway.route.RouteDefinitionWriter;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import reactor.core.publisher.Mono;

import java.net.URI;

/**
 * Controlador REST para gestión dinámica de rutas.
 * Usa RouteDefinitionWriter para guardar/eliminar y
 * ApplicationEventPublisher para publicar RefreshRoutesEvent.
 */
@RestController
@RequestMapping("/admin/routes")
public class DynamicRouteController {

    private final RouteDefinitionWriter routeDefinitionWriter;
    private final ApplicationEventPublisher eventPublisher;

    public DynamicRouteController(
            RouteDefinitionWriter routeDefinitionWriter,
            ApplicationEventPublisher eventPublisher) {
        this.routeDefinitionWriter = routeDefinitionWriter;
        this.eventPublisher = eventPublisher;
    }

    /**
     * Añade una nueva ruta en tiempo de ejecución.
     * La ruta queda activa después de publicar RefreshRoutesEvent.
     *
     * Ejemplo de body:
     * {
     *   "id": "new-service-route",
     *   "uri": "lb://new-service",
     *   "predicates": [{"name": "Path", "args": {"pattern": "/api/new/**"}}],
     *   "filters": [{"name": "StripPrefix", "args": {"parts": "1"}}],
     *   "order": 0
     * }
     */
    @PostMapping
    public Mono<ResponseEntity<String>> addRoute(@RequestBody RouteDefinition routeDefinition) {
        return routeDefinitionWriter
            .save(Mono.just(routeDefinition))
            .then(Mono.fromRunnable(() -> eventPublisher.publishEvent(new RefreshRoutesEvent(this))))
            .then(Mono.just(ResponseEntity.ok("Ruta '" + routeDefinition.getId() + "' añadida")));
    }

    /**
     * Elimina una ruta por ID en tiempo de ejecución.
     */
    @DeleteMapping("/{routeId}")
    public Mono<ResponseEntity<String>> deleteRoute(@PathVariable String routeId) {
        return routeDefinitionWriter
            .delete(Mono.just(routeId))
            .then(Mono.fromRunnable(() -> eventPublisher.publishEvent(new RefreshRoutesEvent(this))))
            .then(Mono.just(ResponseEntity.ok("Ruta '" + routeId + "' eliminada")))
            .onErrorResume(ex -> Mono.just(
                ResponseEntity.notFound().<String>build()));
    }
}
```

### Implementación de RouteDefinitionRepository personalizado (con Redis)

La implementación por defecto `InMemoryRouteDefinitionRepository` pierde las rutas dinámicas al reiniciar. Para persistencia real se implementa `RouteDefinitionRepository`:

```java
package com.example.gateway;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.cloud.gateway.route.RouteDefinition;
import org.springframework.cloud.gateway.route.RouteDefinitionRepository;
import org.springframework.data.redis.core.ReactiveRedisTemplate;
import org.springframework.stereotype.Component;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

/**
 * RouteDefinitionRepository respaldado por Redis.
 * Las rutas dinámicas sobreviven al reinicio del gateway.
 */
@Component
public class RedisRouteDefinitionRepository implements RouteDefinitionRepository {

    private static final String ROUTES_KEY = "gateway:routes";

    private final ReactiveRedisTemplate<String, String> redisTemplate;
    private final ObjectMapper objectMapper;

    public RedisRouteDefinitionRepository(
            ReactiveRedisTemplate<String, String> redisTemplate,
            ObjectMapper objectMapper) {
        this.redisTemplate = redisTemplate;
        this.objectMapper = objectMapper;
    }

    @Override
    public Flux<RouteDefinition> getRouteDefinitions() {
        return redisTemplate.opsForHash()
            .values(ROUTES_KEY)
            .flatMap(json -> {
                try {
                    return Mono.just(objectMapper.readValue((String) json, RouteDefinition.class));
                } catch (Exception e) {
                    return Mono.empty();
                }
            });
    }

    @Override
    public Mono<Void> save(Mono<RouteDefinition> route) {
        return route.flatMap(r -> {
            try {
                String json = objectMapper.writeValueAsString(r);
                return redisTemplate.opsForHash()
                    .put(ROUTES_KEY, r.getId(), json)
                    .then();
            } catch (Exception e) {
                return Mono.error(e);
            }
        });
    }

    @Override
    public Mono<Void> delete(Mono<String> routeId) {
        return routeId.flatMap(id ->
            redisTemplate.opsForHash().remove(ROUTES_KEY, id).then());
    }
}
```

### Recarga vía endpoint Actuator (sin código adicional)

```bash
# Publicar RefreshRoutesEvent vía Actuator (recarga el CachingRouteLocator)
curl -X POST http://localhost:8080/actuator/gateway/refresh

# Verificar rutas activas tras la recarga
curl http://localhost:8080/actuator/gateway/routes | jq '.[].route_id'
```

```yaml
# application.yml — exponer el endpoint de gateway
management:
  endpoints:
    web:
      exposure:
        include: gateway, health
  endpoint:
    gateway:
      enabled: true
```

## Tabla de elementos clave

La siguiente tabla describe las interfaces y componentes clave de las rutas dinámicas.

| Componente | Responsabilidad |
|-----------|-----------------|
| `RouteDefinitionRepository` | Interfaz: `getRouteDefinitions()`, `save()`, `delete()` — fuente de definiciones persistente |
| `InMemoryRouteDefinitionRepository` | Implementación por defecto en memoria; se pierde al reiniciar |
| `RouteDefinitionWriter` | Interfaz de escritura (subconjunto de `RouteDefinitionRepository`); bean inyectable |
| `RefreshRoutesEvent` | Evento Spring que invalida el `CachingRouteLocator` y fuerza la recarga |
| `CachingRouteLocator` | Caché de `Route` resueltas; se invalida con `RefreshRoutesEvent` |
| `POST /actuator/gateway/refresh` | Endpoint que publica `RefreshRoutesEvent`; requiere endpoint gateway expuesto |

> [CONCEPTO] `RefreshRoutesEvent` es el mecanismo central: sin publicarlo después de `save()` o `delete()`, el `CachingRouteLocator` sigue sirviendo la versión anterior de las rutas aunque el repositorio ya tenga los cambios.

> [ADVERTENCIA] Coexistencia de rutas estáticas y dinámicas: las rutas de YAML provienen de `PropertiesRouteDefinitionLocator`; las dinámicas de `InMemoryRouteDefinitionRepository` (o el repositorio personalizado). Ambas contribuyen al mismo `CompositeRouteDefinitionLocator`. Una ruta dinámica con el mismo `id` que una estática reemplaza la estática tras el `RefreshRoutesEvent`. Esto puede causar comportamientos inesperados si los IDs no son únicos.

## Buenas y malas prácticas

**Hacer:**
- Publicar siempre `RefreshRoutesEvent` después de `save()` o `delete()`: sin el evento, el `CachingRouteLocator` no recarga y los cambios no son visibles hasta el próximo reinicio o evento explícito.
- Implementar `RouteDefinitionRepository` con respaldo persistente (Redis, base de datos) en producción: `InMemoryRouteDefinitionRepository` pierde todas las rutas dinámicas al reiniciar el gateway.
- Usar `POST /actuator/gateway/refresh` en pipelines de CI/CD para aplicar cambios de rutas sin reiniciar: es el mecanismo de integración más simple cuando las rutas se gestionan externamente.

**Evitar:**
- Usar rutas dinámicas para reemplazar toda la configuración de rutas estáticas: las rutas YAML son auditables en git y reproducibles; las dinámicas sin repositorio persistente son efímeras. Usar dinámicas solo para el subconjunto que necesita cambiar en tiempo de ejecución.
- Reutilizar IDs de rutas entre rutas estáticas (YAML) y dinámicas: si una ruta dinámica tiene el mismo ID que una estática, el comportamiento tras `RefreshRoutesEvent` depende del orden de contribución de los locators, que no está garantizado.

---

← [3.8 CORS y TLS/HTTPS en Gateway](sc-gateway-cors-tls.md) | [Índice](README.md) | [3.10 WebSockets y Server-Sent Events](sc-gateway-websockets.md) →
