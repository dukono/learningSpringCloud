# 6.7.5 Tests de Circuit Breaker con WireMock

← [6.7.4 Tests de rate limiting](./06-31-gateway-testing-rate-limiting.md) | [Índice](./README.md) | [6.8.1 RouteDefinitionLocator →](./06-33-gateway-route-locator.md)

---

El Circuit Breaker del Gateway reacciona ante fallos del servicio downstream: cuando el servicio responde con errores o tarda demasiado, el circuito se abre y las peticiones siguientes se redirigen al fallback sin llegar al servicio. Testear este comportamiento requiere simular tres condiciones que no pueden reproducirse con un servicio real levantado: fallos controlados (status 500), latencia artificial (timeouts configurables) y recuperación programada. WireMock cubre las tres: puede responder con cualquier código de estado, añadir delays configurables a las respuestas, y cambiar el stub en mitad de la ejecución del test para simular la recuperación del servicio.

> [PREREQUISITO] Dependencia `spring-cloud-starter-circuitbreaker-reactor-resilience4j` y `wiremock-spring-boot`. La configuración del circuito con `resilience4j.*` en `application-test.yml` debe usar umbrales bajos (3-5 peticiones) para que el circuito se abra rápidamente en los tests.

## Diagrama: estados del circuito que los tests deben verificar

El flujo de transiciones tiene tres estados observables que los tests deben cubrir: `CLOSED` (tráfico normal al backend), `OPEN` (tráfico al fallback o 503 directo) y `HALF-OPEN` (sonda para detectar recuperación). Cada transición es desencadenada por una condición específica que WireMock puede simular de forma controlada.

```
Estado inicial: CLOSED
     │
     │  3 peticiones fallidas (WireMock devuelve 500)
     ▼
Estado: OPEN
     │  → Gateway redirige a fallbackUri sin llamar al servicio
     │  → Test verifica respuesta del fallback
     │
     │  waitDurationInOpenState expira (configurado a 1s en test)
     ▼
Estado: HALF-OPEN
     │  → Se permiten N peticiones de sonda
     │
     ├── Sonda falla: vuelve a OPEN
     │
     └── Sonda tiene éxito: transición a CLOSED
         → Test verifica que las peticiones vuelven al servicio
```

## Tests completos del Circuit Breaker

Cada método de test se centra en una transición de estado distinta: el test de fallos consecutivos verifica la apertura del circuito, el test de timeout verifica el `TimeLimiter`, y el test de recuperación verifica la transición `HALF-OPEN → CLOSED`. El perfil `test-cb` configura umbrales mínimos (`slidingWindowSize: 3`, `failureRateThreshold: 100`) para que cada transición sea provocada con exactamente el número de peticiones que indica el test.

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

import static com.github.tomakehurst.wiremock.client.WireMock.*;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test-cb")   // perfil con umbrales bajos del Circuit Breaker
@AutoConfigureWireMock(port = 0)
class CircuitBreakerTest {

