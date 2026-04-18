# 4.13 Testing con Resilience4j

← [4.12 Métricas avanzadas y Health Indicators](sc-circuitbreaker-metricas.md) | [Índice](README.md) | [5.1 Arquitectura y ciclo de vida de Spring Cloud Gateway](sc-gateway-arquitectura.md) →

---

## Introducción

Los patrones de resiliencia solo aportan valor si están correctamente probados. Sin tests, un CircuitBreaker mal configurado puede estar en estado OPEN permanente, o nunca abrir cuando debería, y ninguna métrica en producción lo detectará hasta que haya un incidente. Resilience4j facilita el testing porque expone API programática para forzar estados (`transitionToOpenState()`), los umbrales se pueden bajar a valores mínimos para tests, y la integración con WireMock permite simular fallos de downstream de forma determinista.

> [EXAMEN] La capacidad de forzar programáticamente un CircuitBreaker al estado OPEN en un test de integración es una pregunta directa del examen VMware Spring Professional.

## Estrategias de testing

Existen tres estrategias de testing para Resilience4j, cada una con su alcance y propósito:

La primera estrategia es el test de integración con `@SpringBootTest` y `CircuitBreakerRegistry`. Se usa para verificar el comportamiento completo del CB incluyendo el fallback. Se fuerza el estado OPEN programáticamente para evitar tener que producir suficientes fallos reales.

La segunda estrategia es el test con WireMock para simular fallos de downstream. Se usa para probar las transiciones de estado naturales: se configura WireMock para devolver 500 N veces y se verifica que el CB abre tras superar `failureRateThreshold`.

La tercera estrategia es el test unitario de concurrencia con múltiples threads para Bulkhead. Se usa para verificar que `BulkheadFullException` se lanza correctamente cuando se supera `maxConcurrentCalls`.

## Ejemplo central

El ejemplo cubre las tres estrategias con tests completos, incluyendo uso de Awaitility para condiciones asíncronas:

```java
package com.example.test;

import io.github.resilience4j.circuitbreaker.CircuitBreaker;
import io.github.resilience4j.circuitbreaker.CircuitBreakerRegistry;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.web.servlet.MockMvc;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest
@AutoConfigureMockMvc
class CircuitBreakerIntegrationTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private CircuitBreakerRegistry circuitBreakerRegistry;

    @BeforeEach
    void resetCircuitBreaker() {
        // Resetear el CB a estado CLOSED antes de cada test
        CircuitBreaker cb = circuitBreakerRegistry.circuitBreaker("paymentService");
        cb.transitionToClosedState();
    }

    @Test
    void whenCircuitBreakerOpen_thenFallbackIsReturned() throws Exception {
        // Forzar el estado OPEN programáticamente
        CircuitBreaker cb = circuitBreakerRegistry.circuitBreaker("paymentService");
        cb.transitionToOpenState();

        // Verificar que el endpoint devuelve la respuesta de fallback
        mockMvc.perform(get("/payments/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.status").value("FALLBACK"))
            .andExpect(jsonPath("$.message").value("Payment service temporarily unavailable"));
    }

    @Test
    void whenCircuitBreakerClosed_thenNormalResponseIsReturned() throws Exception {
        // El CB está en CLOSED (reseteado en @BeforeEach)
        mockMvc.perform(get("/payments/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.status").value("SUCCESS"));
    }

    @Test
    void whenCircuitBreakerInHalfOpen_thenProbeCallsAreAllowed() throws Exception {
        CircuitBreaker cb = circuitBreakerRegistry.circuitBreaker("paymentService");
        cb.transitionToHalfOpenState();

        // En HALF_OPEN, las llamadas de prueba deben pasar
        mockMvc.perform(get("/payments/1"))
            .andExpect(status().isOk());

        // Verificar estado después de una llamada exitosa
        // (si minimumCalls en HALF_OPEN=1, debería transicionar a CLOSED)
    }
}
```

Test con WireMock para probar transiciones naturales de estado:

