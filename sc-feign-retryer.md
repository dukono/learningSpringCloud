# 4.10 Retryer — política de reintentos nativa, personalización y conflicto con Resilience4j

<- [4.9 Fallback y Circuit Breaker](sc-feign-fallback.md) | [Índice](README.md) | [4.11 Cliente HTTP subyacente](sc-feign-cliente-http.md) ->

---

## Introducción

Feign tiene su propio mecanismo de reintentos, separado e independiente de Resilience4j Retry. El `Retryer` de Feign actúa justo después de que el `ErrorDecoder` lanza una `RetryableException`: intercepta esa excepción, aplica un backoff y vuelve a ejecutar la llamada HTTP desde cero. Por defecto, Feign tiene el reintento desactivado (`Retryer.NEVER_RETRY`), lo que significa que cualquier error se propaga inmediatamente al llamador.

El problema más crítico de este módulo es el antipatrón del doble reintento: cuando Feign Retryer y Resilience4j Retry están activos simultáneamente sobre el mismo cliente, cada fallo puede generar N reintentos de Feign, y Resilience4j puede reintentar el conjunto entero M veces. El resultado es N×M llamadas al servicio remoto por cada fallo original, multiplicando la carga sobre un servicio ya degradado y pudiendo hacer que las operaciones no idempotentes (POST, PATCH) se ejecuten múltiples veces.

> [ADVERTENCIA] Si Resilience4j Retry está activo para un cliente Feign, desactivar el Retryer de Feign explícitamente (`Retryer.NEVER_RETRY`) para evitar el antipatrón de doble reintento. Usar uno solo: preferir Resilience4j Retry por su mayor expresividad (backoff exponencial, predicados de excepción, métricas).

---

## Diagrama: orden de ejecución de reintentos

El siguiente diagrama muestra dónde actúa cada mecanismo de reintento en el pipeline de llamada.

```
 Llamada al método @FeignClient
         │
         ▼
 Resilience4j Retry (capa externa)
   └─ intenta la llamada completa de Feign
         │
         ▼
     Feign proxy → HTTP → Servicio remoto
         │
         ├─ Éxito → resultado al llamador
         └─ Error → ErrorDecoder
                 │
                 ├─ RetryableException → Feign Retryer (capa interna)
                 │       │
                 │       ├─ intentos < maxAttempts → espera + reintenta Feign
                 │       └─ intentos == maxAttempts → propaga excepción
                 │                   │
                 │                   ▼
                 │             Resilience4j Retry
                 │             ¿reintento externo?
                 │               Sí → vuelve a intentar TODA la llamada Feign
                 │               (incluyendo los N reintentos internos de Feign)
                 │
                 └─ Excepción no retryable → se propaga directamente
                                 │
                                 ▼
                           Resilience4j Retry
                           ¿coincide predicado?
                             Sí → reintento externo
```

---

## Ejemplo central

El siguiente ejemplo muestra la configuración de `Retryer.Default`, un `Retryer` personalizado y la configuración correcta cuando Resilience4j Retry está activo.

**Retryer.Default — configuración de reintentos integrada en Feign:**

```java
package com.example.orderservice.config;

import feign.Retryer;
import org.springframework.context.annotation.Bean;

public class CatalogFeignRetryConfig {

    /**
     * Retryer.Default(period, maxPeriod, maxAttempts):
     *   - period: tiempo inicial de espera entre reintentos (ms)
     *   - maxPeriod: tiempo máximo de espera entre reintentos (ms); el backoff crece exponencialmente
     *   - maxAttempts: número TOTAL de intentos (incluye el primero; maxAttempts=3 → 2 reintentos)
     *
     * Solo activo si el ErrorDecoder lanza RetryableException.
     * Con maxAttempts=3, period=100ms y maxPeriod=1000ms:
     *   intento 1 → fallo → espera ~100ms
     *   intento 2 → fallo → espera ~200ms
     *   intento 3 → fallo → RetryException al llamador
     */
    @Bean
    public Retryer feignRetryer() {
        return new Retryer.Default(100, 1_000, 3);
    }
}
```

**Retryer personalizado — con lógica de backoff propia:**

```java
package com.example.orderservice.feign;

import feign.RetryableException;
import feign.Retryer;

/**
 * Retryer personalizado con backoff fijo (sin crecimiento exponencial)
 * y límite de intentos configurable.
 *
 * Útil cuando el servicio remoto es sensible a ráfagas de reintentos
 * y se prefiere un intervalo constante.
 */
public class FixedBackoffRetryer implements Retryer {

    private final int maxAttempts;
    private final long backoffMillis;
    private int attempt;

    public FixedBackoffRetryer(int maxAttempts, long backoffMillis) {
        this.maxAttempts   = maxAttempts;
        this.backoffMillis = backoffMillis;
        this.attempt       = 0;
    }

    @Override
    public void continueOrPropagate(RetryableException e) {
        if (++attempt >= maxAttempts) {
            throw e;  // agotados los intentos; propaga la excepción original
        }
        try {
            Thread.sleep(backoffMillis);
        } catch (InterruptedException ie) {
            Thread.currentThread().interrupt();
            throw e;
        }
    }

    /**
     * Feign llama a clone() al inicio de cada petición para obtener
     * un Retryer con el estado reiniciado. La implementación por referencia
     * no funciona correctamente si el mismo Retryer se reutiliza entre peticiones.
     */
    @Override
    public Retryer clone() {
        return new FixedBackoffRetryer(maxAttempts, backoffMillis);
    }
}
```

