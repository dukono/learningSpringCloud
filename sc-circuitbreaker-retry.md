# 5.6 Retry: política de reintentos, backoff y anotación @Retry

← [5.5 Fallback: estrategias de respuesta degradada](sc-circuitbreaker-fallback.md) | [Índice](README.md) | [5.7 Bulkhead: aislamiento de recursos con SemaphoreBulkhead y ThreadPoolBulkhead](sc-circuitbreaker-bulkhead.md) →

## Introducción

Los fallos transitorios son el caso de uso donde Retry aporta más valor: una desconexión de red momentánea, un timeout por GC del servicio remoto, o un conflicto de concurrencia que se resuelve al segundo intento. Sin Retry, estos fallos llegan directamente al usuario como errores; con Retry, el sistema los absorbe automáticamente.

El riesgo de Retry sin backoff es la amplificación del problema: si 100 clientes reintentan simultáneamente ante un servicio bajo estrés, se le añaden 300 llamadas adicionales (3 reintentos × 100 clientes) exactamente cuando menos las puede atender. El backoff exponencial con jitter resuelve este problema distribuyendo los reintentos en el tiempo y añadiendo aleatoriedad para evitar que todos los clientes reintenten al mismo momento.

La interacción entre Retry y Circuit Breaker es también un punto crítico: si Retry está activo antes del Circuit Breaker en el orden de decoradores, cada intento fallido cuenta como una llamada fallida para el Circuit Breaker. Si está después, el Circuit Breaker puede rechazar los reintentos con `CallNotPermittedException` antes de que Retry los ejecute.

> [ADVERTENCIA] Retry nunca debe usarse para operaciones no idempotentes (como crear un pedido o cargar una tarjeta) sin garantías de idempotencia en el servicio remoto. Un segundo intento puede crear duplicados o cobros dobles.

## Representación visual

El diagrama muestra el flujo de Retry con backoff exponencial y su interacción con el Circuit Breaker:

```
Llamada inicial
     │
     ▼
  Intento 1 ──→ FALLO (IOException)
     │
     ├── ¿retryOnException(ex)? → sí
     │
     └── Esperar waitDuration × exponentialBackoffMultiplier^0 = 500ms
              + jitter aleatorio (randomizedWaitFactor)
     │
     ▼
  Intento 2 ──→ FALLO (IOException)
     │
     ├── ¿maxAttempts alcanzado? → no (maxAttempts=3, intento=2)
     │
     └── Esperar 500ms × 2^1 = 1000ms + jitter
     │
     ▼
  Intento 3 ──→ FALLO (IOException)
     │
     └── maxAttempts=3 alcanzado → lanzar excepción o invocar fallback

Backoff exponencial con jitter (randomizedWaitFactor=0.5):
  Intento 1: 500ms
  Intento 2: 500 × 2^1 = 1000ms  → rango real: [500ms, 1500ms]
  Intento 3: 500 × 2^2 = 2000ms  → rango real: [1000ms, 3000ms]
```

| Tipo de backoff          | Propiedades                                       | Comportamiento                                  |
|--------------------------|---------------------------------------------------|-------------------------------------------------|
| Fijo                     | `wait-duration`                                   | Siempre espera el mismo tiempo entre reintentos |
| Exponencial              | `enable-exponential-backoff`, `exponential-backoff-multiplier` | Duplica (o multiplica) el tiempo en cada intento |
| Exponencial con jitter   | + `enable-randomized-wait`, `randomized-wait-factor` | Añade variación aleatoria para evitar thundering herd |

## Ejemplo central

