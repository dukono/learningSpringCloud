# Parte 6.4 — API Gateway: Rate Limiting, Autenticación y Zuul

← [Filtros](./06-03-gateway-filtros.md) | [Volver al índice](./README.md) | Siguiente: [Parte 7 — Circuit Breaker →](./07-circuit-breaker.md)

---

## 6.7 Rate Limiting con Redis

Sin rate limiting, cualquier cliente puede enviar miles de peticiones por segundo. Las consecuencias son tres: abuso deliberado (scraping masivo, ataques de fuerza bruta), errores de cliente con bucles infinitos que generan tráfico no intencionado, y picos de uso legítimo en eventos especiales que saturan los microservicios. El rate limiting en el Gateway protege toda la arquitectura en un único punto, sin necesidad de que cada microservicio implemente su propia lógica de throttling.

El filtro `RequestRateLimiter` implementa el algoritmo **Token Bucket**: un cubo con capacidad máxima (`burstCapacity`) que se rellena a un ritmo constante (`replenishRate` tokens/segundo). Cada petición consume tokens. Si el cubo está vacío, la petición se rechaza con 429. Esto permite absorber ráfagas cortas de tráfico (`burstCapacity > replenishRate`) mientras se limita la velocidad media sostenida.

```
replenishRate=10, burstCapacity=20:
→ Permite hasta 20 peticiones instantáneas (ráfaga inicial)
→ A partir de ahí, solo 10 peticiones/segundo de forma sostenida
→ Si no se usan tokens, el cubo se recupera hasta el máximo de 20
```

> **[PREREQUISITO]** El rate limiting de Spring Cloud Gateway usa **Redis** como almacén distribuido del estado de los contadores. Sin Redis en ejecución, el filtro no funcionará.

### Dependencia

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis-reactive</artifactId>
    <!-- reactive es obligatorio: Gateway es reactivo y no puede usar el cliente síncrono -->
