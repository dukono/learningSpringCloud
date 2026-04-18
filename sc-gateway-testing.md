# 5.12 Testing / Verificación de Spring Cloud Gateway

← [5.11 Extensiones custom](sc-gateway-custom-extensiones.md) | [Índice](README.md) | [6.1 Spring Cloud Stream — Setup](sc-stream-setup.md) →

---

## Introducción

Spring Cloud Gateway es una aplicación reactiva basada en Spring WebFlux. Esto cambia fundamentalmente cómo se escribe su suite de testing: MockMvc no funciona porque el servidor no usa el stack Servlet; el cliente de tests correcto es `WebTestClient`. Sin una estrategia de testing específica para Gateway, es imposible verificar que las rutas con predicates, filtros, redirect, CircuitBreaker o StripPrefix se comportan correctamente antes de desplegar en producción.

Este subtema cubre la estrategia completa: `@SpringBootTest` con servidor embebido, `WebTestClient` para verificar transformaciones de rutas, WireMock para simular servicios downstream y test de CircuitBreaker con fallback.

> [PREREQUISITO] Se asume dominio de los filtros y predicates del módulo 5 (5.3–5.5) y conocimiento básico de `WebTestClient` de Spring WebFlux.

---

## Arquitectura del entorno de test

El siguiente diagrama muestra cómo se relacionan los componentes en un test de Gateway: el `WebTestClient` actúa como cliente externo, el Gateway embebido aplica predicates y filtros, y WireMock simula el servicio downstream.

```
┌─────────────────────────────────────────────────────────────────────┐
│  Test JVM                                                           │
│                                                                     │
│  WebTestClient ──→ [Gateway en puerto aleatorio]                    │
│                          │                                          │
│                    aplica predicates                                │
│                    aplica filtros                                   │
│                          │                                          │
│                          ↓                                          │
│                   WireMock Server                                   │
│                   (mock de downstream)                              │
└─────────────────────────────────────────────────────────────────────┘
```

El Gateway se levanta con `@SpringBootTest(webEnvironment = RANDOM_PORT)`, que inicia el contexto completo de WebFlux con puerto aleatorio. WireMock intercepta las peticiones que el Gateway reenvía al servicio downstream.

---

## Ejemplo central

El siguiente ejemplo muestra un test de integración completo para una ruta con `StripPrefix=1`, `AddRequestHeader` y un CircuitBreaker con fallback. Incluye la dependencia correcta de WireMock, la configuración de la ruta y las tres variantes de aserción más frecuentes en el examen.

```java
// pom.xml / build.gradle — dependencia de WireMock para Spring Cloud
// testImplementation 'org.springframework.cloud:spring-cloud-contract-wiremock'
// testImplementation 'org.springframework.boot:spring-boot-starter-test'

package com.example.gateway;

import com.github.tomakehurst.wiremock.client.WireMock;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.cloud.contract.wiremock.AutoConfigureWireMock;
import org.springframework.test.context.TestPropertySource;
import org.springframework.test.web.reactive.server.WebTestClient;

import static com.github.tomakehurst.wiremock.client.WireMock.*;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureWireMock(port = 0)  // WireMock en puerto aleatorio; inyecta wiremock.server.port
@TestPropertySource(properties = {
    // Override del URI del downstream para apuntar a WireMock
    "app.downstream-uri=http://localhost:${wiremock.server.port}"
})
class GatewayRoutingTest {

    @Autowired
    private WebTestClient webTestClient;

    @BeforeEach
    void setup() {
        // Stub: GET /api/productos → 200 OK
        stubFor(get(urlEqualTo("/api/productos"))
            .willReturn(aResponse()
                .withStatus(200)
                .withHeader("Content-Type", "application/json")
                .withBody("[{\"id\":1,\"nombre\":\"Producto A\"}]")));

        // Stub: GET /api/lento → simula timeout para CircuitBreaker
        stubFor(get(urlEqualTo("/api/lento"))
            .willReturn(aResponse()
                .withFixedDelay(5000)  // 5 segundos → dispara TimeLimiter
                .withStatus(200)));
    }

    // Test 1: StripPrefix=1 elimina /productos del path antes de reenviar
    @Test
    void stripPrefix_eliminaPrimerSegmentoDeRuta() {
        webTestClient.get()
            .uri("/productos/api/productos")   // path externo
            .exchange()
            .expectStatus().isOk()
            .expectBody()
            .jsonPath("$[0].nombre").isEqualTo("Producto A");

        // Verificar que WireMock recibió /api/productos (sin /productos)
        verify(getRequestedFor(urlEqualTo("/api/productos")));
    }

    // Test 2: AddRequestHeader inyecta cabecera antes de reenviar
    @Test
    void addRequestHeader_inyectaCabeceraEnDownstream() {
        stubFor(get(urlEqualTo("/api/productos"))
            .withHeader("X-Gateway-Source", equalTo("my-gateway"))
            .willReturn(aResponse().withStatus(200).withBody("[]")));

        webTestClient.get()
            .uri("/productos/api/productos")
            .exchange()
            .expectStatus().isOk();

        verify(getRequestedFor(urlEqualTo("/api/productos"))
            .withHeader("X-Gateway-Source", equalTo("my-gateway")));
    }

    // Test 3: CircuitBreaker fallback devuelve respuesta alternativa
    @Test
    void circuitBreaker_fallbackDevuelveRespuestaAlternativa() {
        webTestClient.get()
            .uri("/lento/api/lento")
            .exchange()
            .expectStatus().isOk()
            .expectBody(String.class)
            .isEqualTo("servicio no disponible");
    }
}
```

