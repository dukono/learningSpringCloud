# 6.7.1 Estrategia de testing: pirámide y configuración application-test.yml

← [6.6.4 TokenRelay](./06-27-gateway-token-relay.md) | [Índice](./README.md) | [6.7.2 Tests de rutas →](./06-29-gateway-testing-rutas.md)

---

El Gateway concentra la lógica de enrutamiento, transformación, seguridad y resiliencia de toda la plataforma. Un bug en la configuración de rutas puede hacer que todas las peticiones de un servicio fallen silenciosamente; un error en el filtro de autenticación puede abrir rutas protegidas o bloquear rutas públicas. La estrategia de testing del Gateway debe cubrir cada una de estas capas de forma independiente, con el nivel de aislamiento adecuado para cada tipo de prueba.

## Pirámide de testing para el Gateway

La pirámide de testing del Gateway refleja el principio general de la pirámide de automatización: cuanto más bajo es el nivel, más pruebas hay, más rápidas son y menor es su coste de mantenimiento. Los tests de rutas son el nivel más bajo porque no levantan servidor; los de integración con WireMock están en el nivel medio; y los E2E contra un entorno real son los más costosos y, por tanto, los menos numerosos.

```
                    ┌──────────────┐
                    │ E2E / smoke  │  ← tests contra entorno real
                    │  (pocos)     │    (postman, k6)
                   /└──────────────┘\
                  /                  \
              ┌──────────────────────────┐
              │  Integración con mocks   │  ← @SpringBootTest + WireMock
              │  (filtros, seguridad,    │    (muchos, rápidos con mock)
              │  circuit breaker)        │
             /└──────────────────────────┘\
            /                              \
        ┌──────────────────────────────────────┐
        │         Tests de rutas               │  ← @WebFluxTest + RouteLocator
        │  (predicates, path transforms, order)│    (muchos, muy rápidos)
        └──────────────────────────────────────┘
```

| Capa | Herramienta | Lo que verifica | Velocidad |
|---|---|---|---|
| Rutas y predicates | `@WebFluxTest` + `RouteLocator` bean | Path matching, StripPrefix, RewritePath, order | Muy rápida (sin servidor) |
| Filtros y seguridad | `@SpringBootTest` + WireMock | Filtros personalizados, JWT, headers propagados | Rápida (servidor en puerto random) |
| Rate limiting | `@SpringBootTest` + Testcontainers Redis | Límites, KeyResolver, respuesta 429 | Media (levanta contenedor Redis) |
| Circuit Breaker | `@SpringBootTest` + WireMock | Estados del circuito, fallback, timeouts | Rápida (WireMock simula latencia) |
| E2E / smoke | Postman / REST Assured / k6 | Flujo completo en entorno real | Lenta (depende del entorno) |

## Configuración application-test.yml

El fichero `src/test/resources/application-test.yml` desactiva las dependencias externas que no deben requerirse en tests unitarios e de integración:

```yaml
spring:
  application:
    name: api-gateway-test

  # Desactiva Redis para tests que no lo necesiten
  # (sobrescribir con Testcontainers en tests específicos)
  autoconfigure:
    exclude:
      - org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration
      - org.springframework.boot.autoconfigure.data.redis.RedisReactiveAutoConfiguration

  cloud:
    gateway:
      # Desactiva el Discovery Locator en tests (no hay Eureka)
      discovery:
        locator:
          enabled: false

    # Desactiva Eureka en tests
    loadbalancer:
      ribbon:
        enabled: false

  # Desactiva el registro en Eureka
eureka:
  client:
    enabled: false

# Desactiva la validación JWT para tests de rutas simples
gateway:
  security:
    enabled: false

# Nivel de log más detallado en tests
logging:
  level:
    org.springframework.cloud.gateway: DEBUG
    reactor.netty: INFO
```

> [CONCEPTO] La separación entre `application.yml` (producción) y `application-test.yml` (tests) sigue el patrón de perfiles Spring. Los tests que necesiten configuración específica usan `@ActiveProfiles("test")` o declaran la propiedad en la anotación del test. Las propiedades de `application-test.yml` sobreescriben las del `application.yml` principal.

## Estructura de paquetes para tests del Gateway

La organización del directorio de tests refleja la separación por capa de responsabilidad: `routes/` agrupa los tests rápidos de `@WebFluxTest`, `filters/` los de `@SpringBootTest` con WireMock, `ratelimit/` los que necesitan Testcontainers, y `resilience/` los de Circuit Breaker. Esta separación permite ejecutar solo una capa durante el desarrollo sin levantar todos los contenedores.

