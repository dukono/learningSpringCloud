# 5.14 Testing del Circuit Breaker, Retry y Bulkhead

← [5.13 Integración con OpenFeign, RestClient y WebClient](sc-circuitbreaker-feign.md) | [Índice (README.md)](README.md) | [6.1 Modelo de programación funcional](sc-stream-modelo-funcional.md) →

---

Los patrones de resiliencia de Resilience4j son difíciles de testear porque su comportamiento correcto depende del estado interno del circuito (CLOSED/OPEN/HALF_OPEN), de contadores que se acumulan con el tiempo, y de decisiones que solo se toman después de varios fallos consecutivos. Un test unitario que llama una sola vez a un método anotado con `@CircuitBreaker` siempre pasa —el circuito está cerrado en el test porque nadie lo ha abierto— y da una cobertura falsa. El testing real requiere: forzar el estado del circuito o simular los fallos que lo llevan a ese estado, verificar los contadores de Retry con métricas, y estresear el Bulkhead con llamadas concurrentes. Existen tres niveles de test progresivamente más fieles.

> [PREREQUISITO] Requiere `spring-cloud-starter-circuitbreaker-resilience4j` y, para tests de integración, `wiremock-spring-boot` o `org.wiremock.integrations:wiremock-spring-boot`. Los tests de Bulkhead concurrente requieren `awaitility` para assertions asíncronos.

## Estrategias de testing de tolerancia a fallos

La elección de estrategia depende de qué aspecto se verifica: la lógica del fallback, la transición de estados, o el comportamiento bajo carga real.

| Estrategia | Contexto Spring | Velocidad | Fidelidad | Qué verifica |
|---|---|---|---|---|
| 1 — Test unitario con CircuitBreakerRegistry | Sin Spring | Muy rápida | Media | Lógica sin AOP, transiciones de estado directas |
| 2 — Test de integración con @SpringBootTest | Con Spring | Media | Alta | AOP, fallback, contadores de Retry, Bulkhead |
| 3 — Test con WireMock | Con Spring + HTTP | Lenta | Máxima | Fallos reales de red, timeouts, cascading failures |

## Estrategia 1: Test unitario con `CircuitBreakerRegistry.ofDefaults()`

Instanciar `CircuitBreakerRegistry` directamente evita cargar el contexto de Spring y permite verificar transiciones de estado manipulando el circuito programáticamente. Es el test más rápido y el más adecuado para verificar la lógica de negocio del fallback.

```xml
<!-- pom.xml -->
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot3</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

```java
package com.example.resilience;

import io.github.resilience4j.circuitbreaker.CircuitBreaker;
import io.github.resilience4j.circuitbreaker.CircuitBreakerConfig;
import io.github.resilience4j.circuitbreaker.CircuitBreakerRegistry;
import io.github.resilience4j.retry.Retry;
import io.github.resilience4j.retry.RetryConfig;
import io.github.resilience4j.retry.RetryRegistry;
import org.junit.jupiter.api.Test;

import java.time.Duration;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.function.Supplier;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class CircuitBreakerUnitTest {

    @Test
    void whenCircuitForcedOpen_thenCallRejectedImmediately() {
        CircuitBreakerConfig config = CircuitBreakerConfig.custom()
            .slidingWindowSize(5)
            .failureRateThreshold(50)
            .build();

        CircuitBreakerRegistry registry = CircuitBreakerRegistry.of(config);
        CircuitBreaker cb = registry.circuitBreaker("test");

        // Forzar estado OPEN directamente sin esperar fallos reales
        cb.transitionToOpenState();

        assertThat(cb.getState()).isEqualTo(CircuitBreaker.State.OPEN);

        // Decorar una llamada que debería rechazarse
        Supplier<String> supplier = CircuitBreaker.decorateSupplier(cb, () -> "ok");
        assertThatThrownBy(supplier::get)
            .isInstanceOf(io.github.resilience4j.circuitbreaker.CallNotPermittedException.class);
    }

    @Test
    void retryExecutesMaxAttemptsOnFailure() {
        AtomicInteger callCount = new AtomicInteger(0);

        RetryConfig config = RetryConfig.custom()
            .maxAttempts(3)
            .waitDuration(Duration.ofMillis(10))
            .build();

        Retry retry = RetryRegistry.of(config).retry("test");

        assertThatThrownBy(() ->
            Retry.decorateCheckedSupplier(retry, () -> {
                callCount.incrementAndGet();
                throw new RuntimeException("simulated failure");
            }).get()
        ).isInstanceOf(RuntimeException.class);

        // 3 intentos: 1 inicial + 2 reintentos
        assertThat(callCount.get()).isEqualTo(3);
    }
}
```

> [CONCEPTO] `transitionToOpenState()` es el mecanismo de test más directo para verificar el comportamiento de un circuito abierto sin necesidad de generar los fallos reales que lo abren. También existe `transitionToHalfOpenState()` y `transitionToClosedState()`.

## Estrategia 2: Test de integración con `@SpringBootTest` y estado forzado

Este nivel verifica que el AOP de Spring aplica los decoradores correctamente, que el fallback se invoca cuando el circuito está abierto, y que los contadores de Retry son visibles a través de Micrometer.

```java
package com.example.resilience;

