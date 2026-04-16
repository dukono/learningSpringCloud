# 5.9 TimeLimiter: acotación de tiempos de ejecución asíncronos

← [5.8 RateLimiter: control de tasa de llamadas con Resilience4j](sc-circuitbreaker-ratelimiter.md) | [Índice](README.md) | [5.10 Anotaciones AOP de Resilience4j y orden de decoradores combinados](sc-circuitbreaker-aop.md) →

## Introducción

El TimeLimiter de Resilience4j acota el tiempo que una operación asíncrona puede ejecutarse antes de ser cancelada. Su caso de uso principal es envolver llamadas que devuelven `CompletableFuture` o `Mono/Flux` para garantizar que, incluso si el hilo subyacente queda colgado esperando respuesta, la llamada del punto de vista del servicio termina en un tiempo acotado.

La confusión más frecuente es entre el TimeLimiter y el timeout TCP de la capa de transporte (`SocketTimeoutException`). El timeout TCP se configura a nivel del cliente HTTP (RestClient, WebClient, Feign) y provoca una excepción cuando la conexión o la lectura tarda demasiado desde el punto de vista del socket. El TimeLimiter opera a nivel de la lógica de la aplicación y puede cancelar tareas asíncronas que no tienen ningún timeout TCP configurado, como tareas en un `ThreadPoolBulkhead` o llamadas reactivas con operaciones complejas.

La interacción con el Circuit Breaker es también importante: el TimeLimiter lanza `TimeoutException`, que el Circuit Breaker puede registrar como un fallo (si está en `record-exceptions`) o como una llamada lenta (si el tiempo de ejecución supera `slow-call-duration-threshold`). Configurar bien esta interacción evita que el circuito abra por timeouts que son esperados durante operaciones de larga duración planificadas.

> [CONCEPTO] El TimeLimiter solo tiene efecto sobre operaciones asíncronas (`CompletableFuture` o `Mono/Flux`). Aplicarlo a un método síncrono no acota su tiempo de ejecución; el thread del caller esperará igualmente hasta que el método retorne.

## Representación visual

La interacción entre el TimeLimiter y los dos contextos de programación (imperativo y reactivo):

```
Contexto imperativo (CompletableFuture):
─────────────────────────────────────────────────────
  TimeLimiter.executeFutureSupplier(
      () -> CompletableFuture.supplyAsync(() -> remoteCall())
  )
         │
         ├─ Completa en < timeoutDuration ──→ retorna resultado
         └─ Supera timeoutDuration ──→ TimeoutException
                    │
                    ├─ cancelRunningFuture=true  ──→ cancela el Future subyacente
                    └─ cancelRunningFuture=false ──→ deja el Future ejecutarse
                                                      (pero ignora su resultado)

Contexto reactivo (Mono/Flux con reactor-resilience4j):
─────────────────────────────────────────────────────────
  reactiveCircuitBreakerFactory.create("serviceName")
      .run(
          webClient.get().uri(...).retrieve().bodyToMono(T.class),
          t -> Mono.just(fallback)
      )
  La fábrica reactiva aplica TimeLimiter automáticamente si está configurado.
```

| Escenario                              | Timeout recomendado           | Mecanismo                                 |
|----------------------------------------|-------------------------------|-------------------------------------------|
| Timeout de conexión TCP                | RestClient / WebClient config | `ConnectTimeout` en cliente HTTP          |
| Timeout de lectura TCP                 | RestClient / WebClient config | `ReadTimeout` en cliente HTTP             |
| Timeout de operación asíncrona (pool)  | TimeLimiter                   | `timeoutDuration` en instancia TimeLimiter|
| Timeout de operación reactiva (Mono)   | TimeLimiter (reactor)         | Aplicado por `ReactiveCircuitBreakerFactory`|
| Apertura por llamadas lentas           | Circuit Breaker               | `slowCallDurationThreshold`               |

## Ejemplo central

Servicio de generación de informes PDF que utiliza un `ThreadPoolBulkhead` para las tareas pesadas y un `TimeLimiter` para garantizar que ninguna tarea dura más de 10 segundos.

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
  timelimiter:
    instances:
      reportGeneration:
        timeout-duration: 10s
        cancel-running-future: true    # cancela el Future si supera el timeout
  
  thread-pool-bulkhead:
    instances:
      reportGeneration:
        core-thread-pool-size: 2
        max-thread-pool-size: 4
        queue-capacity: 5
  
  circuitbreaker:
    instances:
      reportGeneration:
        sliding-window-size: 10
        minimum-number-of-calls: 3
        failure-rate-threshold: 50
        # Los TimeoutException del TimeLimiter cuentan como fallos del CB
        record-exceptions:
          - java.util.concurrent.TimeoutException
          - java.io.IOException
        wait-duration-in-open-state: 60s
```

```java
// ReportGenerationService.java — combinación TimeLimiter + ThreadPoolBulkhead + CB
package com.example.reports.service;

import com.example.reports.exception.ReportTimeoutException;
import com.example.reports.model.PdfReport;
import io.github.resilience4j.bulkhead.annotation.Bulkhead;
import io.github.resilience4j.bulkhead.annotation.Bulkhead.Type;
import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import io.github.resilience4j.timelimiter.annotation.TimeLimiter;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestClient;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.TimeoutException;

@Service
public class ReportGenerationService {

    private final RestClient restClient;