**Configuración correcta cuando Resilience4j Retry está activo — desactivar Feign Retryer:**

```java
package com.example.orderservice.config;

import feign.Retryer;
import org.springframework.context.annotation.Bean;

/**
 * Cuando Resilience4j Retry gestiona los reintentos, Feign Retryer debe estar
 * explícitamente desactivado para evitar el doble reintento N×M.
 */
public class CatalogFeignNoRetryConfig {

    @Bean
    public Retryer feignRetryer() {
        return Retryer.NEVER_RETRY;   // desactiva reintentos de Feign
    }
}
```

**application.yml — configuración de Resilience4j Retry para el mismo cliente:**

```yaml
resilience4j:
  retry:
    instances:
      catalogRead:                        # coincide con contextId del @FeignClient
        max-attempts: 3
        wait-duration: 200ms
        retry-exceptions:
          - feign.RetryableException      # solo reintenta excepciones retryable
          - java.net.ConnectException
        ignore-exceptions:
          - com.example.orderservice.exception.ProductNotFoundException  # 404 no reintenta
```

> [CONCEPTO] `Retryer.clone()` es el mecanismo que garantiza que cada petición comienza con un contador de reintentos en cero. Si el `Retryer` es un bean singleton (registrado con `@Bean`) y no implementa `clone()` correctamente, los reintentos de una petición anterior pueden contaminar el estado de la siguiente.

---

## Tabla de elementos clave

| Elemento | Tipo | Default | Descripción |
|---|---|---|---|
| `Retryer.NEVER_RETRY` | `Retryer` constante | valor por defecto en Spring Cloud | Desactiva reintentos de Feign |
| `Retryer.Default(period, maxPeriod, maxAttempts)` | constructor | — | Reintento con backoff exponencial entre `period` y `maxPeriod` ms |
| `maxAttempts` | `int` | — | Número total de intentos incluido el primero; `3` = 2 reintentos |
| `period` | `long` (ms) | — | Tiempo inicial de espera entre reintentos |
| `maxPeriod` | `long` (ms) | — | Tiempo máximo de espera; el backoff crece hasta este límite |
| `Retryer.continueOrPropagate(e)` | método | — | Llamado por Feign tras cada `RetryableException`; lanza `e` para detener |
| `Retryer.clone()` | método | — | Debe retornar instancia con estado reiniciado; obligatorio en Retryer custom |

---

## Buenas y malas prácticas

**Hacer:**
- Desactivar explícitamente el Retryer de Feign (`Retryer.NEVER_RETRY`) en la configuración del cliente cuando Resilience4j Retry gestiona los reintentos. No confiar en que "el default es NEVER_RETRY": si alguien añade un `Retryer.Default` en la configuración global, todos los clientes heredarán reintentos sin que sea obvio.
- Implementar siempre `clone()` en un `Retryer` personalizado. Sin `clone()`, el contador de intentos no se reinicia entre peticiones y el Retryer puede comportarse incorrectamente tras la primera petición que agota sus intentos.
- Preferir Resilience4j Retry sobre Feign Retryer para nuevos proyectos. Resilience4j ofrece backoff exponencial con jitter, predicados de excepción granulares, métricas Micrometer y endpoint Actuator. El Retryer de Feign no tiene ninguna de estas capacidades.

**Evitar:**
- Activar Feign Retryer para métodos POST o PATCH sin verificar que el servicio remoto es idempotente. Un reintento de un POST puede crear el recurso dos veces; si el servicio no devuelve 409 en duplicados, el reintento pasa inadvertido y los datos quedan inconsistentes.
- Configurar `maxAttempts` alto (> 5) con `period` corto (< 100ms) en un servicio de alto volumen. En un pico de carga, cada petición fallida generará múltiples reintentos rápidos que amplifican la carga sobre el servicio ya degradado, precipitando un fallo total en lugar de permitir recuperación gradual.

---

## Comparación: Feign Retryer vs Resilience4j Retry

| Aspecto | Feign Retryer | Resilience4j Retry |
|---|---|---|
| Activación | Bean `Retryer` en configuración Feign | `@Retry` o propiedades `resilience4j.retry.*` |
| Condición de reintento | Solo `RetryableException` | Predicados de excepción configurables |
| Backoff | Exponencial entre `period` y `maxPeriod` | Fijo, exponencial, aleatorio, con jitter |
| Métricas | No | Sí (Micrometer: `resilience4j.retry.*`) |
| Endpoint Actuator | No | Sí (`/actuator/retries`) |
| Integración con CB | No | Sí (Chain: Retry → CircuitBreaker) |
| Coexistencia | Posible pero peligrosa (N×M reintentos) | Recomendado como única capa de reintento |

> [EXAMEN] En entrevista: "¿Cuántas veces se ejecuta la llamada HTTP si Feign Retryer tiene `maxAttempts=3` y Resilience4j Retry tiene `max-attempts=2` sobre el mismo cliente?" Respuesta: hasta 3×2 = 6 veces. Feign reintentará 3 veces por cada intento de Resilience4j, y Resilience4j hará 2 intentos (1 original + 1 reintento). En la práctica puede ser menos si alguna excepción no coincide con el predicado del reintento externo.

---

<- [4.9 Fallback y Circuit Breaker](sc-feign-fallback.md) | [Índice](README.md) | [4.11 Cliente HTTP subyacente](sc-feign-cliente-http.md) ->
