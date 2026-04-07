# Parte 17 — Guía de Migración: Stack Legacy → Stack Moderno

← [Parte 16 — Saga](./16-saga-transactions.md) | [Volver al índice](./README.md)

---

## 17.1 Mapa de deprecaciones

| Componente Legacy | Estado | Reemplazo moderno |
|-------------------|--------|-------------------|
| **Hystrix** | Mantenimiento desde 2018 | Resilience4j |
| **Ribbon** | Deprecado en 2020 | Spring Cloud LoadBalancer |
| **Zuul 1.x** | Deprecado en 2020 | Spring Cloud Gateway |
| **Spring Cloud Sleuth** | Eliminado en Spring Boot 3 | Micrometer Tracing |
| **Feign (Netflix)** | Sustituido | Spring Cloud OpenFeign |
| **bootstrap.yml** | Deprecado | `spring.config.import` |
| **@EnableCircuitBreaker** | Eliminado | `@CircuitBreaker` (Resilience4j) |
| **@EnableFeignClients** sin cambios | Vigente | Sigue igual |

> `[ADVERTENCIA]` Si tu proyecto usa `spring-cloud-netflix-hystrix` o `spring-cloud-netflix-ribbon`, estas librerías no son compatibles con Spring Boot 3 / Jakarta EE. La migración es obligatoria para actualizar Java.

---

## 17.2 Hystrix → Resilience4j

### Dependencias

```xml
<!-- ELIMINAR -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>

<!-- AÑADIR -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
</dependency>
```

### Anotaciones

```java
// ANTES — Hystrix
@HystrixCommand(fallbackMethod = "fallbackProducto")
public Producto obtenerProducto(Long id) {
    return productosClient.get(id);
}

// DESPUÉS — Resilience4j
@CircuitBreaker(name = "productos-service", fallbackMethod = "fallbackProducto")
public Producto obtenerProducto(Long id) {
    return productosClient.get(id);
}

// La firma del fallback es idéntica en ambos casos
public Producto fallbackProducto(Long id, Throwable ex) {
    return Producto.noDisponible(id);
}
```

### Configuración

```yaml
# ANTES — Hystrix (en application.yml)
hystrix:
  command:
    productos-service:
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 3000
      circuitBreaker:
        requestVolumeThreshold: 5
        errorThresholdPercentage: 50
        sleepWindowInMilliseconds: 10000

# DESPUÉS — Resilience4j
resilience4j:
  circuitbreaker:
    instances:
      productos-service:
        slow-call-duration-threshold: 3s
        minimum-number-of-calls: 5
        failure-rate-threshold: 50
        wait-duration-in-open-state: 10s
  timelimiter:
    instances:
      productos-service:
        timeout-duration: 3s
```

### Tabla de equivalencias Hystrix → Resilience4j

| Hystrix | Resilience4j |
|---------|-------------|
| `requestVolumeThreshold` | `minimum-number-of-calls` |
| `errorThresholdPercentage` | `failure-rate-threshold` |
| `sleepWindowInMilliseconds` | `wait-duration-in-open-state` |
| `timeoutInMilliseconds` | `timeout-duration` (TimeLimiter) |
| `maxConcurrentRequests` | `max-concurrent-calls` (Bulkhead) |
| `@HystrixCommand` | `@CircuitBreaker` |
| `@EnableCircuitBreaker` | eliminar — ya no necesaria |

---

## 17.3 Ribbon → Spring Cloud LoadBalancer

### Dependencias

```xml
<!-- ELIMINAR — Ribbon se excluye explícitamente -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
        </exclusion>
    </exclusions>
</dependency>

<!-- AÑADIR -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-loadbalancer</artifactId>
</dependency>
```

### Configuración de RestTemplate

```java
// ANTES — Ribbon se activaba automáticamente con @LoadBalanced
@Bean
@LoadBalanced
public RestTemplate restTemplate() {
    return new RestTemplate();
}
// Ribbon leía la config desde ribbon.* properties

// DESPUÉS — Spring Cloud LoadBalancer, misma anotación, distinto motor
@Bean
@LoadBalanced  // misma anotación, ahora usa LoadBalancer en lugar de Ribbon
public RestTemplate restTemplate() {
    return new RestTemplate();
}
```