La configuración de rutas en `application.yml` de test referencia el placeholder `${app.downstream-uri}` que `@TestPropertySource` sobreescribe:

```yaml
# src/test/resources/application-test.yml
spring:
  cloud:
    gateway:
      routes:
        - id: productos-route
          uri: ${app.downstream-uri}
          predicates:
            - Path=/productos/**
          filters:
            - StripPrefix=1
            - AddRequestHeader=X-Gateway-Source, my-gateway

        - id: lento-route
          uri: ${app.downstream-uri}
          predicates:
            - Path=/lento/**
          filters:
            - StripPrefix=1
            - name: CircuitBreaker
              args:
                name: lentoCircuitBreaker
                fallbackUri: forward:/fallback

      # Endpoint de fallback interno
      default-filters: []

resilience4j:
  timelimiter:
    instances:
      lentoCircuitBreaker:
        timeout-duration: 1s
```

> [ADVERTENCIA] `@AutoConfigureWireMock(port = 0)` inyecta `${wiremock.server.port}` en el entorno de Spring. Si no se usa `@TestPropertySource` para sobreescribir el URI del downstream, las rutas del Gateway seguirán apuntando al host de producción durante los tests.

---

## Tabla de elementos clave

La tabla siguiente resume las herramientas y anotaciones que forman la base del testing de Gateway. Cada entrada incluye cuándo usarla y su limitación principal.

| Elemento | Tipo | Cuándo usarlo | Limitación |
|----------|------|---------------|------------|
| `WebTestClient` | Cliente reactivo | Siempre en tests de Gateway | No sirve para stacks Servlet |
| `@SpringBootTest(RANDOM_PORT)` | Anotación | Tests de integración completos | Arranque lento del contexto |
| `@AutoConfigureWireMock(port=0)` | Anotación | Mock de servicios downstream | Requiere `spring-cloud-contract-wiremock` |
| `${wiremock.server.port}` | Placeholder | Override de URI en properties | Solo disponible con `@AutoConfigureWireMock` |
| `WebTestClient.bindToServer()` | Factory method | Apuntar a servidor externo ya levantado | No integra con Spring context |
| `stubFor(...)` / `verify(...)` | API WireMock | Definir respuestas y verificar llamadas | Estado compartido entre tests si no se resetea |

> [EXAMEN] El examen distingue entre `spring-cloud-contract-wiremock` (starter para usar WireMock en tests de Gateway y Feign) y el módulo completo de Spring Cloud Contract (CDC, stubs, producer tests). Son dos cosas distintas con el mismo origen.

---

## Buenas y malas prácticas