Servicio de notificaciones por email que reintenta el envío con backoff exponencial y jitter, combinado con Circuit Breaker para proteger contra fallos sostenidos.

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
# application.yml — configuración de Retry con backoff exponencial
resilience4j:
  retry:
    instances:
      emailService:
        max-attempts: 3
        wait-duration: 500ms
        enable-exponential-backoff: true
        exponential-backoff-multiplier: 2.0
        enable-randomized-wait: true
        randomized-wait-factor: 0.5         # ±50% de jitter
        retry-exceptions:
          - java.io.IOException
          - org.springframework.web.client.HttpServerErrorException
          - java.net.ConnectException
        ignore-exceptions:
          - com.example.notify.exception.InvalidEmailException
        # Reintento basado en resultado (no solo en excepción)
        retry-on-result-predicate: com.example.notify.cb.EmailResultPredicate

      # Retry solo para idempotentes, sin backoff (p.ej. consultas de solo lectura)
      catalogService:
        max-attempts: 2
        wait-duration: 200ms
        retry-exceptions:
          - java.net.ConnectException
          - org.springframework.web.client.ResourceAccessException

  circuitbreaker:
    instances:
      emailService:
        sliding-window-size: 10
        minimum-number-of-calls: 5
        failure-rate-threshold: 50
        wait-duration-in-open-state: 60s
        automatic-transition-from-open-to-half-open-enabled: true
```

```java
// EmailResultPredicate.java — reintento basado en resultado
package com.example.notify.cb;

import com.example.notify.model.EmailSendResult;
import java.util.function.Predicate;

/**
 * Reintenta si el servidor de email devolvió un status "QUEUED"
 * (aceptado pero no entregado). Solo para estados que indican reintentabilidad.
 */
public class EmailResultPredicate implements Predicate<EmailSendResult> {

    @Override
    public boolean test(EmailSendResult result) {
        // true = reintenta; solo los estados transitorios justifican reintento
        return "TEMP_FAILURE".equals(result.status()) || "QUEUED".equals(result.status());
    }
}
```

```java
// NotificationService.java — uso combinado de @Retry y @CircuitBreaker
package com.example.notify.service;

import com.example.notify.exception.NotificationException;
import com.example.notify.model.EmailRequest;
import com.example.notify.model.EmailSendResult;
import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import io.github.resilience4j.retry.annotation.Retry;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestClient;

@Service
public class NotificationService {

    private final RestClient restClient;

    public NotificationService(RestClient.Builder builder) {
        this.restClient = builder.baseUrl("http://email-service").build();
    }

    /**
     * Orden de decoradores: @CircuitBreaker envuelve a @Retry.
     * Si el circuito está OPEN, el CB rechaza antes de que Retry intente.
     * Si el circuito está CLOSED, Retry puede hacer varios intentos que
     * todos cuentan como llamadas para el CB.
     *
     * Para invertir el orden y que Retry actúe dentro del CB:
     * usar la API programática en lugar de las anotaciones.
     */
    @CircuitBreaker(name = "emailService", fallbackMethod = "fallbackEmail")
    @Retry(name = "emailService", fallbackMethod = "fallbackEmail")
    public EmailSendResult sendEmail(EmailRequest request) {
        return restClient.post()
                .uri("/api/send")
                .body(request)
                .retrieve()
                .body(EmailSendResult.class);
    }

    /**
     * Este fallback se invoca cuando:
     * a) El CB rechaza la llamada (CallNotPermittedException)
     * b) Retry agota todos los intentos
     * La firma debe aceptar Throwable para capturar ambos casos.
     */
    private EmailSendResult fallbackEmail(EmailRequest request, Throwable ex) {
        // Encolar en base de datos para reintento asíncrono posterior
        // (implementación de cola omitida por brevedad)
        return new EmailSendResult("QUEUED_FOR_RETRY", request.to(),
                "Notificación en cola por fallo transitorio: " + ex.getClass().getSimpleName());
    }
}
```

```java
// Modelos
package com.example.notify.model;

public record EmailRequest(String to, String subject, String body) {}
public record EmailSendResult(String status, String recipient, String message) {}
```

```java
// Excepción de validación — ignorada por el Retry (no es reintentable)
package com.example.notify.exception;

public class InvalidEmailException extends RuntimeException {
    public InvalidEmailException(String message) { super(message); }
}

