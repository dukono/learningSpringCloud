# 13.12 Testing de microservicios: Contract Testing, Component Tests, Chaos Engineering y estrategia E2E

← [13.11 Antipatrones](sc-patrones-antipatrones.md) | [Índice](README.md) →

---

## Introducción

El testing de microservicios requiere una estrategia diferente al testing de monolitos. Las pruebas de integración clásicas que levantan todo el sistema son lentas, frágiles y costosas de mantener en un sistema con decenas de servicios. La estrategia óptima combina Contract Testing para verificar las integraciones sin entornos completos, Component Tests para verificar el comportamiento de cada servicio en aislamiento, testing de resiliencia para verificar el comportamiento ante fallos, y una capa mínima de E2E que verifica únicamente los flujos críticos de negocio.

## La pirámide de testing en microservicios

> [CONCEPTO] **Consumer-Driven Contract Testing**: en lugar de levantar todos los servicios juntos para verificar las integraciones, el consumidor define el contrato de lo que espera del productor (request/response o mensaje). El productor verifica que su implementación satisface ese contrato en tests aislados. Spring Cloud Contract es la implementación de referencia. Esto elimina la necesidad de entornos de integración completos para verificar compatibilidad entre servicios.

La pirámide de testing adaptada a microservicios:

```
                    ┌─────────────┐
                    │  E2E Tests  │  ← mínimos, solo flujos críticos
                    │  (frágiles) │
                ┌───┴─────────────┴───┐
                │  Integration Tests  │  ← Contract Tests + Component Tests
                │  (Spring Cloud      │     con WireMock / Stubs
                │   Contract)         │
        ┌───────┴─────────────────────┴───────┐
        │         Unit Tests                  │  ← la mayor parte
        │  (lógica de negocio, dominio)        │
        └─────────────────────────────────────┘
```

## Component Tests: testing en aislamiento con dependencias stubbeadas

> [CONCEPTO] **Component Test en microservicios**: un Component Test verifica el comportamiento completo de un microservicio (desde su API hasta su base de datos) reemplazando las dependencias externas (otros servicios, brokers) con stubs o mocks. No es un test de integración clásico que necesita todos los servicios reales — es un test del "servicio como unidad de despliegue", aislado del ecosistema.

La herramienta principal para stubs de servicios HTTP en Component Tests es **WireMock**: levanta un servidor HTTP que simula las respuestas de los servicios externos.

## E2E Tests: mínimos y focalizados en flujos críticos

> [CONCEPTO] **End-to-End Test en microservicios**: los tests E2E verifican el sistema completo de extremo a extremo. Son los más costosos: requieren todos los servicios en ejecución, son frágiles (fallan por razones no relacionadas con la funcionalidad — red, timing, datos de test), y son lentos. La estrategia recomendada: minimizar los E2E a los flujos de negocio más críticos (happy path del checkout, login básico) y construir confianza con Contract Tests + Component Tests en su lugar.

## Testing de resiliencia: Chaos Engineering

> [CONCEPTO] **Testing de patrones de resiliencia (Chaos Engineering)**: el Chaos Engineering verifica que los mecanismos de resiliencia (Circuit Breaker, Bulkhead, Retry) funcionan correctamente bajo condiciones reales de fallo. No es suficiente con escribir el código — hay que verificar que el Circuit Breaker se abre cuando debe, que el Bulkhead realmente limita la concurrencia y que el Retry no genera avalanchas. Herramientas: WireMock (simula servicios lentos o caídos), Chaos Monkey for Spring Boot (inyecta fallos en la JVM), Testcontainers (infraestructura real en tests).

La diferencia con el testing funcional clásico: el testing de resiliencia verifica el comportamiento ante fallos del sistema, no ante datos inválidos. Inyecta fallos de red, latencia, servicios no disponibles, y verifica que el sistema degrada graciosamente.

## Ejemplo central: estrategia de testing completa

El siguiente ejemplo implementa los cuatro niveles de testing: un Contract Test con Spring Cloud Contract, un Component Test con WireMock, y un test de resiliencia con Circuit Breaker usando WireMock para simular fallos.