### Configuración

```yaml
# ANTES — Ribbon
productos-service:
  ribbon:
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule
    ConnectTimeout: 1000
    ReadTimeout: 3000

# DESPUÉS — Spring Cloud LoadBalancer
spring:
  cloud:
    loadbalancer:
      ribbon:
        enabled: false   # deshabilitar explícitamente si Ribbon sigue en classpath
```

### Algoritmo personalizado

```java
// ANTES — Ribbon
public class MiRegla extends AbstractLoadBalancerRule {
    @Override
    public Server choose(Object key) { ... }
}

// DESPUÉS — Spring Cloud LoadBalancer
public class MiLoadBalancer implements ReactorServiceInstanceLoadBalancer {

    private final ObjectProvider<ServiceInstanceListSupplier> supplier;

    @Override
    public Mono<Response<ServiceInstance>> choose(Request request) {
        return supplier.getIfAvailable()
            .get()
            .next()
            .map(instances -> {
                // lógica de selección personalizada
                ServiceInstance elegida = instances.get(0);
                return new DefaultResponse(elegida);
            });
    }
}

// Registrar:
@Configuration
@LoadBalancerClient(name = "productos-service", configuration = LoadBalancerConfig.class)
public class LoadBalancerConfig {
    @Bean
    public ReactorServiceInstanceLoadBalancer loadBalancer(
            ObjectProvider<ServiceInstanceListSupplier> supplier) {
        return new MiLoadBalancer(supplier);
    }
}
```

---

## 17.4 Zuul → Spring Cloud Gateway

### Dependencias

```xml
<!-- ELIMINAR -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
</dependency>

<!-- AÑADIR -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
<!-- Gateway es reactivo — requiere spring-boot-starter-webflux, NO spring-boot-starter-web -->
```

> `[ADVERTENCIA]` Gateway usa **Reactor/WebFlux** (no Servlet). Si tu proyecto tiene dependencias bloqueantes (JDBC, JPA directo en filtros), deben moverse a los microservicios de negocio, no al Gateway.

### Rutas

```yaml
# ANTES — Zuul
zuul:
  routes:
    productos:
      path: /api/productos/**
      serviceId: productos-service
      stripPrefix: true

# DESPUÉS — Gateway
spring:
  cloud:
    gateway:
      routes:
        - id: productos-route
          uri: lb://productos-service
          predicates:
            - Path=/api/productos/**
          filters:
            - StripPrefix=1
```

### Filtros personalizados

```java
// ANTES — Zuul (ZuulFilter)
@Component
public class AuthFilter extends ZuulFilter {
    @Override public String filterType() { return "pre"; }
    @Override public int filterOrder() { return 1; }
    @Override public boolean shouldFilter() { return true; }

    @Override
    public Object run() {
        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletRequest request = ctx.getRequest();
        String token = request.getHeader("Authorization");
        if (token == null) {
            ctx.setSendZuulResponse(false);
            ctx.setResponseStatusCode(401);
        }
        return null;
    }
}

// DESPUÉS — Gateway (GatewayFilter reactivo)
@Component
public class AuthFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String token = exchange.getRequest().getHeaders()
            .getFirst("Authorization");

        if (token == null) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }
        return chain.filter(exchange);
    }

    @Override
    public int getOrder() { return -1; }
}
```

---

## 17.5 Spring Cloud Sleuth → Micrometer Tracing

### Dependencias

```xml
<!-- ELIMINAR — Sleuth no existe en Spring Boot 3 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-sleuth</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-sleuth-zipkin</artifactId>
</dependency>

<!-- AÑADIR — Micrometer Tracing -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-brave</artifactId>
</dependency>
<dependency>
    <groupId>io.zipkin.reporter2</groupId>
    <artifactId>zipkin-reporter-brave</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

### Configuración

```yaml
# ANTES — Sleuth
spring:
  sleuth:
    sampler:
      probability: 1.0
  zipkin:
    base-url: http://zipkin:9411