import io.github.resilience4j.circuitbreaker.CircuitBreaker;
import io.github.resilience4j.circuitbreaker.CircuitBreakerRegistry;
import io.github.resilience4j.bulkhead.Bulkhead;
import io.github.resilience4j.bulkhead.BulkheadRegistry;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.atomic.AtomicInteger;

import static org.assertj.core.api.Assertions.assertThat;
import static org.awaitility.Awaitility.await;
import static java.util.concurrent.TimeUnit.SECONDS;

@SpringBootTest
class ResilienceIntegrationTest {

    @Autowired
    private CircuitBreakerRegistry circuitBreakerRegistry;

    @Autowired
    private BulkheadRegistry bulkheadRegistry;

    @Autowired
    private OrderService orderService; // bean decorado con @CircuitBreaker

    @BeforeEach
    void resetState() {
        // Asegurar estado limpio antes de cada test
        circuitBreakerRegistry.circuitBreaker("orders")
            .transitionToClosedState();
    }

    @Test
    void whenCircuitOpen_thenFallbackInvoked() {
        CircuitBreaker cb = circuitBreakerRegistry.circuitBreaker("orders");
        cb.transitionToOpenState();

        // orderService.getOrder() tiene @CircuitBreaker(name="orders", fallbackMethod="fallback")
        String result = orderService.getOrder("123");

        assertThat(result).isEqualTo("fallback-response");
        assertThat(cb.getState()).isEqualTo(CircuitBreaker.State.OPEN);
    }

    @Test
    void bulkheadRejectsConcurrentCallsAboveLimit() throws InterruptedException {
        Bulkhead bulkhead = bulkheadRegistry.bulkhead("orders");
        int maxConcurrent = bulkhead.getBulkheadConfig().getMaxConcurrentCalls();
        int extraCalls = maxConcurrent + 5;

        AtomicInteger rejected = new AtomicInteger(0);
        CountDownLatch latch = new CountDownLatch(extraCalls);
        ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();

        for (int i = 0; i < extraCalls; i++) {
            executor.submit(() -> {
                try {
                    orderService.getOrder("concurrent");
                } catch (io.github.resilience4j.bulkhead.BulkheadFullException e) {
                    rejected.incrementAndGet();
                } finally {
                    latch.countDown();
                }
            });
        }

        latch.await(5, SECONDS);
        assertThat(rejected.get()).isGreaterThan(0);
    }
}
```

> [ADVERTENCIA] El estado del `CircuitBreaker` persiste entre tests del mismo contexto de Spring Boot (`@SpringBootTest` reutiliza el contexto). Siempre restablecer el estado en `@BeforeEach` con `transitionToClosedState()` para evitar dependencias entre tests.

## Estrategia 3: Test con WireMock para simular fallos de red

WireMock simula el servicio downstream con respuestas de error reales (5xx, timeout, connection refused). Es el único test que verifica que el umbral de fallos configura el circuito correctamente en condiciones reales de red.

```xml
<dependency>
    <groupId>org.wiremock.integrations</groupId>
    <artifactId>wiremock-spring-boot</artifactId>
    <version>3.2.0</version>
    <scope>test</scope>
</dependency>
```

```java
package com.example.resilience;

import com.github.tomakehurst.wiremock.client.WireMock;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.wiremock.spring.EnableWireMock;
import io.github.resilience4j.circuitbreaker.CircuitBreaker;
import io.github.resilience4j.circuitbreaker.CircuitBreakerRegistry;

import static com.github.tomakehurst.wiremock.client.WireMock.*;
import static org.assertj.core.api.Assertions.assertThat;
import static org.awaitility.Awaitility.await;
import static java.util.concurrent.TimeUnit.SECONDS;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@EnableWireMock
class CircuitBreakerWireMockTest {

