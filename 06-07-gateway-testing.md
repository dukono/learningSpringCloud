# 6.7 Testing del Gateway

← [6.6 Seguridad: JWT y OAuth2](./06-06-gateway-seguridad.md) | [Índice](./README.md) | [6.8 Extensión Avanzada →](./06-08-gateway-extension.md)

---

El Gateway tiene tres aspectos que necesitan cobertura de tests distinta: (1) que las rutas enrutan al destino correcto y aplican los filtros esperados; (2) que los filtros de seguridad bloquean peticiones sin token y propagan los headers correctos al microservicio downstream; y (3) que la configuración de componentes externos (Redis, Eureka, Resilience4j) no impide arrancar el test. El principal obstáculo es que un Gateway de producción depende de Redis, Eureka y los microservicios destino, que no están disponibles en un test unitario. La estrategia es aislar el Gateway de esas dependencias en cada nivel de test.

> [PREREQUISITO] `spring-cloud-contract-wiremock` para simular microservicios downstream. `spring-boot-starter-test` incluye `WebTestClient` para tests reactivos.

---

## 6.7.1 Pirámide de tests del Gateway

```
                    ┌──────────┐
                    │  E2E     │  ← Gateway real + microservicios reales
                    │  (lento) │    Docker Compose / staging
                    └────┬─────┘
               ┌─────────┴─────────┐
               │  Integración      │  ← @SpringBootTest + WireMock
               │  (@SpringBootTest)│    Gateway real + microservicios simulados
               └─────────┬─────────┘
          ┌───────────────┴───────────────┐
          │  Slice / Config               │  ← @WebFluxTest
          │  (rápido, sin contexto completo)│   solo RouteLocator o filtros
          └───────────────────────────────┘
```

---

## 6.7.2 Estrategia 1: `@WebFluxTest` + `RouteLocator`

Para verificar que las rutas tienen el ID correcto, la URI correcta y los filtros esperados sin arrancar el contexto completo de Spring (sin Redis, sin Eureka, sin Resilience4j).

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

```java
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.reactive.WebFluxTest;
import org.springframework.cloud.gateway.route.RouteLocator;
import org.springframework.context.annotation.Import;
import reactor.test.StepVerifier;

import static org.assertj.core.api.Assertions.assertThat;

@WebFluxTest
@Import(GatewayRoutesConfig.class)   // solo importa la clase de configuración de rutas
class RouteConfigTest {

    @Autowired
    private RouteLocator routeLocator;

    @Test
    void debeExistirRutaDePedidos() {
        routeLocator.getRoutes()
            .filter(route -> "pedidos-route".equals(route.getId()))
            .as(StepVerifier::create)
            .assertNext(route -> {
                assertThat(route.getUri().toString()).isEqualTo("lb://pedidos-service");
                assertThat(route.getFilters()).isNotEmpty();
            })
            .verifyComplete();
    }

    @Test
    void rutaAdminDebeEvaluarseAntes() {
        routeLocator.getRoutes()
            .filter(route -> "admin-route".equals(route.getId()))
            .as(StepVerifier::create)
            .assertNext(route -> {
                assertThat(route.getMetadata().get("order")).isNotNull();
            })
            .verifyComplete();
    }
}
```

---

## 6.7.3 Estrategia 2: `@SpringBootTest` + WireMock

Para verificar que el `JwtAuthFilter` bloquea peticiones sin token, permite las válidas, y propaga correctamente los headers `X-User-Id` y `X-User-Roles` al microservicio downstream.

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-contract-wiremock</artifactId>
    <scope>test</scope>
