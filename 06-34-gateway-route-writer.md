# 6.8.2 RouteDefinitionWriter: CRUD de rutas en runtime sin reiniciar

← [6.8.1 RouteDefinitionLocator](./06-33-gateway-route-locator.md) | [Índice](./README.md) | [6.8.3 GlobalFilter reactivo →](./06-35-gateway-filter-reactivo.md)

---

`RouteDefinitionWriter` es la interfaz complementaria a `RouteDefinitionLocator`: mientras el locator lee rutas, el writer las crea, modifica y elimina en tiempo de ejecución. Spring Cloud Gateway incluye una implementación en memoria (`InMemoryRouteDefinitionRepository`) y expone el endpoint `/actuator/gateway/routes` para consultar el estado actual. La implementación personalizada permite que las rutas escritas persistan en base de datos, de modo que sobrevivan a reinicios del Gateway y sean compartidas entre múltiples instancias.

## Arquitectura: ciclo completo de escritura y activación

Cada operación de escritura sigue el mismo patrón: la petición HTTP llega al `RouteController`, que delega en el `RouteDefinitionWriter` para persistir el cambio en base de datos y publica inmediatamente un `RefreshRoutesEvent` para que `CachingRouteLocator` invalide su caché y recargue las rutas activas sin reiniciar el Gateway.

```
ESCRITURA DE RUTA NUEVA:

  HTTP POST /admin/routes
    │  JSON con RouteDefinition
    ▼
  RouteController (REST)
    │  valida y guarda en BD
    ▼
  RouteDefinitionWriter.save(routeDefinition)
    │  persiste en gateway_routes
    ▼
  ApplicationEventPublisher.publishEvent(RefreshRoutesEvent)
    │
    ▼
  CachingRouteLocator invalida caché
    │
    ▼
  RouteDefinitionLocator.getRouteDefinitions()
    │  lee todas las rutas (incluyendo la nueva)
    ▼
  Nueva ruta activa en el Gateway

ELIMINACIÓN DE RUTA:

  HTTP DELETE /admin/routes/{routeId}
    │
    ▼
  RouteDefinitionWriter.delete(Mono.just(routeId))
    │  elimina de BD
    ▼
  RefreshRoutesEvent publicado
    │
    ▼
  Ruta desaparece del Gateway sin reiniciar
```

## Implementación: DatabaseRouteDefinitionWriter

`RouteDefinitionWriter` declara dos métodos reactivos: `save` recibe un `Mono<RouteDefinition>` y devuelve `Mono<Void>`; `delete` recibe un `Mono<String>` con el id y devuelve `Mono<Void>`.

```java
import org.springframework.cloud.gateway.route.RouteDefinition;
import org.springframework.cloud.gateway.route.RouteDefinitionWriter;
import org.springframework.cloud.gateway.event.RefreshRoutesEvent;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.stereotype.Component;
import reactor.core.publisher.Mono;

@Component
public class DatabaseRouteDefinitionWriter implements RouteDefinitionWriter {

    private final RouteDefinitionRepository routeRepository;
    private final ApplicationEventPublisher publisher;

    // ObjectMapper para serializar predicates y filters a JSON
    private final com.fasterxml.jackson.databind.ObjectMapper objectMapper;

    public DatabaseRouteDefinitionWriter(
            RouteDefinitionRepository routeRepository,
            ApplicationEventPublisher publisher,
            com.fasterxml.jackson.databind.ObjectMapper objectMapper) {
        this.routeRepository = routeRepository;
        this.publisher = publisher;
        this.objectMapper = objectMapper;
    }

    @Override
    public Mono<Void> save(Mono<RouteDefinition> routeMono) {
        return routeMono
            .flatMap(definition -> {
                RouteEntity entity = toEntity(definition);
                return routeRepository.save(entity);
            })
            .doOnSuccess(saved -> publisher.publishEvent(new RefreshRoutesEvent(this)))
            .then();
    }

    @Override
    public Mono<Void> delete(Mono<String> routeIdMono) {
        return routeIdMono
            .flatMap(routeId ->
                routeRepository.findByRouteId(routeId)
                    .switchIfEmpty(Mono.error(
                        new RouteNotFoundException("Ruta no encontrada: " + routeId)))
                    .flatMap(routeRepository::delete)
            )
            .doOnSuccess(ignored -> publisher.publishEvent(new RefreshRoutesEvent(this)))
            .then();
    }

    private RouteEntity toEntity(RouteDefinition definition) {
        RouteEntity entity = new RouteEntity();
        entity.setRouteId(definition.getId());
        entity.setUri(definition.getUri().toString());
        entity.setOrder(definition.getOrder());
        entity.setEnabled(true);

        try {
            entity.setPredicatesJson(
                objectMapper.writeValueAsString(definition.getPredicates()));
            entity.setFiltersJson(
                objectMapper.writeValueAsString(definition.getFilters()));
        } catch (Exception e) {
            throw new RuntimeException("Error serializando predicates/filters", e);
        }

        return entity;
    }
}

// Excepción para distinguir "no encontrado" de errores de infraestructura
class RouteNotFoundException extends RuntimeException {
    public RouteNotFoundException(String message) {
        super(message);
    }
}
```

