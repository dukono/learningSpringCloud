# Parte 6 — API Gateway

← [Parte 5 — Load Balancing](./05-load-balancing.md) | [Volver al índice](./README.md) | Siguiente: [Parte 7 — Circuit Breaker](./07-circuit-breaker.md) →

---

## 6.1 Qué es un API Gateway y sus responsabilidades

Un **API Gateway** es el **punto de entrada único** para todos los clientes externos (browsers, apps móviles, servicios de terceros) a la arquitectura de microservicios.

Sin Gateway, los clientes tendrían que conocer la URL interna de cada microservicio:
```
# SIN GATEWAY — el cliente conoce todo
app móvil → http://10.0.0.1:8081/usuarios/...
app móvil → http://10.0.0.2:8082/productos/...
app móvil → http://10.0.0.3:8083/pedidos/...
```

Con Gateway, hay una única fachada:
```
# CON GATEWAY — el cliente solo conoce una URL
app móvil → https://api.miempresa.com/usuarios/...
app móvil → https://api.miempresa.com/productos/...
app móvil → https://api.miempresa.com/pedidos/...
```

### Responsabilidades del Gateway

| Responsabilidad | Descripción |
|---|---|
| **Routing** | Enruta peticiones al microservicio correcto según la ruta |
| **Autenticación** | Valida tokens JWT antes de dejar pasar la petición |
| **Rate Limiting** | Limita el número de peticiones por cliente/IP |
| **SSL Termination** | Gestiona HTTPS; los servicios internos usan HTTP |
| **CORS** | Centraliza la gestión de Cross-Origin |
| **Transformación** | Modifica headers, rutas, body de peticiones/respuestas |
| **Logging** | Registra todas las peticiones de entrada |
| **Circuit Breaker** | Protege los servicios de sobrecarga desde el Gateway |
| **Load Balancing** | Distribuye carga entre instancias del mismo servicio |

---

## 6.2 Spring Cloud Gateway (arquitectura reactiva)

Spring Cloud Gateway está construido sobre **Spring WebFlux** (reactivo, no-bloqueante), lo que le permite manejar gran concurrencia con pocos hilos.

### Diferencia con Zuul

| Aspecto | Zuul 1.x (antiguo) | Spring Cloud Gateway |
|---|---|---|
| **Modelo** | Bloqueante (Servlet) | No-bloqueante (Reactor/WebFlux) |
| **Rendimiento** | Limitado por hilos | Alto, maneja miles de conexiones concurrentes |
| **Estado** | Deprecado | Activo, mantenido |
| **Filtros** | ZuulFilter | GatewayFilter / GlobalFilter |

### Dependencia Maven

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>

