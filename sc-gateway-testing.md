# 3.15 Testing / Verificación de Gateway

← [3.14 Manejo de errores y respuestas de error](sc-gateway-error-handling.md) | [Índice](README.md) | [4.1 Setup y bootstrap](sc-feign-setup.md) →

---

## Introducción

Verificar el comportamiento de un gateway es más complejo que testear un microservicio individual: implica validar el matching de predicados, la transformación de cabeceras y paths por filtros, la integración con service discovery, y el comportamiento ante fallos del backend. Sin una estrategia de testing clara, los cambios en rutas o filtros se despliegan con riesgo de romper el enrutamiento en producción. El desarrollador necesita verificar el comportamiento de rutas, filtros y predicados de Gateway para garantizar que el enrutamiento es correcto antes de desplegar a producción.

## Estrategia de testing: tres niveles

Los tres niveles de test en Gateway cubren distintos aspectos y tienen distintos costes de ejecución.

```
Test unitario         Test de integración      Test E2E / Contract
────────────────      ───────────────────      ─────────────────────
GatewayFilter         @SpringBootTest          Gateway real +
predicado custom      + WebTestClient           backends reales /
aislado               + MockWebServer          stubs (contract)
                                                
Sin Spring context    Contexto completo         Infraestructura real
Rápido (<100ms)       Moderado (<5s)           Lento (minutos)
```

---

### Estrategia 1: test unitario de GatewayFilter personalizado

Los filtros custom (`AbstractGatewayFilterFactory`) pueden testarse sin levantar el contexto de Spring usando mocks del `ServerWebExchange`.

```java
package com.example.gateway;

import org.junit.jupiter.api.Test;
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.mock.http.server.reactive.MockServerHttpRequest;
import org.springframework.mock.web.server.MockServerWebExchange;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;
import reactor.test.StepVerifier;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

class HmacValidationGatewayFilterFactoryTest {

    private final HmacValidationGatewayFilterFactory factory =
        new HmacValidationGatewayFilterFactory();

    @Test
    void deberiaRechazarPeticionSinFirma() {
        // Arrange
        var request = MockServerHttpRequest.get("/api/orders/123").build();
        var exchange = MockServerWebExchange.from(request);
        var chain = mock(GatewayFilterChain.class);

        var config = new HmacValidationGatewayFilterFactory.Config();
        config.setSecretKey("test-secret");
        config.setSignatureHeader("X-Signature");

        var filter = factory.apply(config);

        // Act
        filter.filter(exchange, chain).block();

        // Assert: debe haber devuelto 401 sin llamar a la cadena
        assertThat(exchange.getResponse().getStatusCode())
            .isEqualTo(org.springframework.http.HttpStatus.UNAUTHORIZED);
        verify(chain, never()).filter(any());
    }

    @Test
    void deberiaPasarPeticionConFirmaValida() {
        // Arrange — calcular firma HMAC real para el path de prueba
        String path = "/api/orders/123?";
        String secret = "test-secret";
        // La firma correcta se calcularía con el mismo algoritmo que el filtro

        var chain = mock(GatewayFilterChain.class);
        when(chain.filter(any())).thenReturn(Mono.empty());

        var config = new HmacValidationGatewayFilterFactory.Config();
        config.setSecretKey(secret);
        config.setSignatureHeader("X-Signature");

        // Usar reflexión o exponer el método de cálculo de firma para el test
        // (ejemplo simplificado; en la práctica: utilitario de test)
        var filter = factory.apply(config);

        var request = MockServerHttpRequest.get("/api/orders/123")
            .header("X-Signature", "firma-calculada-correctamente")
            .build();
        var exchange = MockServerWebExchange.from(request);

        // Act & Assert
        StepVerifier.create(filter.filter(exchange, chain))
            .verifyComplete();
        // Con firma incorrecta, verificar que no se llama a chain.filter()
    }
}
```

---

### Estrategia 2: test de integración con @SpringBootTest y WebTestClient

El test de integración levanta el contexto completo de Spring Cloud Gateway y usa `WebTestClient` para verificar el comportamiento end-to-end de rutas y filtros.

