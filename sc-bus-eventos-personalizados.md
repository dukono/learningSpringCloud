# 7.5 Eventos personalizados de Spring Cloud Bus

← [7.4 Refresh de configuración distribuido con Spring Cloud Bus](sc-bus-refresh-distribuido.md) | [Índice](README.md) | [7.6 Seguridad del endpoint actuator de Spring Cloud Bus](sc-bus-seguridad-endpoint.md) →

## Introducción

Los eventos built-in de Spring Cloud Bus —`RefreshRemoteApplicationEvent` y `EnvironmentChangeRemoteApplicationEvent`— resuelven el caso de uso de infraestructura, pero hay escenarios de producción donde se necesita propagar eventos de infraestructura propios: invalidar una caché distribuida en todos los nodos, notificar a todos los servicios de un cambio en una lista negra de tokens, o coordinar un modo de mantenimiento sin reiniciar los pods. Spring Cloud Bus permite definir eventos personalizados extendiendo `RemoteApplicationEvent`, registrarlos con `@RemoteApplicationEventScan` para que el framework los deserialice correctamente, y enviarlos con el `ApplicationEventPublisher` estándar de Spring. El campo `destinationService` del evento controla si el broadcast llega a todos los nodos del bus o solo a un subconjunto específico.

> [ADVERTENCIA] El registro del paquete con `@RemoteApplicationEventScan` es obligatorio. Si el paquete no está escaneado, el bus intentará deserializar el evento como `RemoteApplicationEvent` genérico y el receptor no podrá procesarlo, generando un error de tipo silencioso difícil de diagnosticar.

## Representación visual

El diagrama siguiente muestra el ciclo de vida completo de un evento personalizado desde su publicación hasta su recepción.

```
Publicador (Service-A)
┌──────────────────────────────────────────────────────┐
│  1. new CacheInvalidationEvent("products", "all")    │
│  2. applicationEventPublisher.publishEvent(event)    │
│  3. BusAutoConfiguration detecta RemoteApplicationEvent│
│  4. Serializa a JSON y publica en springCloudBus     │
└──────────────────────────────────────────────────────┘
                         │
                    Broker (Kafka/RabbitMQ)
                    topic: springCloudBus
                         │
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
  Service-A:inst-1  Service-B:inst-1  Service-C:inst-1
  (origen, ignorado) (recibe y procesa) (recibe y procesa)
        │
  5. Spring deserializa JSON → CacheInvalidationEvent
     (requiere @RemoteApplicationEventScan del paquete)
  6. @EventListener(CacheInvalidationEvent.class) invocado
  7. Lógica de negocio: invalidar caché "products"
```

La tabla siguiente resume los campos de `RemoteApplicationEvent` relevantes para eventos personalizados:

| Campo | Tipo | Descripción |
|---|---|---|
| `originService` | String | `bus.id` del nodo que publicó el evento. Rellenado automáticamente. |
| `destinationService` | String | Patrón de destino. Vacío = broadcast. Soporta wildcards. |
| `id` | String | UUID del evento. Usado para deduplicación. |

## Ejemplo central

El ejemplo siguiente implementa un evento de invalidación de caché distribuida que invalida la caché de productos en todos los nodos del bus cuando el catálogo de productos se actualiza.

```xml
<!-- pom.xml -->
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-bus-kafka</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-cache</artifactId>
    </dependency>
</dependencies>
```

```yaml
# application.yml
spring:
  application:
    name: catalog-service
  cloud:
    bus:
      enabled: true
      id: ${spring.application.name}:${spring.profiles.active:default}:${random.value}
  kafka:
    bootstrap-servers: kafka:9092

management:
  endpoints:
    web:
      exposure:
        include: busrefresh, health, info
```

