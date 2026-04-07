# Parte 15 — Testing de Microservicios con Spring Cloud

← [Parte 14 — Contract Testing](./14-contract-testing.md) | [Volver al índice](./README.md)

---

## 15.1 El problema del testing en microservicios

En una aplicación monolítica, un test de integración levanta una sola aplicación. En microservicios, una petición puede atravesar 5 servicios, 2 colas y una base de datos. ¿Cómo testear sin levantar todo el ecosistema?

```
Estrategia de testing en microservicios (pirámide adaptada):

        ┌─────────────┐
        │  E2E tests  │  ← pocos, lentos, frágiles
        ├─────────────┤
        │ Integration │  ← Testcontainers, WireMock
        │    tests    │
        ├─────────────┤
        │  Component  │  ← @SpringBootTest parcial
        │    tests    │
        ├─────────────┤
        │    Unit     │  ← muchos, rápidos, sin contexto Spring
        │    tests    │
        └─────────────┘
```

---

## 15.2 Niveles de test con Spring Boot

### Test unitario — sin contexto Spring

```java
// Rápido: no levanta contexto, no necesita anotación @SpringBootTest
class ProductoServiceTest {

    private ProductoRepository repo = mock(ProductoRepository.class);
    private ProductoService service = new ProductoService(repo);

    @Test
    void debeCalcularPrecioConDescuento() {
        when(repo.findById(1L)).thenReturn(Optional.of(
            new Producto(1L, "Laptop", BigDecimal.valueOf(1000))
        ));

        BigDecimal precio = service.precioConDescuento(1L, 10);

        assertThat(precio).isEqualByComparingTo("900.00");
    }
}
```

### Test de slice — solo capa web

```java
// @WebMvcTest levanta SOLO el contexto MVC (sin BD, sin servicios reales)
@WebMvcTest(ProductoController.class)
class ProductoControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean  // reemplaza el bean real en el contexto
    private ProductoService productoService;

    @Test
    void debeDevolver200CuandoProductoExiste() throws Exception {
        when(productoService.obtener(1L))
            .thenReturn(new Producto(1L, "Laptop", BigDecimal.valueOf(999)));

        mockMvc.perform(get("/productos/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.nombre").value("Laptop"));
    }

    @Test
    void debeDevolver404CuandoNoExiste() throws Exception {
        when(productoService.obtener(99L))
            .thenThrow(new ProductoNoEncontradoException(99L));

        mockMvc.perform(get("/productos/99"))
            .andExpect(status().isNotFound());
    }
}
```

### Test de repositorio — solo capa datos

```java
// @DataJpaTest levanta SOLO JPA + H2 en memoria (sin contexto web)
@DataJpaTest
class ProductoRepositoryTest {

    @Autowired
    private ProductoRepository repository;

    @Test
    void debePersistirYRecuperarProducto() {
        Producto guardado = repository.save(
            new Producto(null, "Monitor", BigDecimal.valueOf(299))
        );

        assertThat(guardado.getId()).isNotNull();
        assertThat(repository.findById(guardado.getId())).isPresent();
    }
}
```

---

## 15.3 Testcontainers — infraestructura real en tests

**Testcontainers** levanta contenedores Docker reales durante los tests. Elimina el problema de mock/prod divergence: el test usa la misma BD/broker que producción.

### Dependencias

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-testcontainers</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
    <scope>test</scope>
</dependency>
<!-- Módulos específicos -->
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>kafka</artifactId>
    <scope>test</scope>
</dependency>
```

### PostgreSQL con Testcontainers

```java
@SpringBootTest
@Testcontainers
class ProductoRepositoryIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        // Sobreescribe las propiedades de BD con los valores del contenedor
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private ProductoRepository repository;

    @Test
    void debeGuardarYRecuperarDesdePostgres() {
        Producto producto = repository.save(new Producto(null, "Teclado", BigDecimal.valueOf(79)));
        assertThat(repository.findById(producto.getId())).isPresent();
    }
}
```

### Kafka con Testcontainers

```java
@SpringBootTest
@Testcontainers
class PedidoEventoTest {

    @Container
    static KafkaContainer kafka = new KafkaContainer(
        DockerImageName.parse("confluentinc/cp-kafka:7.4.0")
    );