public class NotificationException extends RuntimeException {
    public NotificationException(String message, Throwable cause) { super(message, cause); }
}
```

## Tabla de elementos clave

Propiedades del namespace `resilience4j.retry.instances.[name]` que controlan el comportamiento de Retry.

| Propiedad                           | Tipo                  | Default     | Descripción                                                               |
|-------------------------------------|-----------------------|-------------|---------------------------------------------------------------------------|
| `max-attempts`                      | int                   | `3`         | Número máximo de intentos (incluye el primero)                            |
| `wait-duration`                     | Duration              | `500ms`     | Tiempo base de espera entre reintentos                                    |
| `enable-exponential-backoff`        | boolean               | `false`     | Activa backoff exponencial; multiplica `wait-duration` en cada intento    |
| `exponential-backoff-multiplier`    | double                | `1.5`       | Factor de multiplicación del backoff exponencial                          |
| `exponential-max-wait-duration`     | Duration              | —           | Techo máximo del backoff exponencial; evita esperas demasiado largas      |
| `enable-randomized-wait`            | boolean               | `false`     | Activa jitter aleatorio sobre el wait calculado                           |
| `randomized-wait-factor`            | double (0.0–1.0)      | `0.5`       | Factor de variación: 0.5 = ±50% del wait calculado                       |
| `retry-exceptions`                  | lista de clases       | vacía       | Solo estas excepciones activan el reintento; si vacía, todas activan      |
| `ignore-exceptions`                 | lista de clases       | vacía       | Excepciones que no activan reintento (lanzadas directamente)              |
| `retry-on-result-predicate`         | clase `Predicate<T>`  | —           | Predicate que devuelve true para resultados que deben reintentarse        |
| `retry-on-exception-predicate`      | clase `Predicate<T>`  | —           | Predicate que devuelve true para excepciones que deben reintentarse       |

## Buenas y malas prácticas

**Hacer:**

- Usar backoff exponencial con jitter (`enable-randomized-wait: true`) siempre que el servicio remoto pueda estar bajo carga. El jitter distribuye los reintentos de múltiples clientes y reduce el efecto thundering herd.
- Configurar `retry-exceptions` explícitamente para excluir excepciones de validación, autenticación y negocio. Solo los errores de infraestructura (IOException, ConnectException, 5xx) son genuinamente reintentables.
- Combinar Retry con Circuit Breaker, pero entender el orden de aplicación. Con anotaciones, el orden canónico es `Bulkhead > CircuitBreaker > RateLimiter > TimeLimiter > Retry` (Retry es el más interior). Esto significa que Retry agota sus intentos antes de que el CircuitBreaker cuente un fallo.
- Configurar `exponential-max-wait-duration` para evitar que el backoff crezca indefinidamente en servicios con muchos reintentos configurados. Un techo de 30 segundos es razonable para la mayoría de los casos.

**Evitar:**

- No usar Retry en operaciones que no son idempotentes. Crear un pedido, procesar un pago o enviar un email pueden ser operaciones que el servidor ha completado antes del timeout; reintentar las duplicaría.
- Evitar `max-attempts` altos (> 5) con `wait-duration` largos en rutas síncronas: `3 reintentos × 2s = 6s` de latencia añadida en el peor caso, suficiente para que el cliente HTTP del caller también agote su propio timeout.
- No usar `enable-exponential-backoff: true` sin `exponential-max-wait-duration`. El backoff puede crecer hasta decenas de segundos entre reintentos, haciendo que la respuesta llegue mucho después del timeout del cliente.
- Evitar configurar Retry con las mismas excepciones en `retry-exceptions` e `ignore-exceptions`; `ignore-exceptions` tiene precedencia y el resultado es que la excepción no se reintenta silenciosamente.

---

← [5.5 Fallback: estrategias de respuesta degradada](sc-circuitbreaker-fallback.md) | [Índice](README.md) | [5.7 Bulkhead: aislamiento de recursos con SemaphoreBulkhead y ThreadPoolBulkhead](sc-circuitbreaker-bulkhead.md) →