**Hacer:**
- Usar `@AutoConfigureWireMock(port = 0)` con puerto aleatorio para evitar conflictos entre tests que corren en paralelo.
- Sobreescribir el URI del downstream con `@TestPropertySource` o `@DynamicPropertySource` para que las rutas apunten a WireMock en tests.
- Verificar con `WireMock.verify(...)` que el Gateway envió la petición transformada correctamente (headers añadidos, path stripeado), no solo que la respuesta llegó.
- Resetear el estado de WireMock entre tests con `WireMock.reset()` en `@BeforeEach` si los stubs varían entre tests.

**Evitar:**
- Usar `MockMvc` en tests de Gateway — razón: Gateway es WebFlux/Reactor; MockMvc no funciona con el stack reactivo y fallará al arrancar el contexto.
- Testear solo el happy path — razón: los filtros de resiliencia (CircuitBreaker, Retry, RateLimiter) solo se verifican con escenarios de fallo simulados (delays, status 5xx).
- Hardcodear puertos en WireMock (`port = 8089`) — razón: genera conflictos en entornos CI con tests paralelos.

---

## Comparación: estrategias de testing de Gateway

Existen tres estrategias según el alcance del test. La tabla muestra cuándo usar cada una.

| Estrategia | Scope | Herramientas | Cuándo usar |
|------------|-------|--------------|-------------|
| Test unitario de predicate/filter | Solo el bean | `MockServerWebExchange`, `MockServerRequest` | Verificar lógica de un `GatewayFilter` custom |
| Test de integración con WireMock | Gateway + downstream mock | `@SpringBootTest` + `@AutoConfigureWireMock` | Verificar rutas, filtros built-in, CircuitBreaker |
| Test E2E con servicios reales | Stack completo | Docker Compose, Testcontainers | Validar integración con Eureka, Config Server |

Para certificación, el escenario más frecuente es el test de integración con WireMock (fila 2).

---

## Verificación y práctica

Para verificar el dominio de este subtema, configura un proyecto Gateway con dos rutas: una con `StripPrefix=1` y otra con `CircuitBreaker` + `fallbackUri`. Escribe un test que verifique ambas rutas con `WebTestClient` y WireMock.

**Preguntas estilo examen VMware Spring Professional:**

1. ¿Por qué `MockMvc` no funciona para testear Spring Cloud Gateway y qué cliente debe usarse en su lugar?

   **Respuesta:** Spring Cloud Gateway está basado en Spring WebFlux (stack reactivo no bloqueante). `MockMvc` es para el stack Servlet. El cliente correcto es `WebTestClient`, que entiende el modelo reactivo de Reactor.

2. ¿Qué anotación configura automáticamente WireMock en un test de Spring Cloud Gateway y qué propiedad inyecta en el entorno?

   **Respuesta:** `@AutoConfigureWireMock(port = 0)` arranca un servidor WireMock con puerto aleatorio e inyecta la propiedad `${wiremock.server.port}` en el `Environment` de Spring, usable para sobreescribir URIs de downstream.

3. En un test con `@AutoConfigureWireMock`, las rutas del Gateway siguen apuntando al URI de producción. ¿Cuál es la causa y cómo se corrige?

   **Respuesta:** El URI del downstream está hardcodeado en `application.yml`. Se corrige con `@TestPropertySource(properties = {"app.downstream-uri=http://localhost:${wiremock.server.port}"})` o `@DynamicPropertySource` para sobreescribir la propiedad en tiempo de test.

4. ¿Cómo se verifica que un filtro `AddRequestHeader` realmente añadió la cabecera antes de reenviar la petición al downstream?

   **Respuesta:** Usando `WireMock.verify(getRequestedFor(...).withHeader("nombre", equalTo("valor")))` después de la llamada con `WebTestClient`. Solo WireMock puede verificar qué cabeceras llegaron al downstream real; la respuesta del `WebTestClient` no incluye esa información.

5. ¿Qué dependencia de test proporciona la integración de WireMock con Spring Boot para tests de Gateway?

   **Respuesta:** `org.springframework.cloud:spring-cloud-contract-wiremock`. No es el módulo de Spring Cloud Contract completo; es el starter de WireMock integrado con Spring Boot que provee `@AutoConfigureWireMock`.

---

← [5.11 Extensiones custom](sc-gateway-custom-extensiones.md) | [Índice](README.md) | [6.1 Spring Cloud Stream — Setup](sc-stream-setup.md) →