```java
// src/main/java/com/example/catalog/events/CacheInvalidationEvent.java
package com.example.catalog.events;

import org.springframework.cloud.bus.event.RemoteApplicationEvent;

// IMPORTANTE: esta clase debe estar en un paquete declarado
// en @RemoteApplicationEventScan para ser deserializable
public class CacheInvalidationEvent extends RemoteApplicationEvent {

    private String cacheName;
    private String cacheKey;

    // Constructor por defecto requerido por Jackson para deserialización
    protected CacheInvalidationEvent() {
    }

    // Constructor para broadcast a todos los nodos (destinationService = null)
    public CacheInvalidationEvent(Object source, String originService,
                                   String cacheName, String cacheKey) {
        super(source, originService);
        this.cacheName = cacheName;
        this.cacheKey = cacheKey;
    }

    // Constructor para envío dirigido a un servicio específico
    public CacheInvalidationEvent(Object source, String originService,
                                   String destinationService,
                                   String cacheName, String cacheKey) {
        super(source, originService, destinationService);
        this.cacheName = cacheName;
        this.cacheKey = cacheKey;
    }

    public String getCacheName() { return cacheName; }
    public void setCacheName(String cacheName) { this.cacheName = cacheName; }

    public String getCacheKey() { return cacheKey; }
    public void setCacheKey(String cacheKey) { this.cacheKey = cacheKey; }
}
```

```java
// src/main/java/com/example/catalog/CatalogApplication.java
package com.example.catalog;

import com.example.catalog.events.CacheInvalidationEvent;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cache.annotation.EnableCaching;
// @RemoteApplicationEventScan registra el paquete para que Bus
// pueda deserializar los eventos personalizados
import org.springframework.cloud.bus.jackson.RemoteApplicationEventScan;

@SpringBootApplication
@EnableCaching
@RemoteApplicationEventScan(basePackages = "com.example.catalog.events")
public class CatalogApplication {
    public static void main(String[] args) {
        SpringApplication.run(CatalogApplication.class, args);
    }
}
```

```java
// src/main/java/com/example/catalog/service/CatalogService.java
package com.example.catalog.service;

import com.example.catalog.events.CacheInvalidationEvent;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.cache.CacheManager;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.stereotype.Service;
import java.util.List;

@Service
public class CatalogService {

    private final ApplicationEventPublisher eventPublisher;
    private final CacheManager cacheManager;

    @Value("${spring.cloud.bus.id:unknown}")
    private String busId;

    public CatalogService(ApplicationEventPublisher eventPublisher,
                           CacheManager cacheManager) {
        this.eventPublisher = eventPublisher;
        this.cacheManager = cacheManager;
    }

    @Cacheable("products")
    public List<String> getProducts() {
        // Simulación de carga desde base de datos
        return List.of("product-1", "product-2", "product-3");
    }

    // Llamado cuando el catálogo se actualiza: invalida la caché en todos los nodos
    public void updateProduct(String productId) {
        // ... lógica de actualización en base de datos ...

        // Publicar evento de invalidación — broadcast a todos los nodos del bus
        CacheInvalidationEvent event = new CacheInvalidationEvent(
            this,          // source
            busId,         // originService (bus.id de este nodo)
            "products",    // cacheName
            productId      // cacheKey
        );
        eventPublisher.publishEvent(event);
    }

    // Versión con envío dirigido solo a un servicio específico
    public void invalidateCacheOnlyForSearchService(String productId) {
        CacheInvalidationEvent event = new CacheInvalidationEvent(
            this,
            busId,
            "search-service:**",  // destinationService: solo instancias de search-service
            "products",
            productId
        );
        eventPublisher.publishEvent(event);
    }
}
```

