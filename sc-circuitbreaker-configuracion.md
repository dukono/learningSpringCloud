# 4.2 Circuit Breaker — Configuración completa

← [4.1 Estados y Transiciones](sc-circuitbreaker-estados.md) | [Índice](README.md) | [4.3 Uso con anotaciones y API programática](sc-circuitbreaker-api.md) →

---

## Introducción

Una vez comprendidos los estados del Circuit Breaker, el paso siguiente es conocer con precisión todas las propiedades que controlan su comportamiento. Este fichero es una referencia completa de `CircuitBreakerConfig`: qué hace cada propiedad, su valor por defecto, su rango válido y cómo se especifica tanto en YAML como de forma programática. Conocer esta configuración es imprescindible porque los valores por defecto de Resilience4j están calibrados para cargas altas de producción y son inadecuados para tests o para servicios de tráfico bajo.

> [PREREQUISITO] Este fichero presupone conocimiento de los estados CLOSED/OPEN/HALF_OPEN documentados en [4.1 Estados y Transiciones](sc-circuitbreaker-estados.md).

## Tabla de propiedades completa

La tabla siguiente cubre todas las propiedades de `CircuitBreakerConfig` con sus valores por defecto y su significado. Las propiedades de configuración YAML usan kebab-case; la API programática usa camelCase.

| Propiedad YAML | API programática | Default | Descripción |
|----------------|-----------------|---------|-------------|
| `failure-rate-threshold` | `failureRateThreshold(float)` | 50 | % de llamadas fallidas para abrir el CB |
| `slow-call-rate-threshold` | `slowCallRateThreshold(float)` | 100 | % de llamadas lentas para abrir el CB |
| `slow-call-duration-threshold` | `slowCallDurationThreshold(Duration)` | 60s | Duración a partir de la cual una llamada se considera lenta |
| `wait-duration-in-open-state` | `waitDurationInOpenState(Duration)` | 60s | Tiempo que el CB permanece en OPEN antes de ir a HALF_OPEN |
| `permitted-number-of-calls-in-half-open-state` | `permittedNumberOfCallsInHalfOpenState(int)` | 10 | Llamadas de prueba permitidas en HALF_OPEN |
| `sliding-window-type` | `slidingWindowType(SlidingWindowType)` | COUNT_BASED | Tipo de ventana deslizante |
| `sliding-window-size` | `slidingWindowSize(int)` | 100 | Tamaño de la ventana (llamadas o segundos) |
| `minimum-number-of-calls` | `minimumNumberOfCalls(int)` | 100 | Mínimo de llamadas antes de evaluar tasa de fallos |
| `automatic-transition-from-open-to-half-open-enabled` | `automaticTransitionFromOpenToHalfOpenEnabled(boolean)` | false | Si `true`, transiciona a HALF_OPEN automáticamente al expirar `waitDuration` |
| `record-exceptions` | `recordExceptions(Class...)` | empty (todas) | Excepciones que cuentan como fallo |
| `ignore-exceptions` | `ignoreExceptions(Class...)` | empty (ninguna) | Excepciones que NO cuentan ni como fallo ni como éxito |
| `record-failure-predicate` | `recordException(Predicate<Throwable>)` | — | Predicado personalizado para decidir si contar como fallo |

> [CONCEPTO] Cuando `record-exceptions` está vacío (por defecto), **todas** las excepciones se cuentan como fallo. Si se configura al menos una excepción en `record-exceptions`, entonces SOLO esas excepciones cuentan como fallo; el resto no se registra.

> [CONCEPTO] `ignore-exceptions` tiene prioridad sobre `record-exceptions`. Si una excepción aparece en ambas listas, se ignora (no cuenta ni como fallo ni como éxito).

## Ejemplo central

El ejemplo muestra la configuración completa vía YAML con herencia de configuración `default`, y la configuración equivalente mediante el builder programático:

```yaml
# application.yml — Configuración YAML con perfil default y override por instancia
resilience4j:
  circuitbreaker:
    configs:
      default:
        sliding-window-type: COUNT_BASED
        sliding-window-size: 50
        minimum-number-of-calls: 10
        failure-rate-threshold: 50
        slow-call-rate-threshold: 80
        slow-call-duration-threshold: 3s
        wait-duration-in-open-state: 30s
        permitted-number-of-calls-in-half-open-state: 5
        automatic-transition-from-open-to-half-open-enabled: true
        record-exceptions:
          - java.io.IOException
          - java.util.concurrent.TimeoutException
          - org.springframework.web.client.HttpServerErrorException
        ignore-exceptions:
          - com.example.exceptions.BusinessValidationException
    instances:
      inventoryService:
        base-config: default         # Hereda de "default"
        failure-rate-threshold: 60   # Sobreescribe solo este valor
        wait-duration-in-open-state: 60s
      paymentService:
        base-config: default
        sliding-window-size: 20
        minimum-number-of-calls: 5
```