```java
package com.example.gateway;

import okhttp3.mockwebserver.MockResponse;
import okhttp3.mockwebserver.MockWebServer;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.cloud.contract.wiremock.AutoConfigureWireMock;
import org.springframework.http.HttpStatus;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.springframework.test.context.TestPropertySource;
import org.springframework.test.web.reactive.server.WebTestClient;

import java.io.IOException;

/**
 * Test de integración: levanta el contexto completo de Gateway
 * y usa MockWebServer como backend simulado.
 *
 * WireMock es la alternativa más habitual; MockWebServer (OkHttp) es más ligero.
 */
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@TestPropertySource(properties = {
    // Deshabilitar Eureka para tests (sin infraestructura externa)
    "eureka.client.enabled=false",
    "spring.cloud.discovery.enabled=false"
})
class GatewayIntegrationTest {

    private static MockWebServer mockBackend;

    @DynamicPropertySource
    static void configureBackendUrl(DynamicPropertyRegistry registry) throws IOException {
        mockBackend = new MockWebServer();
        mockBackend.start();
        // Sobrescribir la URI de la ruta para apuntar al MockWebServer
        registry.add("spring.cloud.gateway.routes[0].uri",
            () -> "http://localhost:" + mockBackend.getPort());
    }

    @AfterEach
    void tearDown() throws IOException {
        if (mockBackend != null) {
            mockBackend.shutdown();
        }
    }

    @Autowired
    private WebTestClient webTestClient;

    @Test
    void deberiaPropagar200DelBackend() {
        // Configurar respuesta del backend simulado
        mockBackend.enqueue(new MockResponse()
            .setResponseCode(200)
            .setBody("{\"id\": 123, \"status\": \"PENDING\"}")
            .addHeader("Content-Type", "application/json"));

        // Verificar que Gateway enruta correctamente y retorna 200
        webTestClient.get()
            .uri("/api/orders/123")
            .exchange()
            .expectStatus().isOk()
            .expectHeader().valueEquals("X-Gateway-Version", "1.0")  // header añadido por filtro
            .expectBody()
            .jsonPath("$.id").isEqualTo(123)
            .jsonPath("$.status").isEqualTo("PENDING");
    }

    @Test
    void deberiaPropagar404ComoErrorCanónico() {
        // Backend devuelve 404; Gateway lo propaga con su formato de error
        mockBackend.enqueue(new MockResponse().setResponseCode(404));

        webTestClient.get()
            .uri("/api/orders/999")
            .exchange()
            .expectStatus().isNotFound();
    }

    @Test
    void deberiaAplicarRateLimitingConClave() {
        // Verificar que el filtro RequestRateLimiter responde 429 al superar el límite
        // (Requiere Redis embebido en el test o configurar denyEmptyKey=false)
        webTestClient.get()
            .uri("/api/orders/1")
            .header("X-Forwarded-For", "192.168.1.100")
            .exchange()
            .expectStatus().isOk(); // primera petición

        // Peticiones adicionales al superar replenishRate deberían recibir 429
    }

    @Test
    void deberiaRedirigirRutaConoPrefix() {
        mockBackend.enqueue(new MockResponse()
            .setResponseCode(200)
            .setBody("[]")
            .addHeader("Content-Type", "application/json"));

        // Verificar que /api/catalog/items → /items (StripPrefix=1 + RewritePath)
        webTestClient.get()
            .uri("/api/catalog/items")
            .exchange()
            .expectStatus().isOk();

        // Verificar que el backend recibió el path correcto (sin /api)
        var request = mockBackend.takeRequest();
        assertThat(request.getPath()).isEqualTo("/items");
    }
}
```

---

### Estrategia 3: test del predicado personalizado

Los predicados personalizados (`AbstractRoutePredicateFactory`) se pueden testear con `MockServerWebExchange`:

```java
package com.example.gateway;

import org.junit.jupiter.api.Test;
import org.springframework.mock.http.server.reactive.MockServerHttpRequest;
import org.springframework.mock.web.server.MockServerWebExchange;

import static org.assertj.core.api.Assertions.assertThat;

class ApiVersionRoutePredicateFactoryTest {

    private final ApiVersionRoutePredicateFactory factory =
        new ApiVersionRoutePredicateFactory();

    @Test
    void deberiaMakechearConVersionCorrecta() {
        var config = new ApiVersionRoutePredicateFactory.Config();
        config.setVersion("v2");
        var predicate = factory.apply(config);

        var exchange = MockServerWebExchange.from(
            MockServerHttpRequest.get("/api/test")
                .header("X-API-Version", "v2")
                .build());

        assertThat(predicate.test(exchange)).isTrue();
    }

    @Test
    void noDeberiaMakechearConVersionIncorrecta() {
        var config = new ApiVersionRoutePredicateFactory.Config();
        config.setVersion("v2");
        var predicate = factory.apply(config);

        var exchange = MockServerWebExchange.from(
            MockServerHttpRequest.get("/api/test")
                .header("X-API-Version", "v1")
                .build());

        assertThat(predicate.test(exchange)).isFalse();
    }

    @Test
    void noDeberiaMakechearSinHeaderDiretamente() {
        var config = new ApiVersionRoutePredicateFactory.Config();
        config.setVersion("v2");
        var predicate = factory.apply(config);

        var exchange = MockServerWebExchange.from(
            MockServerHttpRequest.get("/api/test").build());

        assertThat(predicate.test(exchange)).isFalse();
    }
}
```