# DESPUÉS — Micrometer Tracing
management:
  tracing:
    sampling:
      probability: 1.0
  zipkin:
    tracing:
      endpoint: http://zipkin:9411/api/v2/spans
```

### Acceso a Trace/Span ID en código

```java
// ANTES — Sleuth
@Autowired
private Tracer tracer;  // brave.Tracer de Sleuth

String traceId = tracer.currentSpan().context().traceIdString();

// DESPUÉS — Micrometer Tracing
@Autowired
private Tracer tracer;  // io.micrometer.tracing.Tracer

Span span = tracer.currentSpan();
if (span != null) {
    String traceId = span.context().traceId();
}
```

> `[CONCEPTO]` La API de `io.micrometer.tracing.Tracer` es casi idéntica a la de Sleuth. En la mayoría de casos, basta con cambiar el import y las dependencias.

---

## 17.6 bootstrap.yml → spring.config.import

`bootstrap.yml` era necesario para que el Config Client cargara la configuración remota antes de arrancar el contexto. En Spring Boot 2.4+ ya no es necesario.

```yaml
# ANTES — bootstrap.yml (archivo separado)
spring:
  application:
    name: productos-service
  cloud:
    config:
      uri: http://config-server:8888
      fail-fast: true

# DESPUÉS — application.yml (un solo archivo)
spring:
  application:
    name: productos-service
  config:
    import: "configserver:http://config-server:8888"
```

```xml
<!-- Si usas bootstrap.yml y quieres mantenerlo temporalmente durante la migración -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bootstrap</artifactId>
</dependency>
<!-- Esta dependencia re-habilita el soporte de bootstrap.yml -->
<!-- Úsala solo como puente durante la migración, no como destino final -->
```

---

## 17.7 Checklist de migración

### Antes de empezar

- [ ] Identificar qué versión de Spring Cloud usa el proyecto (`spring-cloud.version` en pom.xml)
- [ ] Verificar la [tabla de compatibilidad](https://spring.io/projects/spring-cloud) Spring Boot ↔ Spring Cloud
- [ ] Tener cobertura de tests antes de migrar (no migrar a ciegas)

### Migración paso a paso

- [ ] Actualizar `spring-boot-starter-parent` a 3.x
- [ ] Actualizar `spring-cloud.version` a 2023.x (Leyton)
- [ ] Sustituir `spring-cloud-starter-netflix-hystrix` por `circuitbreaker-resilience4j`
- [ ] Sustituir `spring-cloud-starter-netflix-ribbon` por `spring-cloud-starter-loadbalancer`
- [ ] Sustituir `spring-cloud-starter-netflix-zuul` por `spring-cloud-starter-gateway`
- [ ] Eliminar `spring-cloud-starter-sleuth` — añadir `micrometer-tracing-bridge-brave`
- [ ] Migrar `bootstrap.yml` a `spring.config.import` en `application.yml`
- [ ] Eliminar `@EnableCircuitBreaker`, `@EnableHystrix`, `@EnableZuulProxy`
- [ ] Cambiar imports `javax.*` → `jakarta.*` (cambio de Java EE a Jakarta EE)
- [ ] Ejecutar todos los tests

### Señales de que algo no migró bien

| Síntoma | Causa probable |
|---------|----------------|
| `ClassNotFoundException: com.netflix.hystrix.*` | Hystrix aún en classpath con Boot 3 |
| Circuit Breaker nunca se abre | Resilience4j mal configurado, revisar `minimum-number-of-calls` |
| Rutas del Gateway no funcionan | Conflicto `spring-web` + `spring-webflux` en classpath |
| Trazas no llegan a Zipkin | Falta `zipkin-reporter-brave` o endpoint mal configurado |
| Config Client no carga propiedades | `spring.config.import` faltante o `bootstrap` no habilitado |

---

← [Parte 16 — Saga](./16-saga-transactions.md) | [Volver al índice](./README.md)