La misma configuración mediante builder programático para `paymentService`:

```java
package com.example.config;

import io.github.resilience4j.circuitbreaker.CircuitBreakerConfig;
import io.github.resilience4j.circuitbreaker.CircuitBreakerRegistry;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import java.io.IOException;
import java.time.Duration;

@Configuration
public class Resilience4jConfig {

    @Bean
    public CircuitBreakerRegistry circuitBreakerRegistry() {
        // Configuración default compartida
        CircuitBreakerConfig defaultConfig = CircuitBreakerConfig.custom()
            .slidingWindowType(CircuitBreakerConfig.SlidingWindowType.COUNT_BASED)
            .slidingWindowSize(50)
            .minimumNumberOfCalls(10)
            .failureRateThreshold(50f)
            .slowCallRateThreshold(80f)
            .slowCallDurationThreshold(Duration.ofSeconds(3))
            .waitDurationInOpenState(Duration.ofSeconds(30))
            .permittedNumberOfCallsInHalfOpenState(5)
            .automaticTransitionFromOpenToHalfOpenEnabled(true)
            .recordExceptions(IOException.class)
            .ignoreExceptions(IllegalArgumentException.class)
            .build();

        // Configuración específica para paymentService
        CircuitBreakerConfig paymentConfig = CircuitBreakerConfig.from(defaultConfig)
            .slidingWindowSize(20)
            .minimumNumberOfCalls(5)
            .build();

        CircuitBreakerRegistry registry = CircuitBreakerRegistry.of(defaultConfig);
        registry.circuitBreaker("paymentService", paymentConfig);
        return registry;
    }
}
```

## Jerarquía de configuración

La resolución de configuración sigue este orden de prioridad (mayor prioridad primero):

1. Configuración programática aplicada directamente a la instancia via `registry.circuitBreaker(name, config)`
2. Propiedad `base-config` de la instancia en YAML con overrides específicos
3. Configuración `default` en `resilience4j.circuitbreaker.configs.default`
4. Valores por defecto de `CircuitBreakerConfig.ofDefaults()`

> [ADVERTENCIA] Las configuraciones YAML y programáticas pueden coexistir, pero si se crea un bean `CircuitBreakerRegistry` personalizado, Spring Boot autoconfiguration NO sobreescribe ese bean. El YAML de `resilience4j.*` solo tiene efecto cuando Spring Boot autoconfigura el registry, lo que implica que con un `@Bean CircuitBreakerRegistry` propio hay que gestionar toda la configuración programáticamente.

## Buenas y malas prácticas

**Buenas prácticas:**
- Definir siempre un perfil `default` con los valores base del proyecto y solo sobreescribir por instancia lo necesario.
- Reducir `minimum-number-of-calls` a 5-10 en tests para que el CB evalúe rápidamente.
- Configurar `record-exceptions` explícitamente para evitar que excepciones de negocio (validaciones, `404 Not Found`) abran el CB.
- Usar `slow-call-duration-threshold` alineado con los SLA del servicio downstream.

**Malas prácticas:**
- Dejar `minimum-number-of-calls=100` (default) en servicios con poco tráfico: el CB nunca evaluará.
- Ignorar `slow-call-rate-threshold`: un servicio que responde lentamente degrada el sistema aunque no lance excepciones.
- Mezclar configuración YAML y programática sobre el mismo bean registry: produce resultados impredecibles.

## Verificación y práctica

> [EXAMEN] 1. Un CircuitBreaker tiene `record-exceptions: [IOException]` y `ignore-exceptions: [IOException]`. ¿Qué ocurre cuando se lanza un `IOException`?

> [EXAMEN] 2. ¿Cuál es la diferencia entre `sliding-window-size` cuando `sliding-window-type` es COUNT_BASED frente a TIME_BASED?

> [EXAMEN] 3. ¿Qué propiedad controla cuántas llamadas de prueba se permiten en el estado HALF_OPEN, y cuál es su valor por defecto?

> [EXAMEN] 4. Si un servicio tiene `minimum-number-of-calls=100` y recibe exactamente 99 llamadas, todas fallidas, ¿en qué estado estará el CircuitBreaker?

> [EXAMEN] 5. Describe la diferencia entre `base-config: default` en YAML y `CircuitBreakerConfig.from(otherConfig)` en la API programática.

---

← [4.1 Estados y Transiciones](sc-circuitbreaker-estados.md) | [Índice](README.md) | [4.3 Uso con anotaciones y API programática](sc-circuitbreaker-api.md) →