    @Autowired
    private CircuitBreakerRegistry circuitBreakerRegistry;

    @Autowired
    private OrderServiceClient orderServiceClient; // Feign o RestClient

    @Test
    void whenDownstreamReturns500Repeatedly_thenCircuitOpens() {
        // Configurar WireMock para devolver 500 en todas las peticiones
        stubFor(get(urlPathEqualTo("/orders/123"))
            .willReturn(serverError()));

        CircuitBreaker cb = circuitBreakerRegistry.circuitBreaker("orders");
        // slidingWindowSize = 5, failureRateThreshold = 50% → necesitamos 5 llamadas
        for (int i = 0; i < 5; i++) {
            try {
                orderServiceClient.getOrder("123");
            } catch (Exception ignored) {
                // Se espera excepción
            }
        }

        // Tras 5 fallos (100% > 50%), el circuito debe abrirse
        await()
            .atMost(2, SECONDS)
            .untilAsserted(() ->
                assertThat(cb.getState()).isEqualTo(CircuitBreaker.State.OPEN)
            );
    }
}
```

### Tabla resumen: cuándo usar cada estrategia

| Situación | Estrategia recomendada |
|---|---|
| Verificar lógica del fallback sin Spring | 1 — CircuitBreakerRegistry.ofDefaults() |
| Verificar transición de estado forzada con AOP | 2 — @SpringBootTest + transitionToOpenState() |
| Verificar contadores de Retry con Micrometer | 2 — @SpringBootTest con MeterRegistry |
| Verificar Bulkhead con carga concurrente real | 2 — @SpringBootTest con ExecutorService |
| Verificar que el circuito se abre con fallos reales | 3 — WireMock |
| Verificar comportamiento ante timeouts de red | 3 — WireMock con `fixedDelay` |

## Tabla de elementos clave

Los componentes y métodos que un desarrollador senior debe conocer para testear tolerancia a fallos:

| Componente / Método | Descripción |
|---|---|
| `CircuitBreakerRegistry.ofDefaults()` | Crea un registry en memoria sin Spring para tests unitarios |
| `cb.transitionToOpenState()` | Fuerza el estado OPEN directamente; también: `transitionToClosedState()`, `transitionToHalfOpenState()` |
| `cb.getMetrics().getNumberOfFailedCalls()` | Acceso programático a contadores de fallos para assertions en tests |
| `Retry.decorateCheckedSupplier(retry, supplier)` | Decora un lambda con Retry sin AOP para tests unitarios |
| `BulkheadFullException` | Excepción lanzada cuando el Bulkhead rechaza una llamada; verificar en assertions |
| `@EnableWireMock` | Anotación de wiremock-spring-boot para iniciar WireMock en tests de integración |
| `MeterRegistry` | Inyectable en tests para verificar contadores de Retry: `meterRegistry.counter("resilience4j.retry.calls", "name", "orders", "kind", "successful_with_retry")` |

## Buenas y malas prácticas

**Hacer:**
- Restablecer el estado del circuito en `@BeforeEach` cuando los tests comparten contexto de Spring; el estado persiste entre tests dentro del mismo contexto y puede causar dependencias no intencionales.
- Usar `transitionToOpenState()` en lugar de forzar fallos reales para tests de fallback: es instantáneo y determina con precisión cuándo el circuito está abierto.
- Verificar los contadores de Retry con `MeterRegistry` en lugar de contar llamadas manualmente: los contadores de Micrometer reflejan el comportamiento real del decorador.
- Usar virtual threads (`Executors.newVirtualThreadPerTaskExecutor()`) en tests de Bulkhead concurrente para simular alta concurrencia sin sobrecargar el test runner.

**Evitar:**
- Testear solo el "camino feliz" (servicio disponible, circuito cerrado): no detecta problemas de configuración del umbral ni del fallback.
- Usar `Thread.sleep()` en tests de resiliencia para esperar transiciones de estado: es frágil y lento; usar `Awaitility.await()` con timeout explícito.
- Compartir la misma instancia de `CircuitBreaker` entre tests paralelos sin aislar el estado: los contadores de una suite pueden contaminar a otra.
- Configurar `waitDuration` con valores reales (segundos) en tests de Retry: hace los tests innecesariamente lentos; usar `Duration.ofMillis(10)` en configuración de test.

---

← [5.13 Integración con OpenFeign, RestClient y WebClient](sc-circuitbreaker-feign.md) | [Índice (README.md)](README.md) | [6.1 Modelo de programación funcional](sc-stream-modelo-funcional.md) →
