# 6.8.1 RouteDefinitionLocator: rutas dinámicas desde base de datos con caché reactivo

← [6.7.5 Tests de Circuit Breaker](./06-32-gateway-testing-circuit-breaker.md) | [Índice](./README.md) | [6.8.2 RouteDefinitionWriter →](./06-34-gateway-route-writer.md)

---

La configuración de rutas en YAML es estática: cualquier cambio requiere redesplegar o recargar el contexto. En plataformas multi-tenant o en sistemas donde los equipos de negocio necesitan añadir o modificar rutas sin intervención de operaciones, este modelo no es suficiente. `RouteDefinitionLocator` es la interfaz que Spring Cloud Gateway usa para obtener las definiciones de ruta desde cualquier fuente: YAML, Java, base de datos, Redis, o cualquier sistema externo. Implementarla permite que las rutas del Gateway se lean desde una tabla en base de datos, se actualicen en caliente desde una API REST y se cacheen en memoria para no golpear la base de datos en cada petición.

## Arquitectura: RouteDefinitionLocator en el pipeline del Gateway

Spring Cloud Gateway compone todas las fuentes de rutas a través de `CompositeRouteDefinitionLocator`: detecta automáticamente en el contexto todos los beans que implementen `RouteDefinitionLocator`, incluido el locator nativo de YAML y cualquier implementación personalizada. La caché de rutas en memoria se reconstruye completa cada vez que se publica un `RefreshRoutesEvent`.

```
ARRANQUE Y OPERACIÓN NORMAL:

  Spring Container
    │  detecta todos los beans RouteDefinitionLocator
    │  (incluido el YAML nativo + el custom desde BD)
    ▼
  CompositeRouteDefinitionLocator
    │  combina todas las fuentes de RouteDefinition
    ▼
  RouteDefinitionRouteLocator
    │  convierte RouteDefinition → Route (con predicates y filtros instanciados)
    ▼
  CachingRouteLocator
    │  cachea la lista de Routes en memoria
    ▼
  RoutePredicateHandlerMapping
    │  usa la caché para resolver cada petición entrante
    ▼
  [Petición enrutada]

ACTUALIZACIÓN DE RUTAS EN CALIENTE:

  API REST /admin/routes  →  escritura en BD
    │
    ▼
  Publicar RefreshRoutesEvent (o llamar a RouteDefinitionRouteLocator.refresh())
    │
    ▼
  CachingRouteLocator invalida su caché
    │
    ▼
  RouteDefinitionLocator.getRouteDefinitions() se llama de nuevo
    │  (lee desde BD en lugar de YAML)
    ▼
  Nueva lista de Routes activa sin reiniciar el Gateway
```

## Implementación: DatabaseRouteDefinitionLocator

La clase implementa el único método de la interfaz, `getRouteDefinitions()`, delegando en el repositorio R2DBC y convirtiendo cada fila de la tabla en un `RouteDefinition` con sus predicates y filtros deserializados desde las columnas JSON. La anotación `@Profile("!test")` excluye este bean en los tests, donde las rutas se definen en YAML para evitar la dependencia con la base de datos.