    @Autowired
    private WebTestClient webTestClient;

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        r.add("app.backend.productos-url",
            () -> "http://localhost:${wiremock.server.port}");
    }

    @BeforeEach
    void resetStubs() {
        reset();
    }

    @Test
    void circuito_closed_peticion_exitosa_llega_al_servicio() {
        stubFor(get(urlEqualTo("/productos"))
            .willReturn(ok("{\"items\":[]}")
                .withHeader("Content-Type", "application/json")));

        webTestClient.get().uri("/api/v1/productos")
            .exchange()
            .expectStatus().isOk();

        verify(getRequestedFor(urlEqualTo("/productos")));
    }

    @Test
    void tres_fallos_consecutivos_abren_el_circuito_y_activan_fallback()
            throws InterruptedException {
        // Configurar el backend para que falle (500)
        stubFor(get(urlEqualTo("/productos"))
            .willReturn(serverError()
                .withBody("{\"error\": \"Internal Server Error\"}")));

        // slidingWindowSize=3 y failureRateThreshold=100 en test-cb:
        // 3 peticiones fallidas abren el circuito
        for (int i = 0; i < 3; i++) {
            webTestClient.get().uri("/api/v1/productos")
                .exchange()
                .expectStatus().is5xxServerError();
        }

        // A partir de aquí el circuito está OPEN: las peticiones van al fallback
        // El fallback devuelve 200 con body vacío (definido en el servicio de fallback)
        stubFor(get(urlEqualTo("/fallback/productos"))
            .willReturn(ok("{\"items\":[], \"fallback\": true}")
                .withHeader("Content-Type", "application/json")));

        webTestClient.get().uri("/api/v1/productos")
            .exchange()
            .expectStatus().isOk()
            .expectBody()
                .jsonPath("$.fallback").isEqualTo(true);

        // El servicio original no debe haber recibido la 4ª petición
        verify(3, getRequestedFor(urlEqualTo("/productos")));
    }

    @Test
    void timeout_abre_el_circuito() {
        // WireMock simula latencia de 5 segundos; el timeoutDuration es 2s en test-cb
        stubFor(get(urlEqualTo("/productos"))
            .willReturn(ok("{\"items\":[]}")
                .withFixedDelay(5_000)));    // 5000ms de delay

        stubFor(get(urlEqualTo("/fallback/productos"))
            .willReturn(ok("{\"items\":[], \"fallback\": true}")
                .withHeader("Content-Type", "application/json")));

        // El Gateway debe cortar la petición a los 2s y redirigir al fallback
        webTestClient.get().uri("/api/v1/productos")
            .exchange()
            .expectStatus().isOk()
            .expectBody()
                .jsonPath("$.fallback").isEqualTo(true);
    }

    @Test
    void circuito_open_sin_fallback_devuelve_503() {
        // Para la ruta sin fallback configurado:
        stubFor(get(urlEqualTo("/pagos"))
            .willReturn(serverError()));

        // 3 fallos para abrir el circuito de pagos
        for (int i = 0; i < 3; i++) {
            webTestClient.get().uri("/api/v1/pagos")
                .exchange()
                .expectStatus().is5xxServerError();
        }

        // Circuito abierto sin fallback → 503 directamente del Gateway
        webTestClient.get().uri("/api/v1/pagos")
            .exchange()
            .expectStatus().isEqualTo(503);
    }

    @Test
    void recuperacion_servicio_cierra_el_circuito_tras_half_open()
            throws InterruptedException {
        // Fase 1: abrir el circuito con fallos
        stubFor(get(urlEqualTo("/productos"))
            .willReturn(serverError()));

        for (int i = 0; i < 3; i++) {
            webTestClient.get().uri("/api/v1/productos")
                .exchange()
                .expectStatus().is5xxServerError();
        }

        // Fase 2: esperar que expire waitDurationInOpenState (1s en test-cb)
        Thread.sleep(1_500);

        // Fase 3: el servicio se recupera (cambiar stub a respuesta exitosa)
        reset();
        stubFor(get(urlEqualTo("/productos"))
            .willReturn(ok("{\"items\":[]}")
                .withHeader("Content-Type", "application/json")));

        // En estado HALF-OPEN, la petición de sonda tiene éxito → circuito CLOSED
        webTestClient.get().uri("/api/v1/productos")
            .exchange()
            .expectStatus().isOk();
    }
}
```

> [ADVERTENCIA] El test de recuperación usa `Thread.sleep(1_500)` para esperar que expire `waitDurationInOpenState`. Este sleep es necesario aquí porque el tiempo en estado OPEN es parte del comportamiento a testear. Configura `waitDurationInOpenState: 1s` en el perfil de test para minimizar el tiempo de espera sin hacerlo tan corto que cause flakiness por timing del scheduler de Resilience4j.

## Perfil de configuración para Circuit Breaker

Los umbrales de Resilience4j deben ser bajos en tests para que el circuito se abra con pocas peticiones:

```yaml
# src/test/resources/application-test-cb.yml
spring:
  autoconfigure:
    exclude:
      - org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration
      - org.springframework.boot.autoconfigure.data.redis.RedisReactiveAutoConfiguration

  cloud:
    gateway:
      routes:
        - id: productos-cb-route
          uri: ${app.backend.productos-url:http://localhost:8090}
          predicates:
            - Path=/api/v1/productos/**
          filters:
            - StripPrefix=2
            - name: CircuitBreaker
              args:
                name: productosCB
                fallbackUri: forward:///fallback/productos
                statusCodes:
                  - 500
                  - 503

        - id: productos-fallback-route
          uri: ${app.backend.productos-url:http://localhost:8090}
          predicates:
            - Path=/fallback/productos

        - id: pagos-cb-route
          uri: ${app.backend.productos-url:http://localhost:8090}
          predicates:
            - Path=/api/v1/pagos/**
          filters:
            - StripPrefix=2
            - name: CircuitBreaker
              args:
                name: pagosCB
                # Sin fallbackUri: devuelve 503 cuando está abierto

      discovery:
        locator:
          enabled: false

resilience4j:
  circuitbreaker:
    instances:
      productosCB:
        slidingWindowSize: 3            # solo 3 peticiones para calcular la tasa
        failureRateThreshold: 100       # 100%: 3/3 fallos abren el circuito
        waitDurationInOpenState: 1s     # esperar 1 segundo antes de HALF-OPEN
        permittedNumberOfCallsInHalfOpenState: 1
      pagosCB:
        slidingWindowSize: 3
        failureRateThreshold: 100
        waitDurationInOpenState: 1s
  timelimiter:
    instances:
      productosCB:
        timeoutDuration: 2s
      pagosCB:
        timeoutDuration: 2s

eureka:
  client:
    enabled: false

jwt:
  secret: clave-de-test-de-32-caracteres-min
```

> [EXAMEN] `failureRateThreshold: 100` con `slidingWindowSize: 3` significa que las 3 últimas peticiones deben haber fallado para abrir el circuito. En producción se usa un umbral del 50%, pero en tests el 100% hace los tests deterministas: exactamente 3 fallos abren el circuito, no más, no menos.

## Tabla de escenarios de Circuit Breaker a testear

Cubrir solo el happy path deja sin testear los casos donde el circuito actúa como mecanismo de protección. La tabla recoge los cinco escenarios mínimos que debe incluir una suite completa de Circuit Breaker, con la configuración WireMock correspondiente a cada uno.

| Escenario | Configuración WireMock | Estado resultante | Verificación |
|---|---|---|---|
| Servicio responde 200 | `willReturn(ok(...))` | CLOSED | `expectStatus().isOk()` + `verify(getRequestedFor(...))` |
| N fallos consecutivos | `willReturn(serverError())` × N | OPEN | Petición N+1 va al fallback |
| Timeout del servicio | `.withFixedDelay(ms > timeout)` | OPEN (TimeLimiter) | Respuesta del fallback en < timeout |
| Circuito OPEN sin fallback | Idem a fallos | OPEN | `expectStatus().isEqualTo(503)` |
| Recuperación (HALF-OPEN → CLOSED) | Cambiar stub a `ok()` tras sleep | CLOSED | Petición llega al backend, no al fallback |

## Buenas y malas prácticas

Hacer:
- Usar `failureRateThreshold: 100` y `slidingWindowSize` igual al número de fallos del test. Esto hace el test determinista: exactamente N fallos abren el circuito, sin ambigüedad.
- Nombrar los circuitos por el servicio que protegen (`productosCB`, `pagosCB`) para que las métricas de Actuator y los logs de los tests identifiquen qué circuito está fallando.
- Separar el test de timeout del test de error de respuesta. El TimeLimiter y el CircuitBreaker son mecanismos distintos y pueden tener configuraciones diferentes.

Evitar:
- Usar `waitDurationInOpenState` mayor de 2 segundos en tests. Cada test que espera la expiración del estado OPEN añade ese tiempo de espera. Con 5 tests y 10 segundos de espera cada uno, la suite tarda casi un minuto solo en esperas.
- No incluir el test del caso "circuito abierto sin fallback". Es tentador testear solo el happy path y el caso con fallback, pero el comportamiento sin fallback (503 directo) es un escenario real que debe estar documentado en los tests.

---

← [6.7.4 Tests de rate limiting](./06-31-gateway-testing-rate-limiting.md) | [Índice](./README.md) | [6.8.1 RouteDefinitionLocator →](./06-33-gateway-route-locator.md)