    public ReportGenerationService(RestClient.Builder builder) {
        this.restClient = builder.baseUrl("http://pdf-service").build();
    }

    /**
     * Orden de decoradores con las tres anotaciones:
     * Bulkhead (exterior) > CircuitBreaker > TimeLimiter (interior)
     *
     * 1. Bulkhead comprueba si hay capacidad en el pool dedicado
     * 2. CircuitBreaker comprueba si el servicio está OPEN
     * 3. TimeLimiter acota el tiempo de ejecución del Future
     */
    @Bulkhead(name = "reportGeneration", type = Type.THREADPOOL, fallbackMethod = "fallbackReport")
    @CircuitBreaker(name = "reportGeneration", fallbackMethod = "fallbackReport")
    @TimeLimiter(name = "reportGeneration", fallbackMethod = "fallbackReport")
    public CompletableFuture<PdfReport> generateReport(String reportId, String userId) {
        return CompletableFuture.supplyAsync(() ->
                restClient.post()
                        .uri("/api/reports/generate")
                        .body(new ReportRequest(reportId, userId))
                        .retrieve()
                        .body(PdfReport.class)
        );
    }

    private CompletableFuture<PdfReport> fallbackReport(
            String reportId, String userId, Throwable ex) {

        String reason;
        if (ex instanceof TimeoutException) {
            reason = "TIMEOUT";
        } else if (ex instanceof io.github.resilience4j.bulkhead.BulkheadFullException) {
            reason = "BULKHEAD_FULL";
        } else if (ex instanceof io.github.resilience4j.circuitbreaker.CallNotPermittedException) {
            reason = "CIRCUIT_OPEN";
        } else {
            reason = "ERROR: " + ex.getClass().getSimpleName();
        }

        return CompletableFuture.completedFuture(
                new PdfReport(reportId, "FAILED", reason, null));
    }
}
```

```java
// Modelos
package com.example.reports.model;

public record ReportRequest(String reportId, String userId) {}
public record PdfReport(String reportId, String status, String reason, byte[] content) {}
```

```java
package com.example.reports.exception;

public class ReportTimeoutException extends RuntimeException {
    public ReportTimeoutException(String message) { super(message); }
}
```

## Tabla de elementos clave

Propiedades del namespace `resilience4j.timelimiter.instances.[name]` y su interacción con el Circuit Breaker.

| Propiedad / Concepto               | Tipo / Valores     | Default  | Descripción                                                                            |
|------------------------------------|--------------------|----------|----------------------------------------------------------------------------------------|
| `timeout-duration`                 | Duration           | `1s`     | Tiempo máximo de ejecución antes de lanzar `TimeoutException`                         |
| `cancel-running-future`            | boolean            | `true`   | Si `true`, cancela el `CompletableFuture` subyacente al vencer el timeout             |
| `TimeoutException` vs Circuit CB   | —                  | —        | El CB registra `TimeoutException` como fallo si está en `record-exceptions`           |
| `slow-call-duration-threshold` (CB)| Duration           | `60s`    | El CB registra llamadas lentas si superan este umbral, independientemente del TL       |
| Compatibilidad reactiva            | `Mono<T>`, `Flux<T>` | —      | El starter reactor aplica el TimeLimiter a flujos reactivos automáticamente           |
| AOP con `@TimeLimiter`             | método debe retornar `CompletableFuture` | — | La anotación no funciona en métodos síncronos |

## Buenas y malas prácticas

**Hacer:**

- Configurar `timeout-duration` como el P99 del SLA del servicio remoto, no el valor medio. Si el P99 es 5s, usar `timeout-duration: 6s`; así el 99% de las llamadas completan sin timeout.
- Activar `cancel-running-future: true` para evitar que los threads del `ThreadPoolBulkhead` queden ocupados ejecutando una tarea que ya fue cancelada desde el punto de vista del llamador.
- Registrar `TimeoutException` en `record-exceptions` del Circuit Breaker solo si los timeouts son síntoma de un servicio degradado. Si los timeouts son esperados en operaciones largas planificadas, no deben abrir el circuito.
- Combinar TimeLimiter con `slow-call-duration-threshold` del Circuit Breaker de forma coherente: el umbral de llamada lenta debería ser menor que el `timeout-duration` del TimeLimiter para que el CB detecte la degradación antes de que el TL cancele la llamada.

**Evitar:**

- No aplicar `@TimeLimiter` a métodos síncronos esperando que limite su tiempo de ejecución. La anotación solo tiene efecto en métodos que retornan `CompletableFuture`; en métodos síncronos el decorador no puede interrumpir el thread.
- Evitar `timeout-duration` demasiado corto (< 200ms) para servicios que realizan operaciones de base de datos. Los picos de latencia en BD son frecuentes y legítimos; un timeout muy agresivo causa fallos donde el servicio funciona correctamente.
- No confundir `TimeoutException` del TimeLimiter (`java.util.concurrent.TimeoutException`) con `TimeoutException` de otras librerías. El Circuit Breaker registra por tipo de clase; usar el nombre completo en `record-exceptions` para evitar ambigüedad.

---

← [5.8 RateLimiter: control de tasa de llamadas con Resilience4j](sc-circuitbreaker-ratelimiter.md) | [Índice](README.md) | [5.10 Anotaciones AOP de Resilience4j y orden de decoradores combinados](sc-circuitbreaker-aop.md) →
