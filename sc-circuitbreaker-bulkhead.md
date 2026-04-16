# 5.7 Bulkhead: aislamiento de recursos con SemaphoreBulkhead y ThreadPoolBulkhead

← [5.6 Retry: política de reintentos, backoff y anotación @Retry](sc-circuitbreaker-retry.md) | [Índice](README.md) | [5.8 RateLimiter: control de tasa de llamadas con Resilience4j](sc-circuitbreaker-ratelimiter.md) →

## Introducción

El Circuit Breaker protege contra fallos sostenidos, pero no contra el agotamiento de recursos por concurrencia excesiva. Si 200 threads simultáneos llaman al mismo servicio lento y cada uno espera 10 segundos, el pool de threads del servidor queda completamente bloqueado y el sistema deja de responder a cualquier petición, incluyendo las que no tienen nada que ver con ese servicio. Este patrón de fallo se llama **thread exhaustion** y es el problema que resuelve el Bulkhead.

El nombre viene de los mamparos estancos de los barcos: compartimentos independientes que evitan que una inundación en un compartimento hunda el barco completo. En software, el Bulkhead limita el número de llamadas concurrentes a un servicio específico, aislando su consumo de recursos del resto de la aplicación.

Resilience4j ofrece dos implementaciones con características radicalmente distintas: `SemaphoreBulkhead` (sincrónico, liviano, usa el thread del caller) y `ThreadPoolBulkhead` (asíncrono, crea un pool dedicado, devuelve `CompletableFuture`). La elección entre ellas define la arquitectura de threading del servicio.

> [ADVERTENCIA] ThreadPoolBulkhead cambia la naturaleza de la llamada a asíncrona: el método decorado debe retornar `CompletableFuture<T>`. Usar ThreadPoolBulkhead en métodos síncronos requiere cambios en la firma del método y en todos sus llamadores.

## Representación visual

La diferencia fundamental entre los dos tipos de Bulkhead está en qué thread ejecuta la lógica y qué ocurre cuando el límite se alcanza:

```
SemaphoreBulkhead (semáforo de conteo):
────────────────────────────────────────
Thread A ──→ adquiere permiso ──→ ejecuta en Thread A ──→ libera permiso
Thread B ──→ adquiere permiso ──→ ejecuta en Thread B ──→ libera permiso
Thread C ──→ espera permiso (maxWaitDuration)
               │
               ├─ permiso disponible antes del timeout ──→ ejecuta
               └─ timeout agotado ──→ BulkheadFullException

ThreadPoolBulkhead (pool dedicado):
────────────────────────────────────
Thread A ──→ envía tarea al pool ──→ retorna CompletableFuture<T>
                     │
              Pool dedicado (core=2, max=4, queue=10):
              Thread-Pool-1 ──→ ejecuta la tarea
              Thread-Pool-2 ──→ ejecuta la tarea
              Cola (hasta 10) ──→ espera thread libre
              Cola llena ──→ BulkheadFullException
```

| Característica                  | SemaphoreBulkhead                           | ThreadPoolBulkhead                               |
|---------------------------------|---------------------------------------------|--------------------------------------------------|
| Thread de ejecución             | Thread del llamador                         | Thread del pool dedicado                         |
| Tipo de retorno                 | `T` (síncrono)                              | `CompletableFuture<T>` (asíncrono)               |
| Aislamiento de threads          | No (mismo pool)                             | Sí (pool separado)                               |
| Overhead                        | Muy bajo (semáforo JVM)                     | Mayor (gestión de thread pool)                   |
| Compatible con WebFlux          | Sí                                          | No recomendado                                   |
| Namespace YAML                  | `resilience4j.bulkhead`                     | `resilience4j.thread-pool-bulkhead`              |
| `@Bulkhead` type                | `SEMAPHORE` (default)                       | `THREADPOOL`                                     |

## Ejemplo central

Aplicación Spring MVC que protege dos endpoints con Bulkhead: las consultas de catálogo con `SemaphoreBulkhead` (lectura rápida, aislamiento ligero) y el procesamiento de importaciones con `ThreadPoolBulkhead` (operación lenta, aislamiento completo de threads).

```xml
<!-- pom.xml -->
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-aop</artifactId>
    </dependency>
</dependencies>
```