    @DynamicPropertySource
    static void kafkaProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
    }

    @Autowired
    private PedidoPublisher publisher;

    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    @Test
    void debePublicarEventoAlCrearPedido() throws Exception {
        publisher.publicar(new PedidoCreado(1L, "PENDIENTE"));

        // Verificar que el mensaje llegó al topic
        ConsumerRecord<String, String> record = KafkaTestUtils.getSingleRecord(
            consumer, "pedidos-topic", Duration.ofSeconds(5)
        );
        assertThat(record.value()).contains("PENDIENTE");
    }
}
```

### Configuración compartida con @TestConfiguration

```java
// Reutilizable en múltiples tests
@TestConfiguration(proxyBeanMethods = false)
class TestcontainersConfig {

    @Bean
    @ServiceConnection  // Spring Boot 3.1+ — configura propiedades automáticamente
    PostgreSQLContainer<?> postgresContainer() {
        return new PostgreSQLContainer<>("postgres:16");
    }

    @Bean
    @ServiceConnection
    KafkaContainer kafkaContainer() {
        return new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.4.0"));
    }
}
```

```java
// En cualquier test:
@SpringBootTest
@Import(TestcontainersConfig.class)
class MiTest {
    // BD y Kafka ya configurados automáticamente
}
```

> `[CONCEPTO]` `@ServiceConnection` (Spring Boot 3.1+) detecta el tipo de contenedor y configura las propiedades automáticamente. Elimina el `@DynamicPropertySource` manual.

---

## 15.4 WireMock — simular servicios externos

**WireMock** levanta un servidor HTTP falso que responde según reglas configuradas. Ideal para simular servicios de terceros o microservicios dependientes sin levantar Testcontainers completo.

### Dependencia

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-contract-wiremock</artifactId>
    <scope>test</scope>
</dependency>
```

### Uso básico con @AutoConfigureWireMock

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureWireMock(port = 0)  // puerto aleatorio
class PedidosServiceIntegrationTest {

    @Autowired
    private PedidosService pedidosService;

    @Value("${wiremock.server.port}")
    private int wireMockPort;

    @Test
    void debeObtenerProductoDesdeServicioExterno() {
        // Configurar stub de WireMock
        stubFor(get(urlEqualTo("/productos/1"))
            .willReturn(aResponse()
                .withStatus(200)
                .withHeader("Content-Type", "application/json")
                .withBody("""
                    {"id": 1, "nombre": "Laptop", "precio": 999.99}
                    """)));

        Producto producto = pedidosService.obtenerProducto(1L);

        assertThat(producto.getNombre()).isEqualTo("Laptop");
        verify(getRequestedFor(urlEqualTo("/productos/1")));
    }

    @Test
    void debeManejarErrorDelServicioExterno() {
        stubFor(get(urlPathMatching("/productos/.*"))
            .willReturn(aResponse()
                .withStatus(503)
                .withFixedDelay(5000)));  // simula timeout

        // El circuit breaker debería activar el fallback
        Producto producto = pedidosService.obtenerProducto(99L);
        assertThat(producto.isDisponible()).isFalse();
    }
}
```

### Stubs desde archivos JSON

```java
// Los stubs en src/test/resources/__files/ se cargan automáticamente
@AutoConfigureWireMock(port = 0, stubs = "classpath:/stubs/productos")
```

```json
// src/test/resources/__files/productos/get-producto-1.json
{
  "request": {
    "method": "GET",
    "url": "/productos/1"
  },
  "response": {
    "status": 200,
    "headers": { "Content-Type": "application/json" },
    "bodyFileName": "producto-1-response.json"
  }
}
```

---

## 15.5 Testear un cliente Feign

```java
@SpringBootTest
@AutoConfigureWireMock(port = 0)
class ProductoClientTest {

    @Autowired
    private ProductoClient productoClient;  // cliente Feign real

    @Test
    void debeDeserializarRespuestaCorrectamente() {
        stubFor(get("/productos/5")
            .willReturn(okJson("""
                {"id": 5, "nombre": "Ratón", "precio": 29.99, "stock": 100}
                """)));

        Producto p = productoClient.obtenerProducto(5L);

        assertThat(p.getId()).isEqualTo(5L);
        assertThat(p.getPrecio()).isEqualByComparingTo("29.99");
    }

    @Test
    void debeLanzarExcepcionCon404() {
        stubFor(get("/productos/404")
            .willReturn(notFound()));

        assertThatThrownBy(() -> productoClient.obtenerProducto(404L))
            .isInstanceOf(FeignException.NotFound.class);
    }
}
```

```yaml
# application-test.yml — apuntar Feign al WireMock
productos-service:
  url: http://localhost:${wiremock.server.port}
```

---

## 15.6 Testear filtros del API Gateway

```java
// @WebFluxTest para contexto reactivo del Gateway
@WebFluxTest
@Import(AuthenticationFilter.class)
class AuthenticationFilterTest {