## Tabla resumen de estrategias

La siguiente tabla resume cuándo aplicar cada estrategia de verificación.

| Situación | Estrategia recomendada |
|-----------|----------------------|
| Filtro personalizado con lógica de transformación | Test unitario con `MockServerWebExchange` |
| Predicado personalizado | Test unitario con `MockServerWebExchange` |
| Ruta completa: predicado + filtros + proxy | Test de integración con `@SpringBootTest` + `MockWebServer` |
| Comportamiento del Circuit Breaker con fallback | Test de integración con backend que devuelve 500 |
| Comportamiento del Retry | Test de integración con backend que falla N veces y recupera |
| Rate limiting (sin Redis real) | Test de integración con Redis embebido (`testcontainers-redis`) |
| Verificación de formato de respuesta de error | Test de integración con backend que devuelve 404/503 |
| Rutas con service discovery (Eureka) | Deshabilitar discovery en test; hardcodear URI con `DynamicPropertySource` |
| Gateway MVC | Reemplazar `WebTestClient` con `TestRestTemplate` |

## Preguntas de entrevista frecuentes

**P1: ¿Por qué se usa `WebTestClient` y no `TestRestTemplate` para testear Gateway?**

Gateway está construido sobre WebFlux (stack reactivo); `WebTestClient` es el cliente de test reactivo nativo de Spring. `TestRestTemplate` funciona sobre el stack servlet y no puede interactuar correctamente con respuestas reactivas como SSE o streams. Para Gateway MVC se puede usar `TestRestTemplate` como alternativa.

**P2: ¿Cómo se testea una ruta que usa `lb://` sin tener Eureka levantado?**

Tres opciones: (a) desactivar Eureka con `eureka.client.enabled=false` y sobrescribir la URI de la ruta con `@DynamicPropertySource` apuntando a un `MockWebServer`; (b) usar `spring.cloud.loadbalancer.ribbon.enabled=false` (legacy) y definir instancias estáticas en el `ServiceInstanceListSupplier`; (c) usar un `@Bean ServiceInstanceListSupplier` mock que devuelva instancias conocidas.

**P3: ¿Qué diferencia hay entre testear un `GatewayFilter` y un `GlobalFilter`?**

Un `GatewayFilter` se prueba mejor en test de integración (se configura en una ruta específica y se verifica que el comportamiento de esa ruta es el esperado). Un `GlobalFilter` se puede testear unitariamente si su lógica es independiente de la ruta; o en test de integración verificando que afecta a todas las rutas. La diferencia de testing es que el `GatewayFilter` se instancia por ruta (se puede testear en aislamiento con la factory), mientras que el `GlobalFilter` es un singleton que siempre está activo.

**P4: ¿Cómo se verifica el comportamiento del Circuit Breaker en un test de integración?**

Se configura el `MockWebServer` (o WireMock) para devolver el código HTTP que activa el circuit breaker (ej: 503) las primeras N peticiones, y se verifica que (a) la primera respuesta al cliente es 503 o el fallback, (b) tras el número configurado de fallos el circuito se abre y se devuelve el fallback sin llamar al backend, y (c) tras `waitDurationInOpenState` el circuito pasa a HALF_OPEN y permite una petición de prueba.

**P5: ¿Por qué se desactiva el DiscoveryClient en tests de Gateway?**

El DiscoveryClient (Eureka, Consul) intenta conectarse a su servidor en el arranque. En tests unitarios e integración sin infraestructura externa, esta conexión falla o espera al timeout, ralentizando enormemente la suite de tests. Desactivarlo con `spring.cloud.discovery.enabled=false` y `eureka.client.enabled=false` permite que los tests arranquen en milisegundos sin dependencias externas.

---

← [3.14 Manejo de errores y respuestas de error](sc-gateway-error-handling.md) | [Índice](README.md) | [4.1 Setup y bootstrap](sc-feign-setup.md) →