```java
// NIVEL 1: Contract Test — Producer verifica el contrato definido por el Consumer
// Contrato en src/test/resources/contracts/orders/shouldReturnOrderById.groovy

// OrdersBaseTest.java — clase base para los tests generados por Spring Cloud Contract
package com.example.orders.contract;

import com.example.orders.controller.OrderController;
import com.example.orders.service.OrderService;
import io.restassured.module.mockmvc.RestAssuredMockMvc;
import org.junit.jupiter.api.BeforeEach;
import org.mockito.Mockito;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.mock.mockito.MockBean;

@SpringBootTest
public abstract class OrdersBaseTest {

    @Autowired
    private OrderController orderController;

    @MockBean
    private OrderService orderService;

    @BeforeEach
    void setUp() {
        // Mock del servicio para el contrato: el consumidor espera este comportamiento
        Mockito.when(orderService.findById("order-123"))
            .thenReturn(new OrderDTO("order-123", "customer-456", 99.99, "CONFIRMED"));

        RestAssuredMockMvc.standaloneSetup(orderController);
    }
}
```

```groovy
// src/test/resources/contracts/orders/shouldReturnOrderById.groovy
// Contrato definido por el consumidor (o acordado entre productor y consumidor)

import org.springframework.cloud.contract.spec.Contract

Contract.make {
    description "should return order by id"

    request {
        method GET()
        url "/orders/order-123"
        headers {
            contentType(applicationJson())
        }
    }

    response {
        status OK()
        headers {
            contentType(applicationJson())
        }
        body(
            id: "order-123",
            customerId: "customer-456",
            amount: 99.99,
            status: "CONFIRMED"
        )
    }
}
```

```java
// NIVEL 2: Component Test — verifica el microservicio completo con dependencias stubbeadas

package com.example.orders.component;

import com.github.tomakehurst.wiremock.WireMockServer;
import com.github.tomakehurst.wiremock.client.WireMock;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.cloud.contract.wiremock.AutoConfigureWireMock;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import static com.github.tomakehurst.wiremock.client.WireMock.*;
import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT,
    properties = {
        "inventory-service.url=http://localhost:${wiremock.server.port}",
        "payment-service.url=http://localhost:${wiremock.server.port}"
    })
@AutoConfigureWireMock(port = 0) // WireMock en puerto aleatorio
class PlaceOrderComponentTest {

    @Autowired
    private TestRestTemplate restTemplate;

    @BeforeEach
    void setUpStubs() {
        // Stub del servicio de inventario: devuelve stock disponible
        stubFor(get(urlEqualTo("/stock/product-abc/available?quantity=1"))
            .willReturn(aResponse()
                .withStatus(200)
                .withHeader("Content-Type", "application/json")
                .withBody("true")));

        // Stub del servicio de pagos: pago exitoso
        stubFor(post(urlEqualTo("/payments"))
            .willReturn(aResponse()
                .withStatus(200)
                .withHeader("Content-Type", "application/json")
                .withBody("{\"paymentId\": \"pay-789\", \"status\": \"COMPLETED\"}")));
    }

    @Test
    void shouldPlaceOrderSuccessfully() {
        // Arrange
        CreateOrderRequest request = new CreateOrderRequest("customer-123", "product-abc", 1);

        // Act — llama al microservicio real con BD real (Testcontainers o H2) y stubs WireMock
        ResponseEntity<OrderResponse> response = restTemplate.postForEntity(
            "/orders", request, OrderResponse.class);

        // Assert
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(response.getBody()).isNotNull();
        assertThat(response.getBody().status()).isEqualTo("CONFIRMED");
    }
}
```