```yaml
# application.yml
resilience4j:
  # SemaphoreBulkhead — para operaciones síncronas rápidas
  bulkhead:
    instances:
      catalogService:
        max-concurrent-calls: 20       # máximo 20 llamadas simultáneas
        max-wait-duration: 100ms       # espera hasta 100ms antes de rechazar

  # ThreadPoolBulkhead — para operaciones lentas o que deben aislarse completamente
  thread-pool-bulkhead:
    instances:
      importService:
        core-thread-pool-size: 2
        max-thread-pool-size: 4
        queue-capacity: 10
        keep-alive-duration: 30s
        writable-stack-trace-enabled: true
```

```java
// CatalogQueryService.java — SemaphoreBulkhead
package com.example.catalog.service;

import com.example.catalog.exception.CatalogUnavailableException;
import com.example.catalog.model.Product;
import io.github.resilience4j.bulkhead.annotation.Bulkhead;
import io.github.resilience4j.bulkhead.annotation.Bulkhead.Type;
import org.springframework.core.ParameterizedTypeReference;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestClient;
import java.util.List;

@Service
public class CatalogQueryService {

    private final RestClient restClient;

    public CatalogQueryService(RestClient.Builder builder) {
        this.restClient = builder.baseUrl("http://catalog-service").build();
    }

    // Type.SEMAPHORE es el default; se puede omitir
    @Bulkhead(name = "catalogService", type = Type.SEMAPHORE, fallbackMethod = "fallbackProducts")
    public List<Product> getProductsByCategory(String categoryId) {
        return restClient.get()
                .uri("/api/products?category={id}", categoryId)
                .retrieve()
                .body(new ParameterizedTypeReference<List<Product>>() {});
    }

    private List<Product> fallbackProducts(String categoryId, Throwable ex) {
        if (ex instanceof io.github.resilience4j.bulkhead.BulkheadFullException) {
            // El bulkhead está lleno: devolver lista vacía con indicador de degradación
            return List.of();
        }
        // Otro error: relanzar como excepción controlada
        throw new CatalogUnavailableException("Catálogo no disponible: " + ex.getMessage(), ex);
    }
}
```

```java
// ImportService.java — ThreadPoolBulkhead
package com.example.catalog.service;

import com.example.catalog.model.ImportResult;
import io.github.resilience4j.bulkhead.annotation.Bulkhead;
import io.github.resilience4j.bulkhead.annotation.Bulkhead.Type;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestClient;
import java.util.concurrent.CompletableFuture;

@Service
public class ImportService {

    private final RestClient restClient;

    public ImportService(RestClient.Builder builder) {
        this.restClient = builder.baseUrl("http://import-service").build();
    }

    /**
     * ThreadPoolBulkhead requiere que el método retorne CompletableFuture<T>.
     * La tarea se ejecuta en el pool dedicado "importService".
     * Hasta 4 tareas concurrentes + 10 en cola; las restantes lanzan BulkheadFullException.
     */
    @Bulkhead(name = "importService", type = Type.THREADPOOL, fallbackMethod = "fallbackImport")
    public CompletableFuture<ImportResult> processImport(String importId) {
        return CompletableFuture.supplyAsync(() ->
                restClient.post()
                        .uri("/api/imports/{id}/process", importId)
                        .retrieve()
                        .body(ImportResult.class)
        );
    }

    private CompletableFuture<ImportResult> fallbackImport(String importId, Throwable ex) {
        return CompletableFuture.completedFuture(
                new ImportResult(importId, "REJECTED",
                        "Importación rechazada por sobrecarga: " + ex.getClass().getSimpleName())
        );
    }
}
```

```java
// Modelos
package com.example.catalog.model;

public record Product(String id, String name, String categoryId, double price) {}
public record ImportResult(String importId, String status, String message) {}
```

