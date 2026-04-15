# 6.7.4 Tests de rate limiting: Testcontainers Redis

← [6.7.3 Tests de filtros y seguridad](./06-30-gateway-testing-filtros.md) | [Índice](./README.md) | [6.7.5 Tests de Circuit Breaker →](./06-32-gateway-testing-circuit-breaker.md)

---

El rate limiting del Gateway almacena su estado en Redis mediante scripts Lua atómicos que no se pueden emular con un mock: el algoritmo Token Bucket requiere operaciones de lectura-escritura atómicas que solo un Redis real puede ejecutar correctamente. Por este motivo, los tests de rate limiting son los más lentos de la suite del Gateway: Testcontainers levanta un contenedor Redis 7 antes del primer test y lo mantiene hasta que termina la clase, pagando el coste del arranque del contenedor una sola vez para todos los tests de la clase.

> [PREREQUISITO] Docker disponible en el entorno de ejecución. Sin Docker, `@Testcontainers` no puede levantar el contenedor Redis y los tests fallan en la fase de arranque del contexto.

## Qué verifican los tests de rate limiting

Los tests de rate limiting verifican tres comportamientos del filtro `RequestRateLimiter` con un Redis real:

```
TESTS DE RATE LIMITING verifican:

  Test 1: Dentro del límite
  ┌─────────────────────────────────────────────────┐
  │  Bucket: capacidad=20, recarga=10 tok/s         │
  │  Peticiones 1-10: tokens disponibles → 200 OK   │
  └─────────────────────────────────────────────────┘

  Test 2: Superando el límite
  ┌─────────────────────────────────────────────────┐
  │  Llenamos el bucket con 20 peticiones           │
  │  Petición 21: sin tokens → 429 Too Many Requests│
  │  Respuesta incluye X-RateLimit-Remaining: 0     │
  └─────────────────────────────────────────────────┘

  Test 3: Distintas claves son independientes
  ┌─────────────────────────────────────────────────┐
  │  IP 192.168.1.1: 10 peticiones → bucket propio  │
  │  IP 192.168.1.2: 10 peticiones → bucket propio  │
  │  Ambas pueden hacer 10 peticiones sin 429       │
  └─────────────────────────────────────────────────┘
```

## Test completo con Testcontainers

La clase de test combina `@Testcontainers` con `@SpringBootTest` en modo `RANDOM_PORT` y `@AutoConfigureWireMock` para cubrir los tres comportamientos del filtro `RequestRateLimiter` en un único contexto: `@DynamicPropertySource` inyecta el host y el puerto asignado dinámicamente por Testcontainers para que Spring conecte con el Redis real del contenedor.

```java
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.cloud.contract.wiremock.AutoConfigureWireMock;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.springframework.test.web.reactive.server.WebTestClient;
import org.testcontainers.containers.GenericContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;
import org.testcontainers.utility.DockerImageName;

import static com.github.tomakehurst.wiremock.client.WireMock.*;

@Testcontainers
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test-ratelimit")   // perfil específico que activa Redis
@AutoConfigureWireMock(port = 0)
class RateLimiterIntegrationTest {

    // El contenedor se comparte entre todos los tests de la clase (static)
    @Container
    static GenericContainer<?> redis =
        new GenericContainer<>(DockerImageName.parse("redis:7-alpine"))
            .withExposedPorts(6379);

    @Autowired
    private WebTestClient webTestClient;

    @DynamicPropertySource
    static void redisProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.data.redis.host", redis::getHost);
        registry.add("spring.data.redis.port", redis::getFirstMappedPort);
        // Apuntar las rutas al WireMock del test
        registry.add("app.backend.productos-url",
            () -> "http://localhost:${wiremock.server.port}");
    }

    @BeforeEach
    void configurarStubs() {
        reset();
        // Stub genérico: el backend siempre responde 200
        stubFor(get(urlMatching("/productos.*"))
            .willReturn(ok("{\"items\":[]}")
                .withHeader("Content-Type", "application/json")));
    }

    @Test
    void dentro_del_limite_todas_las_peticiones_pasan() {
        // replenishRate=10 en application-test-ratelimit.yml
        // Bucket lleno al inicio: hasta 10 peticiones deben pasar
        for (int i = 0; i < 10; i++) {
            webTestClient.get().uri("/api/v1/productos")
                .header("X-Forwarded-For", "192.168.1.100")
                .exchange()
                .expectStatus().isOk();
        }
    }

    @Test
    void superando_el_limite_devuelve_429_con_headers_informativos() {
        // Vaciamos el bucket con burstCapacity peticiones
        for (int i = 0; i < 10; i++) {
            webTestClient.get().uri("/api/v1/productos")
                .header("X-Forwarded-For", "10.0.0.99")
                .exchange();
        }

        // La siguiente petición debe ser rechazada con 429
        webTestClient.get().uri("/api/v1/productos")
            .header("X-Forwarded-For", "10.0.0.99")
            .exchange()
            .expectStatus().isEqualTo(429)
            .expectHeader().exists("X-RateLimit-Remaining")
            .expectHeader().valueEquals("X-RateLimit-Remaining", "0");
    }

    @Test
    void distintas_ips_tienen_buckets_independientes() {
        // IP 1: consume su bucket propio
        for (int i = 0; i < 10; i++) {
            webTestClient.get().uri("/api/v1/productos")
                .header("X-Forwarded-For", "172.16.0.1")
                .exchange()
                .expectStatus().isOk();
        }

        // IP 2: tiene su propio bucket intacto
        for (int i = 0; i < 10; i++) {
            webTestClient.get().uri("/api/v1/productos")
                .header("X-Forwarded-For", "172.16.0.2")
                .exchange()
                .expectStatus().isOk();
        }
    }

    @Test
    void sin_ip_y_deny_empty_key_true_devuelve_403() {
        // El KeyResolver por IP no puede resolver la clave si no hay IP ni X-Forwarded-For
        // Con deny-empty-key=true (por defecto), el Gateway devuelve 403
        webTestClient.get().uri("/api/v1/productos")
            // Sin X-Forwarded-For y con remoteAddress no resoluble en test
            .exchange()
            .expectStatus().isForbidden();
    }
}
```

