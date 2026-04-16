# 5.4 Configuración YAML avanzada del Circuit Breaker en Resilience4j

← [5.3 API programática de Spring Cloud Circuit Breaker y Customizer](sc-circuitbreaker-api.md) | [Índice](README.md) | [5.5 Fallback: estrategias de respuesta degradada](sc-circuitbreaker-fallback.md) →

## Introducción

La configuración declarativa en `application.yml` es el mecanismo preferido para ajustar el comportamiento del Circuit Breaker en entornos de producción porque externaliza los valores sin requerir recompilación y es compatible con Spring Cloud Config para actualización centralizada. Resilience4j expone un modelo de configuración jerárquico: se pueden definir configuraciones base reutilizables con `configs.[nombre]` y referirlas desde instancias individuales con `base-config`, reduciendo la duplicación cuando muchos servicios comparten una política similar.

Este fichero cubre todas las propiedades relevantes, sus valores por defecto, efectos operativos y los patrones de herencia de configuración. La distinción entre `recordExceptions`, `ignoreExceptions` y `recordResultPredicate` es especialmente importante y fuente frecuente de errores de configuración en producción.

> [PREREQUISITO] El modelo de estados (CLOSED/OPEN/HALF_OPEN) y el concepto de ventana deslizante se explican en 5.1. La API programática equivalente a estas propiedades YAML se explica en 5.3.

## Representación visual

El modelo jerárquico de configuración de Resilience4j permite definir configuraciones base que heredan múltiples instancias:

```
resilience4j:
  circuitbreaker:
    configs:
      │
      ├── default          ← configuración base para todos
      │     slidingWindowSize: 20
      │     failureRateThreshold: 50
      │
      └── strict           ← configuración base para servicios críticos
            slidingWindowSize: 10
            failureRateThreshold: 30
            waitDurationInOpenState: 60s

    instances:
      │
      ├── paymentService   ← base-config: strict + sobrescribe waitDuration
      │     base-config: strict
      │     wait-duration-in-open-state: 120s
      │
      ├── inventoryService ← base-config: default (comportamiento estándar)
      │     base-config: default
      │
      └── notificationService ← sin base-config → usa defaults de Resilience4j
```

El proceso de resolución de propiedades para una instancia sigue este orden de precedencia (mayor prioridad primero):

```
1. Propiedades declaradas en instances.[name].*  (más específico)
2. Propiedades de la configuración base (base-config)
3. Defaults de Resilience4j                      (menos específico)
```

## Ejemplo central

Configuración completa para un servicio de e-commerce con tres instancias de Circuit Breaker con diferentes perfiles de riesgo: pagos (crítico), inventario (estándar) y notificaciones (tolerante a fallos).

```xml
<!-- pom.xml — dependencias (sin cambios respecto a setup) -->
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-aop</artifactId>
    </dependency>
</dependencies>
```

```yaml
# application.yml — configuración YAML completa con herencia
resilience4j:
  circuitbreaker:
    # Configuraciones base reutilizables
    configs:
      default:
        sliding-window-type: COUNT_BASED
        sliding-window-size: 20
        minimum-number-of-calls: 5
        failure-rate-threshold: 50
        slow-call-rate-threshold: 100         # deshabilitado efectivamente
        slow-call-duration-threshold: 60s
        wait-duration-in-open-state: 30s
        automatic-transition-from-open-to-half-open-enabled: true
        permitted-number-of-calls-in-half-open-state: 3
        max-wait-duration-in-half-open-state: 0    # sin límite
        record-exceptions:
          - java.io.IOException
          - java.util.concurrent.TimeoutException
          - org.springframework.web.client.HttpServerErrorException
        ignore-exceptions:
          - com.example.shop.exception.BusinessValidationException

      strict:
        sliding-window-type: TIME_BASED
        sliding-window-size: 10               # 10 segundos
        minimum-number-of-calls: 3
        failure-rate-threshold: 25
        slow-call-rate-threshold: 50
        slow-call-duration-threshold: 1s
        wait-duration-in-open-state: 60s
        automatic-transition-from-open-to-half-open-enabled: true
        permitted-number-of-calls-in-half-open-state: 2
        max-wait-duration-in-half-open-state: 30s
        record-exceptions:
          - java.io.IOException
          - org.springframework.web.client.HttpServerErrorException
          - java.util.concurrent.TimeoutException

    # Instancias específicas
    instances:
      paymentService:
        base-config: strict
        wait-duration-in-open-state: 120s   # sobrescribe el valor de strict
        record-result-predicate: com.example.shop.cb.PaymentResultPredicate

      inventoryService:
        base-config: default

      notificationService:
        # Sin base-config: usa defaults de Resilience4j
        sliding-window-type: COUNT_BASED
        sliding-window-size: 50
        minimum-number-of-calls: 10
        failure-rate-threshold: 80           # muy tolerante
        wait-duration-in-open-state: 10s     # recuperación rápida
        automatic-transition-from-open-to-half-open-enabled: true
        permitted-number-of-calls-in-half-open-state: 5

management:
  endpoints:
    web:
      exposure:
        include: health,circuitbreakers,circuitbreakerevents
  health:
    circuitbreakers:
      enabled: true
```