```
src/test/java/
└── com/miempresa/gateway/
    ├── routes/
    │   ├── ProductosRouteTest.java       ← @WebFluxTest, rutas específicas
    │   └── AdminRouteTest.java
    ├── filters/
    │   ├── JwtAuthFilterTest.java        ← @SpringBootTest + WireMock
    │   ├── CorrelationIdFilterTest.java
    │   └── HmacValidationFilterTest.java
    ├── ratelimit/
    │   └── RateLimiterIntegrationTest.java  ← Testcontainers Redis
    └── resilience/
        └── CircuitBreakerTest.java       ← WireMock con latencia
```

## Dependencias de test

Las dependencias de test del Gateway cubren tres herramientas: `spring-boot-starter-test` incluye `WebTestClient` para el stack reactivo; `wiremock-spring-boot` integra WireMock con el ciclo de vida de Spring para simular los servicios backend; y `testcontainers` con `junit-jupiter` levanta contenedores Docker reales (Redis) durante los tests de rate limiting.

```xml
<!-- WebTestClient (incluido en spring-boot-starter-test para WebFlux) -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>

<!-- WireMock para mock de servicios backend -->
<dependency>
    <groupId>org.wiremock.integrations</groupId>
    <artifactId>wiremock-spring-boot</artifactId>
    <version>3.3.0</version>
    <scope>test</scope>
</dependency>

<!-- Testcontainers para Redis real en tests de rate limiting -->
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>testcontainers</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
    <scope>test</scope>
</dependency>
```

## Propiedades de test para WireMock

Cuando se usa WireMock para simular los servicios backend, las rutas del Gateway deben apuntar a la URL del servidor WireMock en lugar de a `lb://nombre-servicio`. Esto se gestiona con `@DynamicPropertySource`:

```java
@DynamicPropertySource
static void gatewayProperties(DynamicPropertyRegistry registry) {
    // Redirige las rutas del Gateway al servidor WireMock
    registry.add("app.backend.productos-url",
        () -> "http://localhost:" + wireMockServer.getPort());
}
```

Y en el YAML de test:

```yaml
# application-test.yml
app:
  backend:
    productos-url: http://localhost:8090  # valor por defecto; sobreescrito por @DynamicPropertySource
```

```yaml
# En las rutas del Gateway para tests:
spring:
  cloud:
    gateway:
      routes:
        - id: productos-route
          uri: ${app.backend.productos-url}  # apunta a WireMock en tests
          predicates:
            - Path=/api/productos/**
```

> [EXAMEN] El enfoque de usar `${app.backend.url}` en las rutas del Gateway permite usar el mismo YAML de rutas en tests y en producción, cambiando solo la propiedad que define la URI de destino. En producción sería `lb://productos-service`; en tests sería `http://localhost:PORT` del servidor WireMock.

## Buenas y malas prácticas

Hacer:
- Separar los tests por capa de responsabilidad: no mezclar tests de rutas (rápidos, sin servidor) con tests de filtros (más lentos, con servidor). El tiempo de ejecución total de la suite depende directamente de esta separación.
- Usar `@ActiveProfiles("test")` en la clase base de tests y heredar de ella para que todos los tests del Gateway hereden la configuración común.
- Verificar tanto el happy path como los casos de error en cada test: para un filtro de autenticación, el test debe cubrir token válido (200), token expirado (401), token malformado (401) y ruta pública sin token (200).

Evitar:
- Usar `@SpringBootTest` para todos los tests cuando `@WebFluxTest` es suficiente para los tests de rutas. `@SpringBootTest` levanta el contexto completo de Spring y es significativamente más lento.
- No limpiar el estado de WireMock entre tests. Las stubbing de un test que no se limpian pueden contaminar los tests siguientes con respuestas inesperadas.
- Hardcodear puertos en los tests (`localhost:8080`). Usar `RANDOM_PORT` en `@SpringBootTest` y el `webTestClient` inyectado para evitar conflictos entre tests concurrentes.

## Estrategias de verificación

### **Estrategia 1:** Tests de rutas con `@WebFluxTest` + `RouteLocator`

Verifica que los predicates de path, método y header coinciden correctamente, que los filtros de transformación (`StripPrefix`, `RewritePath`) modifican el path antes de enviarlo al backend, y que el orden de las rutas resuelve los conflictos de prefijo. Se usa cuando la lógica a verificar es solo el enrutador: sin contexto completo de seguridad ni servicios externos. Es la estrategia más rápida porque no levanta servidor.