```java
package com.example.test;

import com.github.tomakehurst.wiremock.client.WireMock;
import io.github.resilience4j.circuitbreaker.CircuitBreaker;
import io.github.resilience4j.circuitbreaker.CircuitBreakerRegistry;
import org.awaitility.Awaitility;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.cloud.contract.wiremock.AutoConfigureWireMock;
import java.time.Duration;
import static com.github.tomakehurst.wiremock.client.WireMock.*;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT,
    properties = {
        // Bajar umbrales para que el CB abra rápido en tests
        "resilience4j.circuitbreaker.instances.paymentService.minimum-number-of-calls=3",
        "resilience4j.circuitbreaker.instances.paymentService.failure-rate-threshold=50",
        "resilience4j.circuitbreaker.instances.paymentService.sliding-window-size=4",
        "resilience4j.circuitbreaker.instances.paymentService.wait-duration-in-open-state=2s",
        "resilience4j.circuitbreaker.instances.paymentService.permitted-number-of-calls-in-half-open-state=1"
    })
@AutoConfigureWireMock(port = 0)  // WireMock en puerto aleatorio
class CircuitBreakerWireMockTest {

    @Autowired
    private PaymentService paymentService;

    @Autowired
    private CircuitBreakerRegistry circuitBreakerRegistry;

    @Test
    void whenDownstreamFails_thenCircuitBreakerOpens() {
        // Configurar WireMock para devolver 500 en todas las llamadas
        stubFor(post(urlEqualTo("/payments"))
            .willReturn(aResponse()
                .withStatus(500)
                .withBody("{\"error\":\"Internal Server Error\"}")));

        // Ejecutar suficientes llamadas para superar minimumNumberOfCalls
        for (int i = 0; i < 4; i++) {
            try {
                paymentService.processPayment(new PaymentRequest());
            } catch (Exception ignored) {
                // Se espera excepción hasta que actúe el fallback
            }
        }

        // Verificar que el CB está ahora en OPEN
        CircuitBreaker cb = circuitBreakerRegistry.circuitBreaker("paymentService");
        assert cb.getState() == CircuitBreaker.State.OPEN;
    }

    @Test
    void whenCircuitOpensAndWaits_thenTransitionsToHalfOpen() throws Exception {
        CircuitBreaker cb = circuitBreakerRegistry.circuitBreaker("paymentService");
        cb.transitionToOpenState();

        // Esperar a que el waitDurationInOpenState expire (configurado a 2s en test)
        // y el CB transicione automáticamente a HALF_OPEN
        // Requiere automaticTransitionFromOpenToHalfOpenEnabled=true
        Awaitility.await()
            .atMost(Duration.ofSeconds(5))
            .pollInterval(Duration.ofMillis(200))
            .until(() -> cb.getState() == CircuitBreaker.State.HALF_OPEN);
    }

    @Test
    void countRetryAttempts_withWireMock() {
        // WireMock devuelve 503 las primeras 2 veces, 200 la tercera
        stubFor(post(urlEqualTo("/payments"))
            .inScenario("retry-scenario")
            .whenScenarioStateIs("Started")
            .willReturn(aResponse().withStatus(503))
            .willSetStateTo("first-fail"));

        stubFor(post(urlEqualTo("/payments"))
            .inScenario("retry-scenario")
            .whenScenarioStateIs("first-fail")
            .willReturn(aResponse().withStatus(503))
            .willSetStateTo("second-fail"));

        stubFor(post(urlEqualTo("/payments"))
            .inScenario("retry-scenario")
            .whenScenarioStateIs("second-fail")
            .willReturn(aResponse()
                .withStatus(200)
                .withBody("{\"status\":\"SUCCESS\"}")));

        // Con Retry maxAttempts=3, debe tener éxito en el tercer intento
        PaymentResult result = paymentService.processPayment(new PaymentRequest());
        assert result.isSuccess();

        // Verificar que WireMock recibió exactamente 3 llamadas
        verify(3, postRequestedFor(urlEqualTo("/payments")));
    }
}
```

Test de concurrencia para Bulkhead:

```java
package com.example.test;

import io.github.resilience4j.bulkhead.BulkheadFullException;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

@SpringBootTest(properties = {
    "resilience4j.bulkhead.instances.inventoryService.max-concurrent-calls=2",
    "resilience4j.bulkhead.instances.inventoryService.max-wait-duration=0"
})
class BulkheadTest {

    @Autowired
    private InventoryService inventoryService;

    @Test
    void whenMaxConcurrencyExceeded_thenBulkheadFullException() throws Exception {
        int threads = 5;
        CountDownLatch startLatch = new CountDownLatch(1);
        CountDownLatch doneLatch = new CountDownLatch(threads);
        AtomicInteger rejections = new AtomicInteger(0);
        AtomicInteger successes = new AtomicInteger(0);

        ExecutorService executor = Executors.newFixedThreadPool(threads);

        for (int i = 0; i < threads; i++) {
            executor.submit(() -> {
                try {
                    startLatch.await(); // Esperar señal para empezar todos a la vez
                    inventoryService.checkInventory(1L);
                    successes.incrementAndGet();
                } catch (BulkheadFullException e) {
                    rejections.incrementAndGet();
                } catch (InterruptedException ignored) {
                } finally {
                    doneLatch.countDown();
                }
            });
        }

        startLatch.countDown(); // Lanzar todos a la vez
        doneLatch.await(5, TimeUnit.SECONDS);
        executor.shutdown();

        // Con maxConcurrentCalls=2 y maxWaitDuration=0, exactamente 3 deben ser rechazados
        assert rejections.get() >= 3;
        assert successes.get() <= 2;
    }
}
```