```java
import org.springframework.cloud.gateway.route.RouteDefinition;
import org.springframework.cloud.gateway.route.RouteDefinitionLocator;
import org.springframework.context.annotation.Profile;
import org.springframework.stereotype.Component;
import reactor.core.publisher.Flux;

// El repositorio R2DBC necesita la dependencia spring-boot-starter-data-r2dbc
// y un driver reactivo (r2dbc-postgresql, r2dbc-h2, etc.)
@Component
@Profile("!test")   // no activar en tests unitarios; usar el YAML de test
public class DatabaseRouteDefinitionLocator implements RouteDefinitionLocator {

    private final RouteDefinitionRepository routeRepository;

    public DatabaseRouteDefinitionLocator(RouteDefinitionRepository routeRepository) {
        this.routeRepository = routeRepository;
    }

    @Override
    public Flux<RouteDefinition> getRouteDefinitions() {
        return routeRepository.findAll()
            .map(this::toRouteDefinition);
    }

    private RouteDefinition toRouteDefinition(RouteEntity entity) {
        RouteDefinition definition = new RouteDefinition();
        definition.setId(entity.getRouteId());
        definition.setUri(java.net.URI.create(entity.getUri()));
        definition.setOrder(entity.getOrder());

        // Parsear predicates desde JSON almacenado en la tabla
        // Formato: [{"name": "Path", "args": {"_genkey_0": "/api/v1/productos/**"}}]
        definition.setPredicates(parsePredicates(entity.getPredicatesJson()));
        definition.setFilters(parseFilters(entity.getFiltersJson()));

        return definition;
    }

    private java.util.List<org.springframework.cloud.gateway.handler.predicate.PredicateDefinition>
            parsePredicates(String json) {
        // Deserializar desde JSON usando ObjectMapper
        // El formato es el mismo que usa Spring al parsear el YAML
        try {
            com.fasterxml.jackson.databind.ObjectMapper mapper =
                new com.fasterxml.jackson.databind.ObjectMapper();
            return mapper.readValue(json,
                mapper.getTypeFactory().constructCollectionType(
                    java.util.List.class,
                    org.springframework.cloud.gateway.handler.predicate.PredicateDefinition.class));
        } catch (Exception e) {
            return java.util.List.of();
        }
    }

    private java.util.List<org.springframework.cloud.gateway.filter.FilterDefinition>
            parseFilters(String json) {
        try {
            com.fasterxml.jackson.databind.ObjectMapper mapper =
                new com.fasterxml.jackson.databind.ObjectMapper();
            return mapper.readValue(json,
                mapper.getTypeFactory().constructCollectionType(
                    java.util.List.class,
                    org.springframework.cloud.gateway.filter.FilterDefinition.class));
        } catch (Exception e) {
            return java.util.List.of();
        }
    }
}
```

## Entidad y repositorio R2DBC

La entidad `RouteEntity` mapea la tabla `gateway_routes` con R2DBC: los predicates y los filtros se almacenan como texto JSON en columnas `TEXT` en lugar de tablas relacionales adicionales, lo que simplifica las operaciones de lectura y escritura a costa de no poder hacer consultas SQL sobre los predicates individuales.

```java
import org.springframework.data.annotation.Id;
import org.springframework.data.relational.core.mapping.Table;
import org.springframework.data.repository.reactive.ReactiveCrudRepository;
import reactor.core.publisher.Flux;

@Table("gateway_routes")
public class RouteEntity {

    @Id
    private Long id;
    private String routeId;       // identificador único de la ruta
    private String uri;           // lb://servicio o http://host:port
    private int order;
    private boolean enabled;      // permite desactivar rutas sin eliminarlas
    private String predicatesJson; // JSON con la lista de PredicateDefinition
    private String filtersJson;    // JSON con la lista de FilterDefinition

    // Getters y setters
    public Long getId() { return id; }
    public String getRouteId() { return routeId; }
    public String getUri() { return uri; }
    public int getOrder() { return order; }
    public boolean isEnabled() { return enabled; }
    public String getPredicatesJson() { return predicatesJson; }
    public String getFiltersJson() { return filtersJson; }

    public void setId(Long id) { this.id = id; }
    public void setRouteId(String routeId) { this.routeId = routeId; }
    public void setUri(String uri) { this.uri = uri; }
    public void setOrder(int order) { this.order = order; }
    public void setEnabled(boolean enabled) { this.enabled = enabled; }
    public void setPredicatesJson(String predicatesJson) { this.predicatesJson = predicatesJson; }
    public void setFiltersJson(String filtersJson) { this.filtersJson = filtersJson; }
}

public interface RouteDefinitionRepository extends ReactiveCrudRepository<RouteEntity, Long> {

    // Solo devuelve rutas activas
    Flux<RouteEntity> findAll();

    Flux<RouteEntity> findByEnabledTrue();
}
```

## Esquema SQL de la tabla de rutas

La tabla `gateway_routes` incluye el campo `enabled` para permitir desactivar rutas temporalmente sin eliminarlas, y las columnas `predicates_json` y `filters_json` con el mismo formato JSON que usa Spring al parsear la configuración YAML, lo que hace posible reutilizar los mismos deserializadores en el locator.

```sql
CREATE TABLE gateway_routes (
    id              BIGSERIAL PRIMARY KEY,
    route_id        VARCHAR(100) NOT NULL UNIQUE,
    uri             VARCHAR(500) NOT NULL,
    "order"         INTEGER DEFAULT 0,
    enabled         BOOLEAN DEFAULT TRUE,
    predicates_json TEXT NOT NULL,  -- JSON array de PredicateDefinition
    filters_json    TEXT NOT NULL DEFAULT '[]',
    created_at      TIMESTAMP DEFAULT NOW(),
    updated_at      TIMESTAMP DEFAULT NOW()
);

-- Insertar una ruta de ejemplo
INSERT INTO gateway_routes (route_id, uri, "order", predicates_json, filters_json)
VALUES (
    'productos-route',
    'lb://productos-service',
    0,
    '[{"name": "Path", "args": {"_genkey_0": "/api/v1/productos/**"}}]',
    '[{"name": "StripPrefix", "args": {"_genkey_0": "2"}}]'
);
```

