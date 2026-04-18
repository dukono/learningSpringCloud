# 4.4 Retry — Configuración y uso

← [4.3 Uso con anotaciones y API programática](sc-circuitbreaker-api.md) | [Índice](README.md) | [4.5 Bulkhead — Semaphore y ThreadPool](sc-circuitbreaker-bulkhead.md) →

---

## Introducción

El patrón Retry resuelve el problema de los fallos transitorios: errores de red momentáneos, timeouts breves o sobrecargas puntuales en los que el downstream recupera su funcionamiento normal en pocos milisegundos o segundos. En lugar de fallar inmediatamente o abrir un Circuit Breaker, Retry vuelve a intentar la llamada un número configurable de veces con una espera entre intentos. Se necesita cuando los fallos son intermitentes y predecibles (sobrecargas momentáneas) y es contraproducente cuando los fallos son sistémicos o permanentes, situación en la que debe actuar el Circuit Breaker.

> [CONCEPTO] Resilience4j Retry y CircuitBreaker son complementarios: Retry gestiona fallos transitorios (pocos intentos, espera corta), CircuitBreaker gestiona fallos sistémicos (cortar el flujo completamente). Se deben combinar con cuidado de orden — ver [4.7 AOP](sc-circuitbreaker-aop.md).

## Estrategias de backoff disponibles

Resilience4j soporta tres estrategias de espera entre reintentos. La elección de la estrategia tiene impacto directo en la carga del servicio downstream durante un período de fallo.

El backoff fijo espera siempre el mismo tiempo (`waitDuration`) entre intentos. Es simple y predecible pero puede causar thundering herd si muchos clientes reintentan a la vez. El backoff exponencial multiplica el tiempo de espera con cada intento, reduciendo la contención. El backoff aleatorio añade jitter al tiempo de espera para distribuir los reintentos en el tiempo.

| Estrategia | Propiedad clave | Comportamiento |
|------------|-----------------|---------------|
| Fijo | `wait-duration` | Espera siempre N ms entre intentos |
| Exponencial | `enable-exponential-backoff: true` + `exponential-backoff-multiplier` | Espera: `waitDuration * multiplier^intento` |
| Aleatorio | `enable-randomized-wait: true` + `randomized-wait-factor` | Jitter: `waitDuration ± (factor * waitDuration)` |

## Ejemplo central

El ejemplo muestra la configuración completa de Retry con backoff exponencial, excepciones filtradas, y uso con anotación y API programática:

```java
package com.example.notification;

import io.github.resilience4j.retry.annotation.Retry;
import io.github.resilience4j.retry.Retry.EventPublisher;
import io.github.resilience4j.retry.RetryRegistry;
import org.springframework.stereotype.Service;

@Service
public class NotificationService {

    private final RetryRegistry retryRegistry;
    private final ExternalEmailClient emailClient;

    public NotificationService(RetryRegistry retryRegistry,
                                ExternalEmailClient emailClient) {
        this.retryRegistry = retryRegistry;
        this.emailClient = emailClient;

        // Registrar listener para contar reintentos
        io.github.resilience4j.retry.Retry retry =
            retryRegistry.retry("emailService");
        retry.getEventPublisher().onRetry(event ->
            System.out.printf("Retry attempt %d for '%s': %s%n",
                event.getNumberOfRetryAttempts(),
                event.getName(),
                event.getLastThrowable().getMessage()));
    }

    // @Retry se aplica DENTRO del @CircuitBreaker (el más interno)
    // Si los 3 intentos fallan, el fallo se cuenta como UNO en el CircuitBreaker
    @Retry(name = "emailService", fallbackMethod = "sendEmailFallback")
    public void sendEmail(EmailRequest request) {
        emailClient.send(request);
    }

    public void sendEmailFallback(EmailRequest request, Throwable ex) {
        // Guardar en cola para reenvío posterior
        System.err.println("Email delivery failed after retries: " + ex.getMessage());
    }

    // Uso programático con RetryRegistry
    public void sendEmailProgrammatic(EmailRequest request) {
        io.github.resilience4j.retry.Retry retry =
            retryRegistry.retry("emailService");
        io.github.resilience4j.retry.Retry.decorateRunnable(retry,
            () -> emailClient.send(request)).run();
    }
}
```

Configuración YAML completa con las tres estrategias:

```yaml
resilience4j:
  retry:
    instances:
      emailService:
        max-attempts: 3
        wait-duration: 500ms
        enable-exponential-backoff: true
        exponential-backoff-multiplier: 2
        exponential-max-wait-duration: 10s
        # Estas excepciones disparan el reintento
        retry-exceptions:
          - java.io.IOException
          - java.util.concurrent.TimeoutException
        # Estas excepciones NO disparan el reintento (pasan directamente al fallback/CB)
        ignore-exceptions:
          - com.example.exceptions.BusinessException

      paymentService:
        max-attempts: 5
        wait-duration: 200ms
        enable-randomized-wait: true
        randomized-wait-factor: 0.5   # Jitter: ±50% del waitDuration
        retry-exceptions:
          - org.springframework.web.client.HttpServerErrorException$ServiceUnavailable
```