<!-- IMPORTANTE: NO incluir spring-boot-starter-web junto con gateway -->
<!-- Gateway usa WebFlux; son incompatibles -->
```

### Clase principal

```java
@SpringBootApplication
public class GatewayApplication {
    // No necesita ninguna anotación especial
    public static void main(String[] args) {
        SpringApplication.run(GatewayApplication.class, args);
    }
}
```

---

## 6.3 Conceptos: Route, Predicate, Filter

Spring Cloud Gateway tiene tres conceptos fundamentales:

### Route (Ruta)

Una **Route** es la unidad básica de configuración. Define:
- Un **ID** único
- La **URI destino** (a dónde enrutar)
- Un conjunto de **Predicates** (condiciones que deben cumplirse)
- Un conjunto de **Filters** (transformaciones a aplicar)

```
Route: si la petición cumple los Predicates, aplicar los Filters y enrutar a la URI
```

### Predicate

Un **Predicate** es una condición que debe cumplir la petición para que la ruta aplique:

| Predicate | Ejemplo | Descripción |
|---|---|---|
| `Path` | `Path=/pedidos/**` | Por ruta URL |
| `Method` | `Method=GET,POST` | Por método HTTP |
| `Header` | `Header=X-Request-Id, \d+` | Por header (con regex) |
| `Query` | `Query=version, v2` | Por query parameter |
| `Host` | `Host=**.miempresa.com` | Por hostname |
| `After` | `After=2024-01-01T00:00:00Z` | Después de una fecha |
| `Before` | `Before=2024-12-31T23:59:59Z` | Antes de una fecha |
| `Weight` | `Weight=group1, 80` | Porcentaje de tráfico (A/B testing) |

### Filter

Un **Filter** transforma la petición antes de enviarla al servicio o la respuesta antes de devolverla al cliente:

| Filter | Tipo | Descripción |
|---|---|---|
| `AddRequestHeader` | Pre | Añade header a la petición |
| `AddResponseHeader` | Post | Añade header a la respuesta |
| `RewritePath` | Pre | Reescribe la URL |
| `StripPrefix` | Pre | Elimina segmentos del path |
| `CircuitBreaker` | Pre | Aplica circuit breaker |
| `RequestRateLimiter` | Pre | Limita peticiones |
| `Retry` | Pre | Reintenta en caso de error |
| `SetStatus` | Post | Cambia el HTTP status code |
| `RedirectTo` | Pre | Redirige a otra URL |

---

## 6.4 Configuración de rutas (YAML y Java DSL)

### Configuración en YAML (más común)

```yaml
spring:
  application:
    name: gateway-service

  cloud:
    gateway:
      routes:
        # Ruta para el servicio de pedidos
        - id: pedidos-route
          uri: lb://pedidos-service       # lb:// = usa LoadBalancer (Eureka)
          predicates:
            - Path=/api/pedidos/**
          filters:
            - StripPrefix=1               # elimina /api del path antes de enrutar
            # /api/pedidos/42 → /pedidos/42

        # Ruta para el servicio de productos
        - id: productos-route
          uri: lb://productos-service
          predicates:
            - Path=/api/productos/**
            - Method=GET                  # solo peticiones GET
          filters:
            - StripPrefix=1
            - AddRequestHeader=X-Gateway-Source, spring-cloud-gateway

        # Ruta con URL directa (sin Eureka)
        - id: servicio-externo
          uri: https://api.externo.com
          predicates:
            - Path=/externo/**
          filters:
            - RewritePath=/externo/(?<segment>.*), /${segment}
```

### Configuración en Java DSL

```java
@Configuration
public class GatewayConfig {

    @Bean
    public RouteLocator customRoutes(RouteLocatorBuilder builder) {
        return builder.routes()
            .route("pedidos-route", r -> r
                .path("/api/pedidos/**")
                .filters(f -> f
                    .stripPrefix(1)
                    .addRequestHeader("X-Gateway-Source", "spring-cloud-gateway")
                    .circuitBreaker(config -> config
                        .setName("pedidosCircuitBreaker")
                        .setFallbackUri("forward:/fallback/pedidos"))
                )
                .uri("lb://pedidos-service")
            )
            .route("productos-route", r -> r
                .path("/api/productos/**")
                .and().method(HttpMethod.GET)
                .filters(f -> f.stripPrefix(1))
                .uri("lb://productos-service")
            )
            .build();
    }
}
```

---

## 6.5 Filtros predefinidos más importantes

### StripPrefix — eliminar segmentos del path

```yaml
filters:
  - StripPrefix=1
# /api/pedidos/42 → /pedidos/42  (elimina el primer segmento)
```

### RewritePath — reescribir la URL

```yaml
filters:
  - RewritePath=/api/v1/(?<segmento>.*), /${segmento}
# /api/v1/pedidos/42 → /pedidos/42
```

### AddRequestHeader / AddResponseHeader

```yaml
filters:
  - AddRequestHeader=X-Request-Source, gateway
  - AddResponseHeader=X-Response-Time, ${date}
```

### RemoveRequestHeader

```yaml
filters:
  - RemoveRequestHeader=Cookie   # elimina cookies antes de pasar al microservicio
```

### SetPath — reemplazar el path completo

```yaml
filters:
  - SetPath=/v2/api/{segment}
```

### Retry — reintentos automáticos

```yaml
filters:
  - name: Retry
    args:
      retries: 3
      statuses: BAD_GATEWAY, SERVICE_UNAVAILABLE
      methods: GET, POST
      backoff:
        firstBackoff: 10ms
        maxBackoff: 50ms
        factor: 2
```

---

## 6.6 Filtros personalizados (GatewayFilter y GlobalFilter)

### GatewayFilter — aplicado a rutas específicas

```java
@Component
public class LoggingGatewayFilter implements GatewayFilter, Ordered {

    private static final Logger log = LoggerFactory.getLogger(LoggingGatewayFilter.class);

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        log.info("Petición entrante: {} {}", request.getMethod(), request.getURI());

        // Pre-filter: antes de enrutar al microservicio
        return chain.filter(exchange).then(Mono.fromRunnable(() -> {
            // Post-filter: cuando el microservicio ya respondió
            ServerHttpResponse response = exchange.getResponse();
            log.info("Respuesta: {}", response.getStatusCode());
        }));
    }

    @Override
    public int getOrder() {
        return -1;  // prioridad (menor número = mayor prioridad)
    }
}
```

### GlobalFilter — aplicado a TODAS las rutas

```java
@Component
public class AuthGlobalFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();

        // Verificar que tiene el header de autorización
        if (!request.getHeaders().containsKey("Authorization")) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }

        String authHeader = request.getHeaders().getFirst("Authorization");
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }

        // Token válido — continúa la cadena
        return chain.filter(exchange);
    }

    @Override
    public int getOrder() {
        return -100;  // ejecutar antes que otros filtros
    }
}
```

---

## 6.7 Rate Limiting con Redis

El filtro `RequestRateLimiter` limita el número de peticiones por cliente para proteger los microservicios de sobrecarga o abuso.

### Dependencia

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis-reactive</artifactId>
</dependency>
```

### Configuración

```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379

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
                # Algoritmo Token Bucket
                redis-rate-limiter.replenishRate: 10    # tokens/segundo generados
                redis-rate-limiter.burstCapacity: 20    # máximo burst permitido
                redis-rate-limiter.requestedTokens: 1   # tokens por petición
                key-resolver: "#{@ipKeyResolver}"       # quién es el "cliente"
```

### Key Resolver — identificar al cliente

```java
@Configuration
public class RateLimiterConfig {

    // Limitar por IP del cliente
    @Bean
    public KeyResolver ipKeyResolver() {
        return exchange -> Mono.just(
            exchange.getRequest().getRemoteAddress().getAddress().getHostAddress()
        );
    }

    // Limitar por usuario autenticado (JWT sub claim)
    @Bean
    public KeyResolver userKeyResolver() {
        return exchange -> Mono.justOrEmpty(
            exchange.getRequest().getHeaders().getFirst("X-User-Id")
        ).defaultIfEmpty("anonymous");
    }
}
```

---

## 6.8 Autenticación y autorización en el Gateway

El patrón más común es validar el JWT en el Gateway y propagar la identidad del usuario a los microservicios:

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

            // Propaga la identidad al microservicio como header
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
    public int getOrder() { return -200; }
}
```

---

## 6.9 Gateway vs Zuul (comparativa)

| Aspecto | Zuul 1.x | Spring Cloud Gateway |
|---|---|---|
| **Modelo de E/S** | Bloqueante (un hilo por conexión) | No-bloqueante (reactor) |
| **Framework base** | Spring MVC (Servlet) | Spring WebFlux |
| **Estado oficial** | Deprecado en Spring Cloud | Activo y mantenido |
| **Rendimiento bajo carga** | Limitado por pool de hilos | Excelente |
| **Configuración** | Java (ZuulFilter) | YAML o Java DSL fluido |
| **WebSocket** | Limitado | Soporte nativo |
| **Predicates** | No existe el concepto | Potentes y extensibles |
| **Compatibilidad** | spring-boot-starter-web | NO compatible con web; usa WebFlux |

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

---

← [Parte 5 — Load Balancing](./05-load-balancing.md) | [Volver al índice](./README.md) | Siguiente: [Parte 7 — Circuit Breaker](./07-circuit-breaker.md) →