```java
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.cloud.contract.wiremock.AutoConfigureWireMock;
import org.springframework.cloud.gateway.route.RouteLocator;
import org.springframework.cloud.gateway.route.builder.RouteLocatorBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.springframework.test.web.reactive.server.WebTestClient;

import static com.github.tomakehurst.wiremock.client.WireMock.*;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")
@AutoConfigureWireMock(port = 0)
class ProductosRouteTest {

    @Autowired WebTestClient webTestClient;

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        r.add("test.backend.url",
            () -> "http://localhost:${wiremock.server.port}");
    }

    @Test
    void stripPrefix_elimina_segmentos_y_backend_recibe_path_transformado() {
        stubFor(get(urlEqualTo("/productos/1")).willReturn(ok("{}")));

        webTestClient.get().uri("/api/v1/productos/1")
            .exchange().expectStatus().isOk();

        verify(getRequestedFor(urlEqualTo("/productos/1")));
    }

    @TestConfiguration
    static class Routes {
        @Bean
        RouteLocator routes(RouteLocatorBuilder b,
                @org.springframework.beans.factory.annotation.Value(
                    "${test.backend.url}") String url) {
            return b.routes()
                .route("prod", r -> r.path("/api/v1/productos/**")
                    .filters(f -> f.stripPrefix(2)).uri(url))
                .build();
        }
    }
}
```

---

### **Estrategia 2:** Tests de filtros y seguridad con `@SpringBootTest` + WireMock

Verifica el comportamiento del pipeline reactivo completo: que el `JwtAuthFilter` rechaza tokens inválidos antes de llamar al backend, que los headers de identidad se propagan correctamente, y que las rutas públicas no requieren autenticación. Se usa cuando la lógica involucra filtros personalizados que solo funcionan con el contexto completo de Spring. WireMock actúa como backend y permite verificar con `verify()` que el filtro cortó el flujo cuando es necesario.

```java
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.security.Keys;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.cloud.contract.wiremock.AutoConfigureWireMock;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.web.reactive.server.WebTestClient;

import javax.crypto.SecretKey;
import java.nio.charset.StandardCharsets;
import java.util.Date;
import java.util.List;

import static com.github.tomakehurst.wiremock.client.WireMock.*;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test-security")
@AutoConfigureWireMock(port = 0)
class JwtFilterTest {

    @Autowired WebTestClient client;
    @Value("${jwt.secret}") String secret;

    private String token(long expiresMs) {
        SecretKey key = Keys.hmacShaKeyFor(secret.getBytes(StandardCharsets.UTF_8));
        return Jwts.builder().subject("u1").claim("roles", List.of("ROLE_USER"))
            .issuedAt(new Date())
            .expiration(new Date(System.currentTimeMillis() + expiresMs))
            .signWith(key).compact();
    }

    @Test
    void token_valido_llega_al_backend_con_headers_de_identidad() {
        stubFor(get(urlEqualTo("/productos")).willReturn(ok("{}")));
        client.get().uri("/api/v1/productos")
            .header("Authorization", "Bearer " + token(60_000)).exchange()
            .expectStatus().isOk();
        verify(getRequestedFor(urlEqualTo("/productos"))
            .withHeader("X-User-Id", equalTo("u1")));
    }

    @Test
    void token_expirado_devuelve_401_y_el_backend_no_recibe_nada() {
        client.get().uri("/api/v1/productos")
            .header("Authorization", "Bearer " + token(-1_000)).exchange()
            .expectStatus().isUnauthorized();
        verify(0, getRequestedFor(urlEqualTo("/productos")));
    }
}
```

---

### **Estrategia 3:** Tests de rate limiting con Testcontainers Redis

Verifica que el Gateway aplica los límites configurados: que las primeras N peticiones dentro de la ventana reciben 200 y que la petición N+1 recibe 429, y que el `KeyResolver` segmenta los contadores correctamente por IP o por usuario. Se usa cuando la lógica de rate limiting es crítica para la plataforma y no puede simularse con mocks. Testcontainers levanta un Redis real en Docker durante el test y lo destruye al terminar, sin necesidad de infraestructura externa.

```java
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.springframework.test.web.reactive.server.WebTestClient;
import org.testcontainers.containers.GenericContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")
@Testcontainers
class RateLimiterTest {

    @Container
    static GenericContainer<?> redis =
        new GenericContainer<>("redis:7-alpine").withExposedPorts(6379);

    @DynamicPropertySource
    static void redisProps(DynamicPropertyRegistry r) {
        r.add("spring.data.redis.host", redis::getHost);
        r.add("spring.data.redis.port", () -> redis.getMappedPort(6379));
    }

    @Autowired WebTestClient client;

    @Test
    void peticion_que_supera_el_limite_devuelve_429() {
        // Asume replenishRate=1, burstCapacity=1 en la ruta de test
        client.get().uri("/api/v1/productos").exchange().expectStatus().isOk();
        client.get().uri("/api/v1/productos").exchange()
            .expectStatus().isEqualTo(429);
    }
}
```

---

### **Estrategia 4:** Tests de Circuit Breaker con WireMock

Verifica que el Circuit Breaker del Gateway responde con el fallback configurado cuando el backend supera el timeout o devuelve errores repetidos, y que el circuito pasa a estado OPEN después de N fallos consecutivos. Se usa cuando la resiliencia frente a backends lentos o caídos es un requisito crítico. WireMock simula la latencia o el error del backend con `withFixedDelay()` o `.willReturn(serverError())`, sin necesidad de un servicio real que falle de forma controlada.