## Tabla de propiedades Retry

| Propiedad YAML | API programática | Default | Descripción |
|----------------|-----------------|---------|-------------|
| `max-attempts` | `maxAttempts(int)` | 3 | Número total de intentos (1 = sin retry) |
| `wait-duration` | `waitDuration(Duration)` | 500ms | Tiempo de espera entre intentos |
| `enable-exponential-backoff` | `enableExponentialBackoff()` | false | Activa backoff exponencial |
| `exponential-backoff-multiplier` | `exponentialBackoffMultiplier(double)` | 1.5 | Factor multiplicador del backoff |
| `exponential-max-wait-duration` | `exponentialMaxWaitDuration(Duration)` | — | Techo máximo del backoff exponencial |
| `enable-randomized-wait` | `enableRandomizedWait()` | false | Activa jitter aleatorio |
| `randomized-wait-factor` | `randomizedWaitFactor(double)` | 0.5 | Factor de jitter (0.0 a 1.0) |
| `retry-exceptions` | `retryExceptions(Class...)` | empty | Excepciones que disparan retry |
| `ignore-exceptions` | `ignoreExceptions(Class...)` | empty | Excepciones que NO disparan retry |

> [EXAMEN] `max-attempts` cuenta el intento inicial. `max-attempts=3` significa: 1 intento original + 2 reintentos. Si todos fallan, se invoca el `fallbackMethod`.

> [ADVERTENCIA] No configurar nunca `retry-exceptions` con excepciones permanentes como `HttpClientErrorException` (4xx). Un `404 Not Found` nunca se resolverá con reintentos y solo añade latencia y carga.

## Interacción Retry + CircuitBreaker

Cuando se combinan `@Retry` y `@CircuitBreaker` en el mismo método, el orden de los aspectos determina el comportamiento. Por defecto, Retry es el aspecto más interno y CircuitBreaker es más externo. Esto significa que Retry agota todos sus intentos primero, y solo cuando todos fallan el CircuitBreaker cuenta ese fallo como uno en su sliding window.

```
Llamada → CircuitBreaker → Retry → Método protegido
                          [intento 1: falla]
                          [intento 2: falla]
                          [intento 3: falla]
             ← cuenta 1 fallo en CB ←
```

Si el orden fuera invertido (Retry externo, CB interno), cada intento individual contaría como fallo en el CB, lo que abriría el circuito más rápido. Ver [4.7 AOP](sc-circuitbreaker-aop.md) para el orden completo de aspectos.

## Buenas y malas prácticas

**Buenas prácticas:**
- Usar `retryOnResultPredicate` para reintentar en respuestas con cuerpo pero status code incorrecto (ej: respuesta 200 con campo `status: "PENDING"`).
- Combinar backoff exponencial con `exponentialMaxWaitDuration` para evitar waits infinitos.
- Añadir jitter (`enable-randomized-wait`) cuando múltiples instancias del servicio pueden reintentar simultáneamente.

**Malas prácticas:**
- Reintentar errores 4xx (BusinessException, validaciones): son errores del cliente, no del servidor.
- Configurar `max-attempts` alto (>5) sin backoff: amplifica la carga sobre el downstream durante un fallo.
- Usar Retry sin Circuit Breaker en servicios con posibilidad de fallo sistémico.

## Verificación y práctica

> [EXAMEN] 1. ¿Cuántas llamadas al downstream se producen con `max-attempts=4` y `wait-duration=200ms` si todas fallan?

> [EXAMEN] 2. Con `enable-exponential-backoff=true`, `wait-duration=100ms` y `exponential-backoff-multiplier=2`, ¿cuánto tiempo espera el Retry entre el intento 1 y el intento 2? ¿Y entre el 2 y el 3?

> [EXAMEN] 3. Si `retry-exceptions` está vacío y `ignore-exceptions` contiene `[IOException]`, ¿qué sucede cuando se lanza una `RuntimeException`?

> [EXAMEN] 4. ¿Cómo se configura un Retry que reintente cuando la respuesta HTTP tiene status 503, usando la propiedad `retry-exceptions`?

> [EXAMEN] 5. Con `@Retry` y `@CircuitBreaker` en el mismo método y el orden por defecto, si Retry agota sus 3 intentos 4 veces seguidas, ¿cuántos fallos se registran en la sliding window del CircuitBreaker?

---

← [4.3 Uso con anotaciones y API programática](sc-circuitbreaker-api.md) | [Índice](README.md) | [4.5 Bulkhead — Semaphore y ThreadPool](sc-circuitbreaker-bulkhead.md) →