## Caché reactivo para evitar lecturas en cada petición

Sin caché, `getRouteDefinitions()` golpea la base de datos en cada petición al Gateway. `CachingRouteLocator` de Spring cachea el resultado, pero solo se invalida cuando se publica un `RefreshRoutesEvent`. Para controlar manualmente la caché:

```java
import org.springframework.cloud.gateway.event.RefreshRoutesEvent;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.stereotype.Service;

@Service
public class RouteRefreshService {

    private final ApplicationEventPublisher publisher;

    public RouteRefreshService(ApplicationEventPublisher publisher) {
        this.publisher = publisher;
    }

    public void refreshRoutes() {
        // Invalida la caché de CachingRouteLocator y fuerza releer todas las fuentes
        publisher.publishEvent(new RefreshRoutesEvent(this));
    }
}
```

> [CONCEPTO] `RefreshRoutesEvent` es el mecanismo estándar de Spring Cloud Gateway para invalidar la caché de rutas. `CachingRouteLocator` escucha este evento, limpia su caché interna y llama de nuevo a todos los `RouteDefinitionLocator` para reconstruir la lista de `Route`. El evento se puede publicar desde un endpoint REST, un Consumer de Kafka, un scheduler o cualquier otro mecanismo que detecte cambios en las rutas.

## Parámetros y dependencias

Los tipos del API público de Spring Cloud Gateway que intervienen en la implementación del locator son clases del módulo `spring-cloud-gateway-server`. Conocer sus campos y su relación con la configuración YAML es necesario para implementar correctamente la conversión desde la entidad de base de datos.

| Elemento | Descripción |
|---|---|
| `RouteDefinitionLocator` | Interfaz con un único método `Flux<RouteDefinition> getRouteDefinitions()` |
| `RouteDefinition` | DTO con `id`, `uri`, `order`, `predicates` (lista de `PredicateDefinition`) y `filters` |
| `PredicateDefinition` | DTO con `name` (ej. `"Path"`) y `args` (mapa de argumentos) |
| `FilterDefinition` | DTO con `name` (ej. `"StripPrefix"`) y `args` |
| `RefreshRoutesEvent` | Evento que invalida la caché de `CachingRouteLocator` |
| `spring-boot-starter-data-r2dbc` | Dependencia para acceso reactivo a base de datos |

## Buenas y malas prácticas

Hacer:
- Añadir el campo `enabled` en la tabla de rutas para poder desactivar una ruta sin eliminarla. Esto permite reactivarla rápidamente si es necesario y mantiene un historial de rutas pasadas.
- Publicar `RefreshRoutesEvent` tras cada modificación de rutas en la base de datos en lugar de esperar a un refresco periódico. El refresco periódico introduce latencia en la activación de nuevas rutas y puede crear inconsistencias si múltiples instancias del Gateway refrescan en momentos distintos.
- Monitorizar el número de rutas activas con el endpoint `/actuator/gateway/routes` y alertar si baja de un umbral esperado, lo que indicaría una pérdida accidental de configuración en la base de datos.

Evitar:
- No implementar caché en el `RouteDefinitionLocator` personalizado. Sin la caché de `CachingRouteLocator` (que funciona automáticamente cuando se publica `RefreshRoutesEvent`), cada petición al Gateway genera una consulta a la base de datos, multiplicando la latencia de cada petición por el tiempo de la consulta.
- Almacenar el `uri` como `lb://nombre-servicio` para servicios de Kubernetes sin tener Eureka configurado. `lb://` requiere un `LoadBalancerClient` que resuelva el nombre de servicio. En Kubernetes sin Eureka, usar la URL de Kubernetes Service directamente o `http://nombre-servicio.namespace.svc.cluster.local`.

---

← [6.7.5 Tests de Circuit Breaker](./06-32-gateway-testing-circuit-breaker.md) | [Índice](./README.md) | [6.8.2 RouteDefinitionWriter →](./06-34-gateway-route-writer.md)