```java
// Excepción controlada
package com.example.catalog.exception;

public class CatalogUnavailableException extends RuntimeException {
    public CatalogUnavailableException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

## Tabla de elementos clave

Propiedades para cada tipo de Bulkhead. Los namespaces son distintos según el tipo.

| Propiedad (SemaphoreBulkhead)          | Tipo     | Default | Descripción                                                          |
|----------------------------------------|----------|---------|----------------------------------------------------------------------|
| `max-concurrent-calls`                 | int      | `25`    | Máximo de llamadas simultáneas permitidas                            |
| `max-wait-duration`                    | Duration | `0`     | Tiempo máximo de espera para adquirir permiso; `0` = rechazar inmediatamente |
| `writable-stack-trace-enabled`         | boolean  | `true`  | Incluye stack trace en `BulkheadFullException`                       |

| Propiedad (ThreadPoolBulkhead)         | Tipo     | Default | Descripción                                                          |
|----------------------------------------|----------|---------|----------------------------------------------------------------------|
| `core-thread-pool-size`                | int      | `N_CPU - 1` | Threads fijos en el pool                                         |
| `max-thread-pool-size`                 | int      | `N_CPU` | Threads máximos cuando la cola está llena                            |
| `queue-capacity`                       | int      | `100`   | Tamaño de la cola de tareas pendientes                               |
| `keep-alive-duration`                  | Duration | `20ms`  | Tiempo que un thread ocioso extra permanece antes de terminar        |
| `writable-stack-trace-enabled`         | boolean  | `true`  | Incluye stack trace en `BulkheadFullException`                       |

## Buenas y malas prácticas

**Hacer:**

- Usar `SemaphoreBulkhead` para servicios de lectura frecuente y latencia baja (< 500ms). El overhead del semáforo es mínimo y no cambia la API del método.
- Usar `ThreadPoolBulkhead` para operaciones lentas (importaciones, exportaciones, llamadas a servicios de terceros con SLA de segundos) donde el aislamiento real de threads es importante para no consumir el pool de Tomcat/Netty.
- Calibrar `max-concurrent-calls` en `SemaphoreBulkhead` basándose en el throughput esperado y la latencia media: `max-concurrent-calls ≈ throughput_rps × latencia_segundos` (Ley de Little). Un valor 2x este cálculo como margen es razonable.
- Ajustar `queue-capacity` en `ThreadPoolBulkhead` al volumen de tareas que el sistema puede acumular sin que el tiempo de cola supere el SLA. Una cola grande puede generar latencias muy altas.

**Evitar:**

- No usar `ThreadPoolBulkhead` en aplicaciones WebFlux. La programación reactiva ya gestiona la concurrencia con pocos threads; añadir un pool dedicado por servicio invierte el beneficio del modelo reactivo.
- Evitar `max-wait-duration: 0` (rechazar inmediatamente) en servicios donde los picos de concurrencia son muy cortos (< 50ms). Un poco de espera puede absorber picos transitorios sin rechazar llamadas legítimas.
- No asumir que `ThreadPoolBulkhead` hace el método thread-safe. El aislamiento es de threads del caller, no de la lógica del servicio; si el método accede a estado compartido, sigue siendo necesaria la sincronización.
- Evitar configurar `max-concurrent-calls` demasiado bajo sin medir la concurrencia real. Un valor de 5 en un servicio con 50 req/s garantiza que el 90% de las llamadas sean rechazadas, no que el servicio esté protegido.

## Comparación: cuándo usar cada tipo

La siguiente tabla resume los criterios de decisión para elegir entre las dos implementaciones:

| Criterio                          | SemaphoreBulkhead                       | ThreadPoolBulkhead                         |
|-----------------------------------|-----------------------------------------|--------------------------------------------|
| Latencia del servicio remoto      | Baja (< 500ms)                          | Alta (> 1s) o variable                     |
| Modelo de programación            | Síncrono / blocking                     | Asíncrono / non-blocking                   |
| Tipo de retorno del método        | `T` (cualquier tipo)                    | `CompletableFuture<T>` obligatorio         |
| Impacto en threads del servidor   | Usa threads del caller                  | Pool dedicado, no toca threads del caller  |
| Overhead de configuración         | Mínimo                                  | Requiere dimensionar el pool               |
| Compatible con WebFlux            | Sí                                      | No recomendado                             |

---

← [5.6 Retry: política de reintentos, backoff y anotación @Retry](sc-circuitbreaker-retry.md) | [Índice](README.md) | [5.8 RateLimiter: control de tasa de llamadas con Resilience4j](sc-circuitbreaker-ratelimiter.md) →