## Tabla de métodos de transición programática

Los métodos disponibles en `CircuitBreaker` para forzar estados en tests:

| Método | Estado resultante | Uso en test |
|--------|------------------|-------------|
| `transitionToClosedState()` | CLOSED | Reset en `@BeforeEach` |
| `transitionToOpenState()` | OPEN | Probar fallback sin ejecutar fallos reales |
| `transitionToHalfOpenState()` | HALF_OPEN | Probar lógica de llamadas de prueba |
| `transitionToDisabledState()` | DISABLED | Desactivar el CB en tests de integración |
| `transitionToForcedOpenState()` | FORCED_OPEN | Probar que el sistema funciona con CB siempre abierto |

> [EXAMEN] `transitionToOpenState()` requiere que el CB haya acumulado al menos `minimumNumberOfCalls` en su sliding window O que use la transición directa del API. En la práctica, para tests siempre se usa `transitionToOpenState()` directamente sin necesitar producir fallos reales.

## Configuración de test recomendada

Para que los CBs se comporten de forma predecible en tests, siempre sobrescribir las propiedades críticas:

```properties
# Valores recomendados para tests
resilience4j.circuitbreaker.instances.myService.minimum-number-of-calls=3
resilience4j.circuitbreaker.instances.myService.sliding-window-size=4
resilience4j.circuitbreaker.instances.myService.failure-rate-threshold=50
resilience4j.circuitbreaker.instances.myService.wait-duration-in-open-state=2s
resilience4j.circuitbreaker.instances.myService.permitted-number-of-calls-in-half-open-state=1
resilience4j.circuitbreaker.instances.myService.automatic-transition-from-open-to-half-open-enabled=true
```

## Buenas y malas prácticas

**Buenas prácticas:**
- Siempre hacer reset del CB en `@BeforeEach` para evitar interferencias entre tests.
- Usar `@SpringBootTest(properties = {...})` para sobrescribir umbrales directamente en la anotación del test.
- Usar Awaitility para condiciones asíncronas (transiciones de estado) en lugar de `Thread.sleep()`.
- Usar WireMock scenarios para tests de Retry deterministas que verifican el número exacto de llamadas.

**Malas prácticas:**
- No hacer reset del CB entre tests: un test que deja el CB en OPEN hace fallar todos los tests siguientes.
- Usar `Thread.sleep()` fijo para esperar transiciones de estado: fragiliza los tests ante variaciones de carga del CI.
- Confiar solo en tests de `@CircuitBreaker` vía AOP sin probar el fallback directamente: el CB puede estar activo pero el fallback puede tener un bug silencioso.

## Verificación y práctica

> [EXAMEN] 1. ¿Qué método del `CircuitBreaker` se usa para forzar el estado OPEN en un test de integración?

> [EXAMEN] 2. ¿Por qué es necesario hacer reset del CB (`transitionToClosedState()`) en `@BeforeEach` en los tests?

> [EXAMEN] 3. ¿Cómo se verifican con WireMock exactamente 3 llamadas al downstream cuando Retry tiene `max-attempts=3`?

> [EXAMEN] 4. ¿Qué librería se recomienda para esperar condiciones asíncronas (como transiciones de estado) en tests de Resilience4j?

> [EXAMEN] 5. En un test de Bulkhead SEMAPHORE con `max-wait-duration=0` y `max-concurrent-calls=2`, si se lanzan 5 threads simultáneos, ¿cuántos `BulkheadFullException` se esperan como mínimo?

---

← [4.12 Métricas avanzadas y Health Indicators](sc-circuitbreaker-metricas.md) | [Índice](README.md) | [5.1 Arquitectura y ciclo de vida de Spring Cloud Gateway](sc-gateway-arquitectura.md) →