```java
// NIVEL 3: Test de resiliencia — verifica que el Circuit Breaker se abre bajo fallos

package com.example.orders.resilience;

import com.github.tomakehurst.wiremock.client.WireMock;
import io.github.resilience4j.circuitbreaker.CircuitBreaker;
import io.github.resilience4j.circuitbreaker.CircuitBreakerRegistry;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.cloud.contract.wiremock.AutoConfigureWireMock;
import static com.github.tomakehurst.wiremock.client.WireMock.*;
import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT,
    properties = {"inventory-service.url=http://localhost:${wiremock.server.port}"})
@AutoConfigureWireMock(port = 0)
class InventoryCircuitBreakerTest {

    @Autowired
    private CircuitBreakerRegistry circuitBreakerRegistry;

    @Autowired
    private OrderService orderService;

    @Test
    void shouldOpenCircuitBreakerAfterConsecutiveFailures() {
        // Arrange: simular que el servicio de inventario está caído (503)
        stubFor(get(urlMatching("/stock/.*"))
            .willReturn(aResponse()
                .withStatus(503)
                .withBody("Service Unavailable")));

        CircuitBreaker circuitBreaker = circuitBreakerRegistry.circuitBreaker("inventory-service");

        // Act: hacer suficientes llamadas para abrir el Circuit Breaker
        // (configuración: 5 llamadas mínimas, 50% de tasa de fallo)
        for (int i = 0; i < 10; i++) {
            try {
                orderService.checkInventoryAvailability("product-abc", 1).block();
            } catch (Exception ignored) {
                // se espera que fallen
            }
        }

        // Assert: el Circuit Breaker debe estar abierto
        assertThat(circuitBreaker.getState())
            .isEqualTo(CircuitBreaker.State.OPEN);

        // Assert: las llamadas subsiguientes fallan rápido (sin llegar al stub)
        // reset WireMock para verificar que no se hace ninguna llamada
        WireMock.reset();
        try {
            orderService.checkInventoryAvailability("product-abc", 1).block();
        } catch (Exception e) {
            // se espera CallNotPermittedException del Circuit Breaker
        }
        WireMock.verify(0, getRequestedFor(urlMatching("/stock/.*")));
    }
}
```

## Buenas y malas prácticas

**Buenas prácticas:**
- Seguir la pirámide: muchos unit tests, Component Tests + Contract Tests para integraciones, E2E mínimos.
- Usar Testcontainers para levantar infraestructura real (PostgreSQL, Kafka) en Component Tests — más fiel que los mocks en memoria.
- Incluir tests de resiliencia en la pipeline de CI/CD para verificar que los Circuit Breakers se comportan como se configuró.
- Versionar los contratos de Spring Cloud Contract junto con el código del productor.

**Malas prácticas:**
- Intentar replicar el comportamiento completo del sistema en tests E2E — solo flujos críticos de negocio.
- Mockear las bases de datos en Component Tests — usar H2 o Testcontainers para fidelidad real.
- No tener Contract Tests y solo confiar en E2E — los E2E son los más frágiles y caros de mantener.

> [ADVERTENCIA] Los tests de Chaos Engineering que inyectan fallos a nivel de producción (Chaos Monkey) deben ejecutarse con un plan de recuperación claro y durante ventanas de mantenimiento o en entornos de staging, no en producción sin supervisión. Chaos Engineering en producción requiere madurez en observabilidad y capacidad de respuesta del equipo.

## Verificación y práctica

> [EXAMEN] 1. ¿Cómo Spring Cloud Contract permite que el consumidor defina el contrato y el productor lo verifique sin necesitar un entorno de integración real?

> [EXAMEN] 2. ¿Qué prueba un Component Test en un microservicio, qué colaboradores se stubbbean con WireMock, y en qué se diferencia de un test de integración clásico?

> [EXAMEN] 3. ¿Por qué los tests E2E son frágiles en arquitecturas de microservicios, y cuál es la estrategia recomendada para minimizarlos sin perder confianza en el sistema?

> [EXAMEN] 4. ¿Cómo verificas en un test que un Circuit Breaker se abre correctamente bajo fallos simulados, y qué herramienta usas para simular el fallo del servicio downstream?

> [EXAMEN] 5. ¿Qué ventaja aporta Testcontainers sobre H2/bases de datos en memoria en los Component Tests de un microservicio?

---

← [13.11 Antipatrones](sc-patrones-antipatrones.md) | [Índice](README.md) →