## Repositorio con búsqueda por routeId

El repositorio R2DBC necesita el método `findByRouteId` para la operación de borrado por identificador lógico (no por id numérico de la tabla):

```java
import org.springframework.data.repository.reactive.ReactiveCrudRepository;
import reactor.core.publisher.Mono;

public interface RouteDefinitionRepository extends ReactiveCrudRepository<RouteEntity, Long> {

    Mono<RouteEntity> findByRouteId(String routeId);

    reactor.core.publisher.Flux<RouteEntity> findByEnabledTrue();
}
```

## REST Controller para gestión de rutas

El controller expone el CRUD de rutas sobre el `RouteDefinitionWriter`. La validación básica se hace antes de delegar en el writer para devolver errores descriptivos al cliente de la API:

```java
import org.springframework.cloud.gateway.event.RefreshRoutesEvent;
import org.springframework.cloud.gateway.route.RouteDefinition;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import reactor.core.publisher.Mono;

@RestController
@RequestMapping("/admin/routes")
public class RouteController {

    private final DatabaseRouteDefinitionWriter writer;
    private final RouteDefinitionRepository repository;
    private final ApplicationEventPublisher publisher;

    public RouteController(
            DatabaseRouteDefinitionWriter writer,
            RouteDefinitionRepository repository,
            ApplicationEventPublisher publisher) {
        this.writer = writer;
        this.repository = repository;
        this.publisher = publisher;
    }

    // Listar todas las rutas almacenadas
    @GetMapping
    public reactor.core.publisher.Flux<RouteEntity> listRoutes() {
        return repository.findAll();
    }

    // Crear o reemplazar una ruta
    @PostMapping
    public Mono<ResponseEntity<String>> createRoute(
            @RequestBody RouteDefinition routeDefinition) {

        if (routeDefinition.getId() == null || routeDefinition.getId().isBlank()) {
            return Mono.just(ResponseEntity
                .badRequest()
                .body("El campo 'id' es obligatorio"));
        }
        if (routeDefinition.getUri() == null) {
            return Mono.just(ResponseEntity
                .badRequest()
                .body("El campo 'uri' es obligatorio"));
        }

        return writer.save(Mono.just(routeDefinition))
            .then(Mono.just(ResponseEntity
                .status(HttpStatus.CREATED)
                .body("Ruta '" + routeDefinition.getId() + "' creada")))
            .onErrorResume(ex -> Mono.just(ResponseEntity
                .status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body("Error al crear la ruta: " + ex.getMessage())));
    }

    // Eliminar una ruta por su identificador
    @DeleteMapping("/{routeId}")
    public Mono<ResponseEntity<String>> deleteRoute(@PathVariable String routeId) {
        return writer.delete(Mono.just(routeId))
            .then(Mono.just(ResponseEntity.ok("Ruta '" + routeId + "' eliminada")))
            .onErrorResume(RouteNotFoundException.class, ex ->
                Mono.just(ResponseEntity.status(HttpStatus.NOT_FOUND).body(ex.getMessage())))
            .onErrorResume(ex -> Mono.just(ResponseEntity
                .status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body("Error al eliminar la ruta: " + ex.getMessage())));
    }

    // Activar o desactivar una ruta sin eliminarla
    @PatchMapping("/{routeId}/enabled")
    public Mono<ResponseEntity<String>> toggleRoute(
            @PathVariable String routeId,
            @RequestParam boolean enabled) {

        return repository.findByRouteId(routeId)
            .switchIfEmpty(Mono.error(new RouteNotFoundException("Ruta no encontrada: " + routeId)))
            .flatMap(entity -> {
                entity.setEnabled(enabled);
                return repository.save(entity);
            })
            .doOnSuccess(ignored -> publisher.publishEvent(new RefreshRoutesEvent(this)))
            .then(Mono.just(ResponseEntity.ok(
                "Ruta '" + routeId + "' " + (enabled ? "activada" : "desactivada"))))
            .onErrorResume(RouteNotFoundException.class, ex ->
                Mono.just(ResponseEntity.status(HttpStatus.NOT_FOUND).body(ex.getMessage())));
    }
}
```