</dependency>
```

```yaml
# src/test/resources/application-test.yml
# Sustituye rutas lb:// por URLs fijas apuntando a WireMock
spring:
  cloud:
    gateway:
      routes:
        - id: pedidos-route
          uri: http://localhost:${wiremock.server.port}
          predicates:
            - Path=/api/pedidos/**
          filters:
            - StripPrefix=1
    loadbalancer:
      enabled: false   # deshabilita Eureka en tests
  autoconfigure:
    exclude:
      - org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration
      - org.springframework.boot.autoconfigure.data.redis.RedisReactiveAutoConfiguration

jwt:
  secret: test-secret-key-32-characters-long!!
```

```java
import com.github.tomakehurst.wiremock.client.WireMock;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.security.Keys;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.cloud.contract.wiremock.AutoConfigureWireMock;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.web.reactive.server.WebTestClient;

import java.nio.charset.StandardCharsets;
import java.util.Date;
import java.util.Map;

import static com.github.tomakehurst.wiremock.client.WireMock.*;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureWireMock(port = 0)     // WireMock en puerto aleatorio; se registra en wiremock.server.port
@ActiveProfiles("test")
class JwtAuthFilterTest {

    @Autowired
    private WebTestClient webTestClient;

    @BeforeEach
    void configurarWireMock() {
        // Simular microservicio pedidos respondiendo 200
        stubFor(get(urlPathEqualTo("/pedidos/42"))
            .willReturn(aResponse()
                .withStatus(200)
                .withHeader("Content-Type", "application/json")
                .withBody("{\"id\": 42, \"estado\": \"activo\"}")));
    }

    @Test
    void sinTokenDevuelve401() {
        webTestClient.get().uri("/api/pedidos/42")
            .exchange()
            .expectStatus().isUnauthorized();
    }

    @Test
    void tokenInvalidoDevuelve401() {
        webTestClient.get().uri("/api/pedidos/42")
            .header("Authorization", "Bearer token-invalido")
            .exchange()
            .expectStatus().isUnauthorized();
    }

    @Test
    void conTokenValidoEnrutaAlMicroservicio() {
        webTestClient.get().uri("/api/pedidos/42")
            .header("Authorization", "Bearer " + generarToken("user-123", "ROLE_USER"))
            .exchange()
            .expectStatus().isOk()
            .expectBody()
            .jsonPath("$.id").isEqualTo(42)
            .jsonPath("$.estado").isEqualTo("activo");
    }

    @Test
    void filtroPropagoHeadersAlMicroservicio() {
        webTestClient.get().uri("/api/pedidos/42")
            .header("Authorization", "Bearer " + generarToken("user-123", "ROLE_USER"))
            .exchange()
            .expectStatus().isOk();

        // Verifica que WireMock recibió los headers propagados por el filtro
        verify(getRequestedFor(urlPathEqualTo("/pedidos/42"))
            .withHeader("X-User-Id", equalTo("user-123"))
            .withHeader("X-User-Roles", equalTo("ROLE_USER")));
    }

    private String generarToken(String userId, String roles) {
        String secret = "test-secret-key-32-characters-long!!";
        return Jwts.builder()
            .setSubject(userId)
            .addClaims(Map.of("roles", roles))
            .setExpiration(new Date(System.currentTimeMillis() + 60_000))
            .signWith(Keys.hmacShaKeyFor(secret.getBytes(StandardCharsets.UTF_8)))
            .compact();
    }
}
```

---

## 6.7.4 Estrategia 3: `@SpringBootTest` + Testcontainers Redis

Para verificar que el rate limiter devuelve 429 cuando se supera el límite, sin depender de un Redis real en el entorno de CI.

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
    <scope>test</scope>
</dependency>
```

```java
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.testcontainers.service.connection.ServiceConnection;
import org.springframework.cloud.contract.wiremock.AutoConfigureWireMock;
import org.springframework.test.web.reactive.server.WebTestClient;
import org.testcontainers.containers.GenericContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import static com.github.tomakehurst.wiremock.client.WireMock.*;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureWireMock(port = 0)
@Testcontainers
class RateLimiterTest {

    @Container
    @ServiceConnection
    static GenericContainer<?> redis =
        new GenericContainer<>("redis:7-alpine").withExposedPorts(6379);

    @Autowired
    private WebTestClient webTestClient;

    @Test
    void debeDevolver429AlSupeararLimite() {
        stubFor(get(urlPathMatching("/productos/.*"))
            .willReturn(aResponse().withStatus(200).withBody("{}")));

        // Enviar 11 peticiones con límite de 10
        for (int i = 0; i < 10; i++) {
            webTestClient.get().uri("/api/productos/123")
                .exchange()
                .expectStatus().isOk();
        }

        // La petición 11 debe ser rechazada
        webTestClient.get().uri("/api/productos/123")
            .exchange()
            .expectStatus().isEqualTo(429)
            .expectHeader().exists("X-RateLimit-Remaining");
    }
}
```

---

## 6.7.5 Estrategia 4: test del Circuit Breaker con WireMock

El Circuit Breaker actúa a nivel de Gateway, no de microservicio. Para verificar que devuelve el fallback cuando el servicio falla, WireMock simula el servicio devolviendo errores consecutivos hasta que el CB abre.

```java
import com.github.tomakehurst.wiremock.client.WireMock;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.cloud.contract.wiremock.AutoConfigureWireMock;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.web.reactive.server.WebTestClient;

import static com.github.tomakehurst.wiremock.client.WireMock.*;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureWireMock(port = 0)
@ActiveProfiles("test")
class CircuitBreakerGatewayTest {

    @Autowired
    private WebTestClient webTestClient;

    @BeforeEach
    void setUp() {
        // Configurar el servicio para que siempre falle con 503
        stubFor(get(urlPathMatching("/pedidos/.*"))
            .willReturn(aResponse().withStatus(503)));
    }

    @Test
    void cuandoServicioFallaDebeResponderConFallback() {
        // Enviar suficientes peticiones para abrir el Circuit Breaker
        // (slidingWindowSize=5 en la config de test con threshold=50%)
        for (int i = 0; i < 5; i++) {
            webTestClient.get().uri("/api/pedidos/1")
                .exchange()
                .expectStatus().is5xxServerError();
        }

        // Una vez el CB está abierto, debe responder con el fallback (503 del FallbackController)
        webTestClient.get().uri("/api/pedidos/1")
            .exchange()
            .expectStatus().isEqualTo(503)
            .expectBody()
            .jsonPath("$.error").isEqualTo("service_unavailable");
    }

    @Test
    void cuandoServicioRecuperaDebeVolverAResponder() {
        // Primero hacer que el servicio falle para abrir el CB
        for (int i = 0; i < 5; i++) {
            webTestClient.get().uri("/api/pedidos/1").exchange();
        }

        // Ahora configurar WireMock para que responda 200
        stubFor(get(urlPathMatching("/pedidos/.*"))
            .willReturn(aResponse().withStatus(200).withBody("{\"id\":1}")));

        // Esperar a que el CB entre en estado HALF-OPEN (waitDurationInOpenState=1s en test)
        try { Thread.sleep(1100); } catch (InterruptedException ignored) {}

        webTestClient.get().uri("/api/pedidos/1")
            .exchange()
            .expectStatus().isOk();
    }
}
```

---

## 6.7.6 `application-test.yml` — deshabilitar componentes externos

El fichero de configuración de tests desactiva componentes externos de forma selectiva. Cada exclusión tiene una razón concreta:

```yaml
# src/test/resources/application-test.yml
spring:
  cloud:
    gateway:
      # Sustituir rutas lb:// por URLs fijas de WireMock
      # WireMock registra su puerto en la propiedad wiremock.server.port
      routes:
        - id: pedidos-route
          uri: http://localhost:${wiremock.server.port}   # WireMock en vez de lb://pedidos-service
          predicates:
            - Path=/api/pedidos/**
          filters:
            - StripPrefix=1

        - id: productos-route
          uri: http://localhost:${wiremock.server.port}
          predicates:
            - Path=/api/productos/**
          filters:
            - StripPrefix=1

    # Deshabilitar Eureka: no hay registro de servicios en tests
    loadbalancer:
      enabled: false

  # Deshabilitar Redis: no hay Redis en tests unitarios/integración ligeros
  autoconfigure:
    exclude:
      - org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration
      - org.springframework.boot.autoconfigure.data.redis.RedisReactiveAutoConfiguration

# Clave JWT fija para tests — nunca usar en producción
jwt:
  secret: test-secret-key-minimum-32-characters!!

# Deshabilitar Resilience4j en tests para evitar falsos positivos por timeouts del CB
resilience4j:
  circuitbreaker:
    instances:
      pedidosCB:
        slidingWindowSize: 100
        failureRateThreshold: 100   # nunca abre el CB en tests
```

---

## 6.7.7 Tabla resumen: situación → estrategia recomendada

| Situación | Estrategia recomendada |
|---|---|
| Verificar que una ruta existe y tiene la URI correcta | `@WebFluxTest` + `RouteLocator` — más rápido, sin contexto completo |
| Verificar que un `GlobalFilter` bloquea peticiones inválidas | `@SpringBootTest` + WireMock + perfil `test` |
| Verificar que los headers se propagan correctamente al microservicio | `@SpringBootTest` + WireMock con `verify()` |
| Verificar que el rate limiter devuelve 429 | `@SpringBootTest` + Testcontainers Redis |
| Verificar el comportamiento del Circuit Breaker | `@SpringBootTest` + WireMock con respuestas de error configuradas |
| Test E2E con todos los componentes reales | Docker Compose en entorno de staging |

---

← [6.6 Seguridad: JWT y OAuth2](./06-06-gateway-seguridad.md) | [Índice](./README.md) | [6.8 Extensión Avanzada →](./06-08-gateway-extension.md)