```java
// src/main/java/com/example/catalog/listener/CacheInvalidationListener.java
package com.example.catalog.listener;

import com.example.catalog.events.CacheInvalidationEvent;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.cache.CacheManager;
import org.springframework.context.event.EventListener;
import org.springframework.stereotype.Component;

@Component
public class CacheInvalidationListener {

    private static final Logger log = LoggerFactory.getLogger(CacheInvalidationListener.class);

    private final CacheManager cacheManager;

    public CacheInvalidationListener(CacheManager cacheManager) {
        this.cacheManager = cacheManager;
    }

    @EventListener
    public void onCacheInvalidation(CacheInvalidationEvent event) {
        String cacheName = event.getCacheName();
        String cacheKey = event.getCacheKey();

        log.info("Recibido CacheInvalidationEvent: cache={}, key={}, origen={}",
                 cacheName, cacheKey, event.getOriginService());

        var cache = cacheManager.getCache(cacheName);
        if (cache != null) {
            if ("all".equals(cacheKey)) {
                cache.clear();
                log.info("Caché '{}' invalidada completamente", cacheName);
            } else {
                cache.evict(cacheKey);
                log.info("Entrada '{}' invalidada en caché '{}'", cacheKey, cacheName);
            }
        }
    }
}
```

## Tabla de elementos clave

La tabla recoge los elementos de la API de eventos personalizados de Bus.

| Elemento | Tipo | Default | Descripción |
|---|---|---|---|
| `RemoteApplicationEvent` | Clase base | — | Clase que deben extender todos los eventos personalizados de Bus. |
| `@RemoteApplicationEventScan` | Anotación | — | Declara los paquetes donde Bus busca subclases de `RemoteApplicationEvent` para deserialización. Obligatorio. |
| `destinationService` | String | `null` (broadcast) | Patrón de nodos destino. Vacío o nulo = todos. Soporta wildcards `app:**`. |
| `originService` | String | `bus.id` automático | Identificador del nodo emisor. Se rellena automáticamente si el constructor llama a `super(source, originService)`. |
| `ApplicationEventPublisher` | Interface Spring | — | Publicador estándar de Spring. Inyectado por Spring; Bus intercepta los `RemoteApplicationEvent`. |
| `@EventListener` | Anotación | — | Marca el método receptor del evento personalizado. Solo se invoca si el evento llega a este nodo. |
| `id` | String (UUID) | Auto-generado | Identificador único del evento. Bus lo usa para evitar procesar el mismo evento dos veces en el nodo origen. |

## Buenas y malas prácticas

**Hacer:**

- Colocar todas las subclases de `RemoteApplicationEvent` en un paquete dedicado (por ejemplo `com.company.app.bus.events`) y declararlo en `@RemoteApplicationEventScan`. Esto facilita el mantenimiento y garantiza que el escaneo sea predecible: si el paquete se mueve, el error de deserialización es inmediato y obvio.
- Añadir el constructor por defecto sin argumentos a cada subclase de `RemoteApplicationEvent`. Jackson requiere un constructor sin argumentos para la deserialización. Sin él, los nodos receptores lanzan `InvalidDefinitionException` al intentar deserializar el evento.
- Usar `destinationService` cuando el evento solo es relevante para un tipo de servicio. Publicar en broadcast cuando solo el 10% de los nodos necesita el evento consume recursos de red y procesamiento en el 90% restante innecesariamente.

**Evitar:**

- No incluir objetos de dominio complejos en el payload del evento personalizado. Los eventos Bus son de infraestructura: deben transportar identificadores y tipos, no entidades completas. Un payload grande (más de 1 KB) indica que el caso de uso debería usar Spring Cloud Stream con un topic de negocio.
- No reutilizar el campo `originService` para lógica de negocio. El nodo origen siempre ignorará su propio evento (Bus deduplica por `originService` + `id`). Si la lógica requiere que el origen también procese el evento, publicarlo como un `ApplicationEvent` local además del `RemoteApplicationEvent`.
- No crear eventos personalizados para casos de uso que ya tienen soporte built-in. Crear un evento `ConfigRefreshEvent` personalizado en lugar de usar `busrefresh` es trabajo duplicado que no añade valor y complica el diagnóstico.

---

← [7.4 Refresh de configuración distribuido con Spring Cloud Bus](sc-bus-refresh-distribuido.md) | [Índice](README.md) | [7.6 Seguridad del endpoint actuator de Spring Cloud Bus](sc-bus-seguridad-endpoint.md) →