</dependency>
```

### Configuración completa

```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
      # Para Redis con autenticación:
      # password: ${REDIS_PASSWORD}
      # Para Redis Cluster:
      # cluster:
      #   nodes: redis-1:6379, redis-2:6379, redis-3:6379

  cloud:
    gateway:
      routes:
        - id: productos-route
          uri: lb://productos-service
          predicates:
            - Path=/api/productos/**
          filters:
            - name: RequestRateLimiter
              args:
                # Algoritmo Token Bucket:
                # se generan replenishRate tokens/segundo hasta un máximo de burstCapacity
                redis-rate-limiter.replenishRate: 10    # tokens generados por segundo (velocidad sostenida)
                redis-rate-limiter.burstCapacity: 20    # capacidad máxima del bucket (permite ráfagas)
                redis-rate-limiter.requestedTokens: 1   # tokens que consume cada petición (defecto: 1)
                key-resolver: "#{@ipKeyResolver}"       # bean que identifica al "cliente"
                # Si no hay tokens disponibles, devuelve 429 Too Many Requests
```

### Key Resolver — identificar al cliente

El `KeyResolver` determina quién es "el cliente" para el rate limiter. Se puede limitar por IP, usuario, API key o cualquier criterio:

```java
@Configuration
public class RateLimiterConfig {

    // Limitar por IP del cliente (más básico)
    @Bean
    public KeyResolver ipKeyResolver() {
        return exchange -> Mono.just(
            exchange.getRequest().getRemoteAddress().getAddress().getHostAddress()
        );
    }

    // Limitar por usuario autenticado (requiere que el JWT ya esté procesado)
    @Bean
    public KeyResolver userKeyResolver() {
        return exchange -> Mono.justOrEmpty(
            exchange.getRequest().getHeaders().getFirst("X-User-Id")
        ).defaultIfEmpty("anonymous");  // usuarios anónimos comparten el mismo bucket
    }

    // Limitar por API key en el header
    @Bean
    public KeyResolver apiKeyResolver() {
        return exchange -> Mono.justOrEmpty(
            exchange.getRequest().getHeaders().getFirst("X-API-Key")
        ).defaultIfEmpty("no-key");
    }

    // Limitar por ruta + usuario (granularidad máxima)
    @Bean
    public KeyResolver routeUserKeyResolver() {
        return exchange -> {
            String path = exchange.getRequest().getURI().getPath();
            String userId = exchange.getRequest().getHeaders()
                .getFirst("X-User-Id");
            return Mono.just(path + ":" + (userId != null ? userId : "anonymous"));
        };
    }
}
```

> **[EXAMEN]** El `KeyResolver` por defecto es `PrincipalNameKeyResolver` que lee el nombre del `Principal` de seguridad. Si no hay seguridad configurada, lanza excepción. Siempre declarar un `KeyResolver` propio.

### Comportamiento cuando el KeyResolver devuelve vacío

Si el `KeyResolver` devuelve `Mono.empty()` (por ejemplo, un usuario no autenticado sin cabecera `X-User-Id`), el comportamiento por defecto es **denegar la petición con 403 Forbidden**. Este comportamiento es configurable:

```yaml
spring:
  cloud:
    gateway:
      filter:
        request-rate-limiter:
          deny-empty-key: true          # defecto: true — deniega con 403 si KeyResolver devuelve vacío
                                        # false — permite pasar la petición sin aplicar rate limiting
          empty-key-status-code: 403    # código HTTP cuando deny-empty-key=true (defecto: 403)
                                        # Cambiar a 401 si se quiere indicar que falta autenticación
```

Ejemplo práctico: rutas con usuarios anónimos que quieres limitar bajo una clave común:

```java
@Bean
public KeyResolver userKeyResolver() {
    return exchange -> Mono.justOrEmpty(
        exchange.getRequest().getHeaders().getFirst("X-User-Id")
    ).defaultIfEmpty("anonymous");  // usuarios sin header usan el bucket "anonymous"
    // Con defaultIfEmpty, el KeyResolver nunca devuelve vacío → deny-empty-key no aplica
}
```

> Usar `defaultIfEmpty("anonymous")` en el `KeyResolver` es la forma más sencilla de manejar usuarios no autenticados: todos comparten el mismo bucket limitado, en lugar de ser rechazados con 403.

### Headers de respuesta del rate limiter

El filtro `RequestRateLimiter` añade automáticamente estos headers en **cada respuesta** (no solo en las bloqueadas), para que el cliente pueda adaptar su ritmo de llamadas:

| Header | Descripción |
|---|---|
| `X-RateLimit-Limit` | `replenishRate` — tokens generados por segundo |
| `X-RateLimit-Burst-Capacity` | `burstCapacity` — capacidad máxima del bucket |
| `X-RateLimit-Remaining` | Tokens disponibles en este momento |
| `X-RateLimit-Requested-Tokens` | Tokens consumidos por esta petición |

Cuando se supera el límite:
```
HTTP/1.1 429 Too Many Requests
X-RateLimit-Remaining: 0
X-RateLimit-Limit: 10
Retry-After: 1          ← segundos hasta que se regeneren tokens (no siempre presente)
```

### Rate Limiting sin Redis — implementación personalizada

Para entornos de una sola instancia (desarrollo, staging) o cuando no se quiere añadir Redis, se puede implementar un `RateLimiter` personalizado:

```java
@Component
public class InMemoryRateLimiter implements RateLimiter<InMemoryRateLimiter.Config> {

    // Map con contadores por clave (IP, usuario, etc.)
    private final Map<String, AtomicLong> counters = new ConcurrentHashMap<>();
    private final Map<String, Long> windowStart = new ConcurrentHashMap<>();

    @Override
    public Mono<Response> isAllowed(String routeId, String id) {
        long now = System.currentTimeMillis();
        long windowMs = 1000L;     // ventana de 1 segundo
        int maxRequests = 10;      // máximo 10 peticiones por ventana

        windowStart.putIfAbsent(id, now);
        counters.putIfAbsent(id, new AtomicLong(0));

        // Si ha pasado la ventana, resetear el contador
        if (now - windowStart.get(id) > windowMs) {
            windowStart.put(id, now);
            counters.get(id).set(0);
        }

        long count = counters.get(id).incrementAndGet();
        boolean allowed = count <= maxRequests;

        return Mono.just(new Response(allowed, Map.of(
            "X-RateLimit-Remaining", String.valueOf(Math.max(0, maxRequests - count)),
            "X-RateLimit-Limit", String.valueOf(maxRequests)
        )));
    }

    @Override
    public Class<Config> getConfigClass() { return Config.class; }

    @Override
    public Config newConfig() { return new Config(); }

    public static class Config {}
}
```

```yaml
# Usar el RateLimiter personalizado en la ruta
filters:
  - name: RequestRateLimiter
    args:
      rate-limiter: "#{@inMemoryRateLimiter}"
      key-resolver: "#{@ipKeyResolver}"
```

> **[ADVERTENCIA]** Un `RateLimiter` in-memory no funciona en despliegues con múltiples instancias del Gateway: cada instancia tiene su propio contador. Para clústeres, usar Redis.

---

## 6.8 Autenticación y autorización en el Gateway

La decisión de validar el JWT en el Gateway en lugar de en cada microservicio tiene implicaciones arquitectónicas que conviene entender antes de implementarla.

**A favor:** centraliza el código de seguridad en un único punto — si hay que actualizar la lógica de validación (cambiar el algoritmo, rotar la clave pública), se hace en un solo lugar. Los microservicios se simplifican: no necesitan dependencia de JWT, no necesitan conocer la clave de firma, y su lógica de negocio no está contaminada con código de seguridad.

**En contra:** si el Gateway es el único punto de validación, un microservicio accesible directamente sin pasar por el Gateway queda completamente desprotegido. También significa que el Gateway es un punto crítico: si falla, ningún servicio puede autenticar usuarios.

**La regla de producción:** si los microservicios son inaccesibles desde fuera excepto a través del Gateway (red privada, Kubernetes sin NodePort/LoadBalancer directo), la validación en el Gateway es suficiente y es la opción recomendada. Si algún microservicio puede recibir tráfico externo directo por cualquier motivo, debe validar el token también de forma independiente.

### Patrón recomendado: validar JWT en el Gateway y propagar identidad

El Gateway valida el token JWT centralizando la seguridad. Los microservicios no necesitan verificar el token, solo leer los headers que el Gateway propaga:

```
Cliente → [Token JWT] → Gateway → valida JWT → propaga X-User-Id, X-User-Roles → Microservicio
```

```xml
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.11.5</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.11.5</version>
    <scope>runtime</scope>
</dependency>
```

```java
@Component
public class JwtAuthFilter implements GlobalFilter, Ordered {

    @Value("${jwt.secret}")
    private String jwtSecret;

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String token = extraerToken(exchange.getRequest());

        if (token == null) {
            return unauthorized(exchange);
        }

        try {
            Claims claims = validarJwt(token);

            // Propaga la identidad al microservicio como headers
            // El microservicio confía en estos headers porque provienen del Gateway (red interna)
            ServerHttpRequest request = exchange.getRequest().mutate()
                .header("X-User-Id", claims.getSubject())
                .header("X-User-Roles", claims.get("roles", String.class))
                .build();

            return chain.filter(exchange.mutate().request(request).build());

        } catch (JwtException e) {
            return unauthorized(exchange);
        }
    }

    private String extraerToken(ServerHttpRequest request) {
        String authHeader = request.getHeaders().getFirst("Authorization");
        if (authHeader != null && authHeader.startsWith("Bearer ")) {
            return authHeader.substring(7);
        }
        return null;
    }

    private Mono<Void> unauthorized(ServerWebExchange exchange) {
        exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
        return exchange.getResponse().setComplete();
    }

    private Claims validarJwt(String token) {
        return Jwts.parserBuilder()
            .setSigningKey(jwtSecret.getBytes())
            .build()
            .parseClaimsJws(token)
            .getBody();
    }

    @Override
    public int getOrder() { return -200; }  // ejecutar antes de cualquier otro filtro
}
```

### Alternativa: Spring Security en el Gateway

El `JwtAuthFilter` manual con JJWT es simple pero tiene limitaciones: gestiona solo tokens JWT firmados con clave simétrica o asimétrica fija, no soporta rotación automática de claves (JWKS), y no tiene integración con el modelo de seguridad de Spring (`SecurityContext`, `@PreAuthorize`). Para casos que involucran OAuth2 (tokens emitidos por un servidor de autorización externo como Keycloak, Auth0, o Azure AD), OpenID Connect, o múltiples proveedores de identidad, Spring Security con OAuth2 Resource Server es la opción correcta: valida la firma contra el JWKS del servidor de autorización, gestiona la rotación automática de claves, y rellena el `SecurityContext` para que `TokenRelay` pueda propagar el token downstream.

Para casos más complejos (OAuth2, OpenID Connect, múltiples proveedores de identidad):

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
```

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth-server.miempresa.com   # discovery automático del JWKS
          # o directamente:
          # jwk-set-uri: https://auth-server.miempresa.com/.well-known/jwks.json
```

```java
@Configuration
@EnableWebFluxSecurity
public class SecurityConfig {

    @Bean
    public SecurityWebFilterChain springSecurityFilterChain(ServerHttpSecurity http) {
        http
            .authorizeExchange(exchanges -> exchanges
                .pathMatchers("/actuator/health", "/auth/**").permitAll()
                .pathMatchers("/admin/**").hasRole("ADMIN")
                .anyExchange().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()));
        return http.build();
    }
}
```

> **[ADVERTENCIA]** No configurar Spring Security en los microservicios internos si ya el Gateway los protege, a menos que los microservicios sean también accesibles directamente (sin pasar por el Gateway). En ese caso, sí deben tener su propia seguridad.

---

## 6.9 Ejemplo integrado completo

Configuración real de un Gateway con múltiples servicios, seguridad JWT, rate limiting y circuit breaker combinados:

```yaml
# application.yml del Gateway
server:
  port: 8080

spring:
  application:
    name: gateway-service

  data:
    redis:
      host: ${REDIS_HOST:localhost}
      port: 6379

  cloud:
    gateway:
      # Filtros aplicados a TODAS las rutas
      default-filters:
        - DedupeResponseHeader=Access-Control-Allow-Origin
        - AddResponseHeader=X-Gateway-Version, 1.0

      # CORS centralizado
      globalcors:
        cors-configurations:
          '[/**]':
            allowed-origins: "${ALLOWED_ORIGINS:http://localhost:3000}"
            allowed-methods: GET,POST,PUT,DELETE,OPTIONS
            allowed-headers: "*"
            allow-credentials: true

      routes:
        # Ruta pública — sin JWT, sin rate limiting estricto
        - id: auth-route
          uri: lb://auth-service
          predicates:
            - Path=/auth/**
          order: 1              # prioridad alta para que no lo capture la ruta genérica

        # Pedidos — JWT + rate limiting + circuit breaker
        - id: pedidos-route
          uri: lb://pedidos-service
          predicates:
            - Path=/api/pedidos/**
          filters:
            - StripPrefix=1
            - AddRequestHeader=X-Gateway-Source, gateway
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 20
                redis-rate-limiter.burstCapacity: 40
                key-resolver: "#{@userKeyResolver}"
            - name: CircuitBreaker
              args:
                name: pedidosCB
                fallbackUri: forward:/fallback/pedidos
            - name: Retry
              args:
                retries: 2
                statuses: BAD_GATEWAY
                methods: GET
          order: 10

        # Productos — JWT + rate limiting más restrictivo (proteger endpoint costoso)
        - id: productos-route
          uri: lb://productos-service
          predicates:
            - Path=/api/productos/**
            - Method=GET,POST
          filters:
            - StripPrefix=1
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 10
                redis-rate-limiter.burstCapacity: 15
                key-resolver: "#{@userKeyResolver}"
            - name: CircuitBreaker
              args:
                name: productosCB
                fallbackUri: forward:/fallback/productos
          order: 10

        # Admin — solo desde red interna
        - id: admin-route
          uri: lb://admin-service
          predicates:
            - Path=/admin/**
            - RemoteAddr=10.0.0.0/8
          filters:
            - AddRequestHeader=X-Admin-Request, true
          order: 5

      httpclient:
        connect-timeout: 2000
        response-timeout: 15s

      metrics:
        enabled: true

# GlobalFilter de JWT (ver 06-03-gateway-filtros.md)
jwt:
  secret: ${JWT_SECRET}

resilience4j:
  circuitbreaker:
    instances:
      pedidosCB:
        slidingWindowSize: 10
        failureRateThreshold: 50
        waitDurationInOpenState: 10s
      productosCB:
        slidingWindowSize: 10
        failureRateThreshold: 60
        waitDurationInOpenState: 5s

management:
  endpoints:
    web:
      exposure:
        include: health, gateway, prometheus
  endpoint:
    health:
      show-details: always
```

```java
// KeyResolver y FallbackController necesarios para la config anterior
@Configuration
public class GatewayBeans {

    @Bean
    public KeyResolver userKeyResolver() {
        return exchange -> Mono.justOrEmpty(
            exchange.getRequest().getHeaders().getFirst("X-User-Id")
        ).defaultIfEmpty("anonymous");
    }
}

@RestController
public class FallbackController {

    @RequestMapping("/fallback/{service}")
    public ResponseEntity<Map<String, String>> fallback(@PathVariable String service) {
        return ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE)
            .body(Map.of(
                "error", "service_unavailable",
                "service", service,
                "message", "El servicio no está disponible temporalmente"
            ));
    }
}
```

---

## 6.10 Testing del API Gateway

El Gateway tiene tres aspectos que necesitan cobertura de tests distinta: (1) que las rutas enrutan al destino correcto y aplican los filtros esperados; (2) que los filtros de seguridad (`JwtAuthFilter`, etc.) bloquean peticiones sin token y propagan los headers correctos al downstream; y (3) que la configuración de componentes externos (Redis para rate limiting, Eureka para discovery) no impide arrancar el test. Para el punto 3, la clave es aislar el Gateway de esas dependencias en los tests usando YAML de test que los deshabilita y WireMock para simular los microservicios downstream.

### Dependencias de test

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-contract-wiremock</artifactId>
    <scope>test</scope>
    <!-- Incluye WireMock para simular microservicios downstream -->
</dependency>
```

### Test de ruta básica con WebTestClient

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureWebTestClient
class GatewayRoutesTest {

    @Autowired
    private WebTestClient webTestClient;

    @Test
    void rutaPublicaNoRequiereToken() {
        webTestClient.get()
            .uri("/auth/login")
            .exchange()
            .expectStatus().isNotFound();   // 404 del servicio simulado, no 401 del Gateway
                                            // confirma que el Gateway NO bloqueó la petición
    }

    @Test
    void rutaProtegidaSinTokenDevuelve401() {
        webTestClient.get()
            .uri("/api/pedidos/42")
            .exchange()
            .expectStatus().isUnauthorized();
    }

    @Test
    void rutaProtegidaConTokenValidoEnruta() {
        webTestClient.get()
            .uri("/api/pedidos/42")
            .header("Authorization", "Bearer " + tokenValido())
            .exchange()
            .expectStatus().isOk()
            .expectHeader().exists("X-Gateway-Source");
    }
}
```

### Test de GlobalFilter con WireMock

El `JwtAuthFilter` actúa antes de contactar al microservicio. Para verificar que funciona correctamente, el test necesita dos cosas: controlar qué responde el microservicio downstream (para distinguir un 401 del Gateway de un 401 del servicio), y verificar que los headers que el filtro propaga (`X-User-Id`, `X-Gateway-Source`) llegaron realmente al servicio. WireMock cumple ambas funciones: simula el microservicio con la respuesta que se le configura, y registra las peticiones recibidas para poder inspeccionarlas con `verify()`.

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureWireMock(port = 0)   // puerto aleatorio, registrado en spring.cloud.gateway.routes
@AutoConfigureWebTestClient
class JwtAuthFilterTest {

    @Autowired
    private WebTestClient webTestClient;

    @BeforeEach
    void setUp() {
        // Simular microservicio downstream respondiendo 200
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
    void conTokenValidoRecibeRespuestaDelServicio() {
        webTestClient.get().uri("/api/pedidos/42")
            .header("Authorization", "Bearer " + generarToken("user-123", "ROLE_USER"))
            .exchange()
            .expectStatus().isOk()
            .expectBody()
                .jsonPath("$.id").isEqualTo(42);
    }

    @Test
    void filtroPropagoHeadersAlServicio() {
        webTestClient.get().uri("/api/pedidos/42")
            .header("Authorization", "Bearer " + generarToken("user-123", "ROLE_USER"))
            .exchange()
            .expectStatus().isOk();

        // Verificar que el microservicio recibió los headers propagados por el filtro
        verify(getRequestedFor(urlPathEqualTo("/pedidos/42"))
            .withHeader("X-User-Id", equalTo("user-123"))
            .withHeader("X-Gateway-Source", equalTo("gateway")));
    }
}
```

### Test de configuración de rutas con `@TestConfiguration`

Los tests de integración completos (`@SpringBootTest`) arrancan todo el contexto de Spring: seguridad, Redis, Eureka, Resilience4j. Para verificar únicamente que las rutas están correctamente configuradas (ID correcto, URI correcta, filtros presentes), ese coste es innecesario. `@WebFluxTest` limita el contexto a la capa web reactiva y permite verificar el `RouteLocator` directamente con programación reactiva (`StepVerifier`), siendo diez veces más rápido que un `@SpringBootTest` completo.

Para tests más rápidos (sin Spring Boot completo), se puede definir rutas de test en memoria:

```java
@WebFluxTest
@Import(GatewayConfig.class)
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
}
```

### Configuración para tests: deshabilitar componentes externos

Un Gateway de producción depende de Redis (rate limiting), Eureka (discovery) y Resilience4j (circuit breaker). Si estos componentes están activos en el test, el contexto no arranca a menos que haya instancias reales disponibles en CI. La solución es un `application-test.yml` en `src/test/resources` que sustituye las rutas `lb://` por URLs directas de WireMock y excluye las autoconfigurations de Redis, dejando solo lo que el test necesita verificar.

```yaml
# src/test/resources/application-test.yml
spring:
  cloud:
    gateway:
      # Usar rutas fijas apuntando a WireMock en lugar de Eureka
      routes:
        - id: pedidos-route
          uri: http://localhost:${wiremock.server.port}
          predicates:
            - Path=/api/pedidos/**
          filters:
            - StripPrefix=1
    loadbalancer:
      enabled: false   # deshabilitar Eureka en tests

  # Deshabilitar Redis para tests de rate limiting
  autoconfigure:
    exclude: org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration
```

---

## 6.11 Gateway vs Zuul (comparativa)

Zuul 1.x fue el gateway estándar de Spring Cloud hasta 2018 y sigue presente en proyectos legacy. La diferencia arquitectónica es fundamental: Zuul usa el modelo de Servlets bloqueantes (un hilo por conexión activa), lo que limita la concurrencia al tamaño del pool de hilos. Spring Cloud Gateway usa Reactor/WebFlux (un hilo maneja miles de conexiones mediante I/O no-bloqueante), lo que lo hace significativamente más eficiente bajo carga. Si se está migrando un sistema con Zuul, la configuración de rutas es directamente equivalente y el mayor esfuerzo está en reescribir los `ZuulFilter` como `GatewayFilter`/`GlobalFilter`.

| Aspecto | Zuul 1.x | Spring Cloud Gateway |
|---|---|---|
| **Modelo de E/S** | Bloqueante (un hilo por conexión) | No-bloqueante (reactor) |
| **Framework base** | Spring MVC (Servlet) | Spring WebFlux |
| **Estado oficial** | Deprecado en Spring Cloud | Activo y mantenido |
| **Rendimiento bajo carga** | Limitado por pool de hilos | Excelente |
| **Configuración** | Java (ZuulFilter) | YAML o Java DSL fluido |
| **WebSocket** | Limitado | Soporte nativo |
| **Predicates** | No existe el concepto | Potentes y extensibles |
| **Compatibilidad** | `spring-boot-starter-web` | NO compatible con web; usa WebFlux |

### Migración de Zuul a Gateway

```yaml
# ZUUL (antiguo)
zuul:
  routes:
    pedidos:
      path: /pedidos/**
      serviceId: pedidos-service
      stripPrefix: true

# GATEWAY (nuevo equivalente)
spring:
  cloud:
    gateway:
      routes:
        - id: pedidos-route
          uri: lb://pedidos-service
          predicates:
            - Path=/pedidos/**
          filters:
            - StripPrefix=1
```

La guía de migración completa de Zuul a Gateway (incluyendo equivalencias de `ZuulFilter` → `GatewayFilter`, `ZuulProperties` → `spring.cloud.gateway.*`, y el cambio de dependencia Maven) está en [17-migracion-legacy.md](./17-migracion-legacy.md).

---

← [Filtros](./06-03-gateway-filtros.md) | [Volver al índice](./README.md) | Siguiente: [Parte 7 — Circuit Breaker →](./07-circuit-breaker.md)