```java
// PaymentResultPredicate.java — apertura por resultado (no solo por excepción)
package com.example.shop.cb;

import io.github.resilience4j.core.functions.CheckedFunction;
import java.util.function.Predicate;

/**
 * Abre el circuito cuando el servicio de pagos devuelve un status SYSTEM_ERROR,
 * incluso si la respuesta HTTP fue 200 OK.
 * La clase debe implementar Predicate<T> donde T es el tipo de retorno del método.
 */
public class PaymentResultPredicate implements Predicate<com.example.shop.model.PaymentResponse> {

    @Override
    public boolean test(com.example.shop.model.PaymentResponse response) {
        // true = contar este resultado como fallo para el circuito
        return "SYSTEM_ERROR".equals(response.status());
    }
}
```

```java
// PaymentResponse.java
package com.example.shop.model;

public record PaymentResponse(String status, String transactionId, String message) {}
```

```java
// OrderController.java — uso con anotación AOP (requiere spring-boot-starter-aop)
package com.example.shop.controller;

import com.example.shop.model.PaymentResponse;
import com.example.shop.service.PaymentService;
import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/orders")
public class OrderController {

    private final PaymentService paymentService;

    public OrderController(PaymentService paymentService) {
        this.paymentService = paymentService;
    }

    @PostMapping("/{orderId}/pay")
    public PaymentResponse processPayment(@PathVariable String orderId) {
        return paymentService.processPayment(orderId);
    }
}
```

```java
// PaymentService.java — método protegido por CB con nombre de instancia YAML
package com.example.shop.service;

import com.example.shop.model.PaymentResponse;
import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestClient;

@Service
public class PaymentService {

    private final RestClient restClient;

    public PaymentService(RestClient.Builder builder) {
        this.restClient = builder.baseUrl("http://payment-service").build();
    }

    @CircuitBreaker(name = "paymentService", fallbackMethod = "fallbackPayment")
    public PaymentResponse processPayment(String orderId) {
        return restClient.post()
                .uri("/api/process/{id}", orderId)
                .retrieve()
                .body(PaymentResponse.class);
    }

    private PaymentResponse fallbackPayment(String orderId, Throwable ex) {
        return new PaymentResponse("PENDING", null,
                "Pago en cola de reintento: " + ex.getClass().getSimpleName());
    }
}
```

## Tabla de elementos clave

Todas las propiedades del namespace `resilience4j.circuitbreaker.instances.[name]` que un profesional senior debe conocer.

| Propiedad                                      | Tipo                        | Default         | Descripción                                                                      |
|------------------------------------------------|-----------------------------|-----------------|----------------------------------------------------------------------------------|
| `sliding-window-type`                          | `COUNT_BASED`, `TIME_BASED` | `COUNT_BASED`   | Unidad de la ventana deslizante                                                  |
| `sliding-window-size`                          | int                         | `100`           | Tamaño de la ventana (llamadas o segundos)                                       |
| `minimum-number-of-calls`                      | int                         | `100`           | Llamadas mínimas antes de evaluar umbrales; clave para tráfico bajo              |
| `failure-rate-threshold`                       | float (0–100)               | `50`            | % de fallos que abre el circuito                                                 |
| `slow-call-rate-threshold`                     | float (0–100)               | `100`           | % de llamadas lentas que abre el circuito (`100` = deshabilitado)                |
| `slow-call-duration-threshold`                 | Duration                    | `60s`           | Umbral de duración para considerar una llamada lenta                             |
| `wait-duration-in-open-state`                  | Duration                    | `60s`           | Tiempo en estado OPEN antes de pasar a HALF_OPEN                                 |
| `automatic-transition-from-open-to-half-open-enabled` | boolean            | `false`         | Transición automática OPEN → HALF_OPEN sin esperar llamada                       |
| `permitted-number-of-calls-in-half-open-state` | int                         | `10`            | Llamadas de prueba en HALF_OPEN                                                  |
| `max-wait-duration-in-half-open-state`         | Duration                    | `0` (sin límite)| Tiempo máximo en HALF_OPEN; `0` = esperar indefinidamente                        |
| `record-exceptions`                            | lista de clases             | vacía (todas)   | Solo estas excepciones cuentan como fallo; si vacía, todas las excepciones       |
| `ignore-exceptions`                            | lista de clases             | vacía           | Excepciones ignoradas (ni fallo ni éxito); tienen precedencia sobre record       |
| `record-result-predicate`                      | clase `Predicate<T>`        | —               | Predicate que devuelve `true` para resultados que deben contar como fallo        |
| `base-config`                                  | nombre de config            | —               | Nombre de la configuración base en `configs.[nombre]` de la que hereda           |