> [CONCEPTO] En el método `toggleRoute`, el `RefreshRoutesEvent` debe publicarse tras guardar el cambio de `enabled` para que `CachingRouteLocator` recargue las rutas activas. En el código anterior se omite el publisher por brevedad; en un proyecto real se inyecta `ApplicationEventPublisher` directamente en el controller y se llama a `publisher.publishEvent(new RefreshRoutesEvent(this))` en el `doOnSuccess`.

## Parámetros del contrato REST

El `RouteController` expone cuatro operaciones sobre el path base `/admin/routes`: listado, creación, borrado por `routeId` y activación o desactivación sin eliminar la ruta. Las operaciones de escritura devuelven siempre `404` si el `routeId` no existe en la tabla, diferenciando este caso de los errores de infraestructura con códigos `5xx`.

| Operación | Método | Path | Body / Parámetro | Respuesta |
|---|---|---|---|---|
| Listar rutas | `GET` | `/admin/routes` | — | Array JSON de `RouteEntity` |
| Crear ruta | `POST` | `/admin/routes` | `RouteDefinition` JSON | `201 Created` |
| Eliminar ruta | `DELETE` | `/admin/routes/{routeId}` | — | `200 OK` / `404` |
| Activar/desactivar | `PATCH` | `/admin/routes/{routeId}/enabled?enabled=true` | — | `200 OK` / `404` |

## Seguridad del endpoint de administración

Los endpoints `/admin/routes` deben estar protegidos para impedir que actores externos inyecten rutas arbitrarias. La forma recomendada es restringir el acceso a una red interna mediante un `Header` predicate (como el `X-Internal-Token` visto en las rutas de admin de la sección 6.7.2) o mediante el `JwtAuthFilter` con un rol de administrador.

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: route-admin-api
          uri: lb://gateway-self   # ruta al propio Gateway (loop interno)
          predicates:
            - Path=/admin/routes/**
            - Header=X-Internal-Token, [a-f0-9]{40}
          filters:
            - RemoveRequestHeader=X-Internal-Token
```

## Buenas y malas prácticas

Hacer:
- Validar la `RouteDefinition` antes de delegarla al writer: verificar que `id` no esté vacío, que `uri` sea parseable y que `predicates` no sea una lista vacía. Un `RouteDefinition` sin predicates enruta todas las peticiones al destino indicado, lo que puede generar tráfico inesperado.
- Publicar `RefreshRoutesEvent` solo una vez por operación de escritura, no una vez por ruta. Si se crean varias rutas en batch, acumularlas y publicar el evento al final reduce el número de reconstrucciones del `CachingRouteLocator`.
- Registrar en un log de auditoría cada modificación de rutas: qué ruta se creó o eliminó, quién la solicitó (extraído del JWT o de un header de identidad) y cuándo. Las rutas son configuración crítica del sistema.

Evitar:
- Exponer el endpoint `/admin/routes` sin autenticación. Cualquier cliente que conozca la URL podría añadir una ruta que redirija tráfico a un servidor externo, produciendo una vulnerabilidad de Server-Side Request Forgery (SSRF).
- Confiar solo en la validación del JSON schema del body. Un `uri` con el valor `file:///etc/passwd` o `http://169.254.169.254/latest/meta-data/` es sintácticamente válido pero apunta a recursos internos o del sistema. Validar el esquema de la URI (`http` o `lb` únicamente) y bloquear rangos de IPs privadas.

---

← [6.8.1 RouteDefinitionLocator](./06-33-gateway-route-locator.md) | [Índice](./README.md) | [6.8.3 GlobalFilter reactivo →](./06-35-gateway-filter-reactivo.md)