## Perfil de configuración específico para rate limiting

Los tests de rate limiting necesitan un perfil separado porque activan Redis (que el perfil `test` base desactiva):

```yaml
# src/test/resources/application-test-ratelimit.yml
spring:
  # No excluimos Redis: Testcontainers lo levanta y @DynamicPropertySource lo configura
  cloud:
    gateway:
      routes:
        - id: productos-rate-limited
          uri: ${app.backend.productos-url:http://localhost:8090}
          predicates:
            - Path=/api/v1/productos/**
          filters:
            - StripPrefix=2
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 10
                redis-rate-limiter.burstCapacity: 10  # igual al replenish: sin burst en tests
                redis-rate-limiter.requestedTokens: 1
                key-resolver: "#{@ipKeyResolver}"
                deny-empty-key: true

      discovery:
        locator:
          enabled: false

eureka:
  client:
    enabled: false

jwt:
  secret: clave-de-test-de-32-caracteres-min
```

> [EXAMEN] `burstCapacity == replenishRate` en los tests elimina la variabilidad del burst. Si `burstCapacity` fuera mayor que `replenishRate`, el test "superando el límite" necesitaría enviar `burstCapacity` peticiones para vaciar el bucket, no `replenishRate`. Igualarlos hace los tests predecibles y deterministas.

## Parámetros y configuración de Testcontainers

Los parámetros de Testcontainers relevantes para el contenedor Redis en tests de Gateway son los siguientes:

| Parámetro | Tipo | Descripción |
|---|---|---|
| `DockerImageName.parse("redis:7-alpine")` | String | Imagen Docker de Redis a usar. `alpine` es más ligera que la imagen base. |
| `.withExposedPorts(6379)` | int | Puerto que el contenedor expone; Testcontainers asigna un puerto aleatorio en el host. |
| `redis::getHost` | Supplier | Host donde escucha el contenedor (normalmente `localhost`). |
| `redis::getFirstMappedPort` | Supplier | Puerto aleatorio del host mapeado al 6379 del contenedor. |
| `static` en la declaración | — | Contenedor compartido entre todos los tests de la clase; sin `static`, se levanta uno por test. |

> [ADVERTENCIA] Si el `@Container` no es `static`, Testcontainers levanta y destruye un contenedor Redis por cada método de test. Con 10 tests, esto añade varios minutos al tiempo de ejecución. Declararlo `static` hace que se levante una vez por clase, que es el comportamiento correcto para tests de integración.

## Tabla de estrategias de test según el escenario de rate limiting

Los escenarios de rate limiting no son homogéneos: algunos requieren Redis real con Testcontainers mientras que otros pueden verificarse con mocks o con tests unitarios del `KeyResolver`. La tabla siguiente ayuda a elegir la estrategia adecuada según lo que se quiere verificar.

| Escenario | Cómo testear | Por qué |
|---|---|---|
| Límite básico (N peticiones → 429) | Loop con `WebTestClient` | Directo y determinista con `burstCapacity == replenishRate` |
| Buckets independientes por clave | Peticiones con distintos headers de IP o usuarios | Verifica el particionado correcto en Redis |
| `deny-empty-key` | Petición sin el header que usa el `KeyResolver` | Verifica el comportamiento cuando la clave no se puede extraer |
| Recuperación tras esperar | `Thread.sleep()` + nuevas peticiones | Verifica que el bucket se recarga; frágil en CI, evitar si es posible |
| `requestedTokens > 1` | Buckets con capacidad `N` y `requestedTokens=M` | Verifica que el coste por petición reduce el throughput efectivo |

## Buenas y malas prácticas

Hacer:
- Usar `burstCapacity == replenishRate` en los tests para hacer el límite predecible. Un bucket con `burstCapacity=20` y `replenishRate=10` requiere enviar 20 peticiones para agotarlo, no 10.
- Declarar el contenedor Redis como `static` para compartirlo entre todos los tests de la clase. El coste de arranque del contenedor (3-5 segundos) se paga una sola vez.
- Usar un perfil separado para los tests de rate limiting que no excluya Redis de la autoconfiguración. El perfil `test` base excluye Redis para que los tests de rutas y filtros sean más rápidos.

Evitar:
- Testear la recuperación del bucket con `Thread.sleep()` en tests de CI. El sleep introduce flakiness: si el sistema está bajo carga, el bucket puede no haberse recargado completamente cuando el test continúa.
- Usar el mismo rango de IPs para todos los tests sin resetear el estado de Redis entre tests. Las IPs usadas en un test que no agotó su bucket afectan a los tests siguientes si usan la misma IP.

---

← [6.7.3 Tests de filtros y seguridad](./06-30-gateway-testing-filtros.md) | [Índice](./README.md) | [6.7.5 Tests de Circuit Breaker →](./06-32-gateway-testing-circuit-breaker.md)