## Buenas y malas prácticas

**Hacer:**

- Definir siempre una configuración `default` en `configs.default` y referenciarla con `base-config: default` en todas las instancias. Esto garantiza un comportamiento predefinido explícito y evita depender de los defaults de Resilience4j que tienen `minimumNumberOfCalls: 100`.
- Usar `record-result-predicate` para servicios que devuelven errores de negocio con HTTP 200. Sin él, el circuito solo reacciona a excepciones, ignorando respuestas que indican problemas reales del servicio.
- Revisar `ignore-exceptions` en cada instancia. Las excepciones de dominio (validación, not-found) que el servicio remoto lanza intencionalmente no deben contar como fallos del circuito; ignorarlas evita aperturas por lógica de negocio correcta.
- Activar `automatic-transition-from-open-to-half-open-enabled: true` en todos los servicios. Sin esto, el circuito permanece OPEN hasta que llega una llamada real para activar la transición, lo que puede alargar el tiempo de degradación innecesariamente.

**Evitar:**

- No dejar `minimum-number-of-calls` en su valor por defecto (`100`) en servicios de bajo tráfico. Con 2 llamadas por minuto tardaría 50 minutos en acumular suficientes datos para evaluar; el circuito nunca funcionará.
- Evitar `slow-call-duration-threshold` sin ajustar `slow-call-rate-threshold`. Si no se modifica el threshold, el 100% por defecto hace que el detector de llamadas lentas nunca abra el circuito aunque el 99% de las llamadas sean lentas.
- No mezclar configuración YAML con Customizer programático en la misma instancia sin documentar la precedencia. Los Customizer tienen mayor prioridad que el YAML cuando configuran la misma instancia a través de `Resilience4JCircuitBreakerFactory`, pero no cuando se usan directamente vía `CircuitBreakerRegistry`.

## Variante: configuración con Retry compartiendo la misma base

Resilience4j permite definir configuraciones base también para Retry, Bulkhead y RateLimiter usando el mismo patrón de herencia.

La siguiente tabla muestra cómo se mapean los namespaces para cada componente:

| Componente         | Namespace de configuración                            | Namespace de instancias                              |
|--------------------|-------------------------------------------------------|------------------------------------------------------|
| Circuit Breaker    | `resilience4j.circuitbreaker.configs.[name]`          | `resilience4j.circuitbreaker.instances.[name]`       |
| Retry              | `resilience4j.retry.configs.[name]`                   | `resilience4j.retry.instances.[name]`                |
| Bulkhead           | `resilience4j.bulkhead.configs.[name]`                | `resilience4j.bulkhead.instances.[name]`             |
| Thread Pool BH     | `resilience4j.thread-pool-bulkhead.configs.[name]`    | `resilience4j.thread-pool-bulkhead.instances.[name]` |
| RateLimiter        | `resilience4j.ratelimiter.configs.[name]`             | `resilience4j.ratelimiter.instances.[name]`          |
| TimeLimiter        | `resilience4j.timelimiter.configs.[name]`             | `resilience4j.timelimiter.instances.[name]`          |

---

← [5.3 API programática de Spring Cloud Circuit Breaker y Customizer](sc-circuitbreaker-api.md) | [Índice](README.md) | [5.5 Fallback: estrategias de respuesta degradada](sc-circuitbreaker-fallback.md) →