    @Autowired
    private WebTestClient webTestClient;

    @MockBean
    private TokenValidator tokenValidator;

    @Test
    void debeRechazarPeticionSinToken() {
        webTestClient.get()
            .uri("/api/productos/1")
            .exchange()
            .expectStatus().isUnauthorized();
    }

    @Test
    void debePermitirPeticionConTokenValido() {
        when(tokenValidator.validate("valid-token")).thenReturn(true);

        webTestClient.get()
            .uri("/api/productos/1")
            .header("Authorization", "Bearer valid-token")
            .exchange()
            .expectStatus().isOk();
    }
}
```

---

## 15.7 Testear Circuit Breaker con Resilience4j

```java
@SpringBootTest
class CircuitBreakerTest {

    @Autowired
    private ProductoService productoService;

    @Autowired
    private CircuitBreakerRegistry circuitBreakerRegistry;

    @BeforeEach
    void resetCircuitBreaker() {
        circuitBreakerRegistry.circuitBreaker("productos-service")
            .reset();
    }

    @Test
    void debeAbrirCircuitoTrasUmbralDeFailures() {
        // Forzar 10 fallos consecutivos
        IntStream.range(0, 10).forEach(i -> {
            try { productoService.obtenerProducto(999L); }
            catch (Exception ignored) {}
        });

        CircuitBreaker cb = circuitBreakerRegistry
            .circuitBreaker("productos-service");

        assertThat(cb.getState())
            .isEqualTo(CircuitBreaker.State.OPEN);
    }

    @Test
    void debeUsarFallbackConCircuitoAbierto() {
        // Abrir el circuito forzosamente
        circuitBreakerRegistry
            .circuitBreaker("productos-service")
            .transitionToOpenState();

        // El fallback debe responder sin llamar al servicio real
        Producto resultado = productoService.obtenerProductoConFallback(1L);
        assertThat(resultado.isDisponible()).isFalse();
    }
}
```

---

## 15.8 Test de integración completo con @SpringBootTest

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
@Import(TestcontainersConfig.class)
@AutoConfigureWireMock(port = 0)
class PedidosIntegrationTest {

    @Autowired
    private TestRestTemplate restTemplate;

    @Autowired
    private PedidoRepository pedidoRepository;

    @BeforeEach
    void setup() {
        // Stub para productos-service
        stubFor(get("/productos/1")
            .willReturn(okJson("""
                {"id":1,"nombre":"Laptop","precio":999,"stock":5}
                """)));
    }

    @Test
    @Transactional
    void debeCrearPedidoCompleto() {
        CrearPedidoRequest request = new CrearPedidoRequest(1L, 2);

        ResponseEntity<PedidoResponse> response = restTemplate
            .postForEntity("/pedidos", request, PedidoResponse.class);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(pedidoRepository.count()).isEqualTo(1);
        assertThat(response.getBody().getEstado()).isEqualTo("PENDIENTE");
    }
}
```

---

## 15.9 Resumen: qué herramienta usar en cada caso

| Escenario | Herramienta | Anotación |
|-----------|-------------|-----------|
| Test unitario de lógica | JUnit + Mockito | ninguna |
| Test de controller REST | MockMvc | `@WebMvcTest` |
| Test de repositorio JPA | H2 in-memory | `@DataJpaTest` |
| Test con BD real | Testcontainers | `@Testcontainers` |
| Test con Kafka/RabbitMQ real | Testcontainers | `@Testcontainers` |
| Simular servicio HTTP externo | WireMock | `@AutoConfigureWireMock` |
| Test de cliente Feign | WireMock | `@AutoConfigureWireMock` |
| Test de filtros Gateway | WebTestClient | `@WebFluxTest` |
| Test de Circuit Breaker | CircuitBreakerRegistry | `@SpringBootTest` |
| Test de contrato | Spring Cloud Contract | `@AutoConfigureStubRunner` |

> `[ADVERTENCIA]` No uses `@SpringBootTest` para todo. Cada slice test (`@WebMvcTest`, `@DataJpaTest`) levanta un contexto reducido y es 5-10x más rápido.

> `[EXAMEN]` `@MockBean` reemplaza el bean en el contexto Spring. `mock()` de Mockito crea un mock fuera del contexto. En `@WebMvcTest` necesitas `@MockBean` para las dependencias del controller.

---

← [Parte 14 — Contract Testing](./14-contract-testing.md) | [Volver al índice](./README.md)