```java
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.cloud.contract.wiremock.AutoConfigureWireMock;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.web.reactive.server.WebTestClient;

import static com.github.tomakehurst.wiremock.client.WireMock.*;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")
@AutoConfigureWireMock(port = 0)
class CircuitBreakerTest {

    @Autowired WebTestClient client;

    @Test
    void backend_lento_activa_fallback_en_lugar_de_timeout() {
        // WireMock simula un backend que tarda más que el timeout del circuito
        stubFor(get(urlEqualTo("/productos"))
            .willReturn(ok("{}").withFixedDelay(5_000))); // 5 s > timeout configurado

        client.get().uri("/api/v1/productos")
            .exchange()
            // El fallback devuelve 503 o la ruta de fallback configurada
            .expectStatus().isEqualTo(503);
    }

    @Test
    void backend_con_errores_repetidos_activa_circuito_abierto() {
        // Genera suficientes fallos para que el circuito pase a OPEN
        stubFor(get(urlEqualTo("/productos")).willReturn(serverError()));

        for (int i = 0; i < 5; i++) {
            client.get().uri("/api/v1/productos").exchange();
        }

        // Con el circuito OPEN, la siguiente petición recibe la respuesta de fallback sin llamar al backend
        client.get().uri("/api/v1/productos")
            .exchange().expectStatus().isEqualTo(503);

        // El backend no debe haber recibido la última petición
        verify(lessThanOrExactly(5), getRequestedFor(urlEqualTo("/productos")));
    }
}
```

---

### **Estrategia 5:** Tests E2E / smoke (Postman, k6)

Verifica el flujo completo de extremo a extremo contra un entorno real o de staging, donde todos los servicios (Gateway, backends, Redis, Eureka) están levantados con su configuración real. Se usa como validación final antes de un despliegue en producción o como smoke test tras el despliegue para confirmar que el entorno está sano. A diferencia de las estrategias 1-4, no se aísla ninguna capa: se comprueba que una petición real de un cliente llega al backend correcto con los headers esperados y la respuesta correcta.

Las herramientas habituales para esta estrategia son:

- **Postman / Newman**: colecciones de requests con assertions sobre status code, cuerpo y headers. Se ejecutan en CI con `newman run collection.json --environment env-staging.json`.
- **k6**: scripts en JavaScript que combinan verificaciones funcionales (check de status y cuerpo) con métricas de rendimiento (percentiles de latencia, tasa de error). Útil para validar que el rate limiting y el Circuit Breaker funcionan bajo carga real.
- **REST Assured** (si se prefiere Java): misma aproximación que WebTestClient pero contra la URL del entorno real, sin contexto de Spring en el test.

Los smoke tests deben cubrir al menos: una ruta pública sin autenticación, una ruta protegida con token válido, una ruta protegida con token inválido (espera 401), y el endpoint de health de Actuator.

---

## Tabla resumen: situación y estrategia recomendada

La tabla siguiente resume qué estrategia aplicar según lo que se quiere verificar, permitiendo elegir el nivel de aislamiento adecuado sin sobreinstrumentar los tests.

| Situación | Estrategia recomendada |
|---|---|
| Verificar que un predicate de path o `StripPrefix` funciona correctamente | Estrategia 1: `@SpringBootTest` + `RouteLocator` Java DSL + WireMock |
| Verificar que un filtro JWT rechaza tokens inválidos sin llamar al backend | Estrategia 2: `@SpringBootTest` + WireMock + `verify(0, ...)` |
| Verificar que el header de identidad se propaga correctamente al backend | Estrategia 2: `@SpringBootTest` + WireMock + `withHeader(...)` |
| Verificar que la petición N+1 recibe 429 después de superar el límite | Estrategia 3: `@SpringBootTest` + Testcontainers Redis |
| Verificar que el `KeyResolver` segmenta contadores por IP o usuario | Estrategia 3: `@SpringBootTest` + Testcontainers Redis |
| Verificar que el fallback se activa cuando el backend supera el timeout | Estrategia 4: `@SpringBootTest` + WireMock con `withFixedDelay()` |
| Verificar que el circuito pasa a OPEN tras N errores consecutivos | Estrategia 4: `@SpringBootTest` + WireMock con `serverError()` |
| Validar el entorno de staging antes de un despliegue | Estrategia 5: Postman / Newman o k6 contra la URL real |
| Smoke test post-despliegue en producción | Estrategia 5: k6 o REST Assured contra la URL de producción |

---

← [6.6.4 TokenRelay](./06-27-gateway-token-relay.md) | [Índice](./README.md) | [6.7.2 Tests de rutas →](./06-29-gateway-testing-rutas.md)
