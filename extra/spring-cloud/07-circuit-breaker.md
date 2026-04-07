# Parte 7 — Circuit Breaker (Tolerancia a fallos)

← [Parte 6 — API Gateway](./06-api-gateway.md) | [Volver al índice](./README.md) | Siguiente: [Parte 8 — Comunicación](./08-comunicacion.md) →

---

## 7.1 El problema: fallos en cascada

En una arquitectura de microservicios, los servicios se llaman entre sí. Si un servicio se vuelve lento o no disponible, puede provocar un **fallo en cascada** que derrumba todo el sistema:

```
Usuario → Gateway → Pedidos → Productos → Inventario
                                  ↑
                              Inventario tarda 30s en responder

RESULTADO:
- Productos acumula hilos esperando Inventario → pool exhausto
- Pedidos acumula hilos esperando Productos → pool exhausto
- Gateway acumula hilos esperando Pedidos → pool exhausto
- El sistema entero se cuelga por un único servicio lento
```

Este patrón se llama **cascading failure** y es uno de los problemas más graves en arquitecturas distribuidas.

**La solución:** detectar el fallo y "cortar el circuito" rápidamente, devolviendo una respuesta alternativa (fallback) en lugar de esperar indefinidamente.

---

## 7.2 Patrón Circuit Breaker explicado

El **Circuit Breaker** (disyuntor eléctrico) actúa como un proxy que monitoriza las llamadas a un servicio remoto. Cuando detecta un porcentaje elevado de fallos, "abre el circuito" y devuelve un fallback inmediatamente sin intentar la llamada real.

### Analogía con el mundo real

En electricidad, un disyuntor corta el circuito cuando hay sobrecarga para evitar que el cable se queme. Cuando se soluciona el problema, se puede "reconectar" (HALF-OPEN).

---

## 7.3 Estados: CLOSED, OPEN, HALF-OPEN

```
                 fallo > umbral
    CLOSED ─────────────────────→ OPEN
      ↑                              │
      │      timeout pasado          │
      ←─── HALF-OPEN ←──────────────┘
              │      │
         éxito│      │fallo
              │      │
           CLOSED  OPEN
```

### CLOSED (circuito cerrado — normal)

- Las llamadas pasan normalmente al servicio real
- El Circuit Breaker monitoriza: cuenta éxitos y fallos
- Si la tasa de fallos supera el umbral configurado → transición a OPEN

### OPEN (circuito abierto — protegido)

- **Ninguna llamada llega al servicio real**
- Se devuelve inmediatamente el fallback configurado
- El sistema no desperdicia recursos esperando respuestas
- Después de un tiempo de espera (`waitDurationInOpenState`) → transición a HALF-OPEN

### HALF-OPEN (semi-abierto — prueba)

- Se deja pasar un **número limitado de llamadas de prueba** al servicio real
- Si las llamadas de prueba tienen éxito → transición a CLOSED (servicio recuperado)
- Si las llamadas de prueba fallan → vuelve a OPEN

---

## 7.4 Resilience4j en Spring Cloud

**Hystrix** (Netflix OSS) era el Circuit Breaker original en Spring Cloud. Netflix lo puso en modo mantenimiento en 2018. Su reemplazo oficial es **Resilience4j**.

### Módulos de Resilience4j

| Módulo | Descripción |
|---|---|
| `CircuitBreaker` | Corta el circuito ante fallos repetidos |
| `Retry` | Reintenta la llamada ante fallos transitorios |
| `TimeLimiter` | Limita el tiempo de espera de una llamada |
| `Bulkhead` | Limita la concurrencia (pool de hilos o semáforo) |
| `RateLimiter` | Limita el número de llamadas por periodo |
| `Cache` | Cachea resultados para reducir llamadas |

### Dependencias Maven

```xml
<!-- Spring Cloud Circuit Breaker con Resilience4j -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
</dependency>

<!-- Para actuator y métricas -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

---

## 7.5 Configuración de CircuitBreaker con Resilience4j

### Configuración en application.yml

```yaml
resilience4j:
  circuitbreaker:
    instances:
      # Nombre del circuit breaker (referenciado en el código)
      productos-service:
        # UMBRAL DE FALLO
        failure-rate-threshold: 50          # % de fallos para abrir (defecto: 50)
        slow-call-rate-threshold: 100       # % de llamadas lentas para abrir
        slow-call-duration-threshold: 2s    # qué se considera "lento"

        # VENTANA DE EVALUACIÓN
        sliding-window-type: COUNT_BASED    # COUNT_BASED o TIME_BASED
        sliding-window-size: 10             # últimas N llamadas
        minimum-number-of-calls: 5          # mínimo de llamadas antes de evaluar

        # ESTADO OPEN
        wait-duration-in-open-state: 10s    # tiempo en OPEN antes de pasar a HALF-OPEN

        # ESTADO HALF-OPEN
        permitted-number-of-calls-in-half-open-state: 3  # llamadas de prueba

        # EXCEPCIONES
        record-exceptions:                  # estas excepciones cuentan como fallo
          - java.net.ConnectException
          - java.util.concurrent.TimeoutException
          - feign.FeignException
        ignore-exceptions:                  # estas NO cuentan como fallo
          - com.miempresa.exception.RecursoNoEncontradoException
```

### Uso con anotación @CircuitBreaker

```java
@Service
public class PedidosService {

    @Autowired
    private ProductosClient productosClient;

    @CircuitBreaker(name = "productos-service", fallbackMethod = "fallbackProducto")
    public Producto obtenerProducto(Long id) {
        return productosClient.obtenerProducto(id);
    }

    // Fallback — misma firma + el Throwable como último parámetro
    public Producto fallbackProducto(Long id, Throwable ex) {
        log.warn("Circuit Breaker activo para productos-service: {}", ex.getMessage());
        return Producto.builder()
            .id(id)
            .nombre("Producto no disponible")
            .disponible(false)
            .build();
    }
}
```

### Uso programático con CircuitBreakerFactory

```java
@Service
public class PedidosService {

    @Autowired
    private CircuitBreakerFactory circuitBreakerFactory;

    public Producto obtenerProducto(Long id) {
        CircuitBreaker cb = circuitBreakerFactory.create("productos-service");
        return cb.run(
            () -> productosClient.obtenerProducto(id),      // llamada principal
            throwable -> Producto.noDisponible(id)          // fallback
        );
    }
}
```

---

## 7.6 Fallback: respuestas de emergencia

El **fallback** es la respuesta que se devuelve cuando el circuito está abierto o la llamada falla. Un buen fallback:

- Devuelve datos en caché si están disponibles
- Devuelve una respuesta degradada pero funcional
- Lanza una excepción de negocio clara (no la excepción técnica de red)

### Tipos de fallback

```java
@Service
public class CatalogoService {

    // Fallback 1: Datos cacheados
    private Map<Long, Producto> cache = new ConcurrentHashMap<>();

    @CircuitBreaker(name = "productos-service", fallbackMethod = "fallbackDesdeCache")
    public Producto obtenerProducto(Long id) {
        Producto p = productosClient.obtenerProducto(id);
        cache.put(id, p);  // actualiza caché en cada llamada exitosa
        return p;
    }

    public Producto fallbackDesdeCache(Long id, Throwable ex) {
        Producto cached = cache.get(id);
        if (cached != null) {
            log.warn("Usando dato cacheado para producto {}", id);
            return cached;
        }
        throw new ServicioNoDisponibleException("Productos no disponible", ex);
    }

    // Fallback 2: Respuesta genérica
    @CircuitBreaker(name = "inventario-service", fallbackMethod = "stockDesconocido")
    public Integer obtenerStock(Long productoId) {
        return inventarioClient.getStock(productoId);
    }

    public Integer stockDesconocido(Long productoId, Throwable ex) {
        return -1;  // -1 = stock desconocido, la UI mostrará "bajo pedido"
    }
}
```

---

## 7.7 Retry, TimeLimiter y Bulkhead

### Retry — reintentos automáticos

```yaml
resilience4j:
  retry:
    instances:
      productos-service:
        max-attempts: 3
        wait-duration: 500ms
        retry-exceptions:
          - java.net.ConnectException
          - feign.RetryableException
        ignore-exceptions:
          - com.miempresa.exception.RecursoNoEncontradoException
```

```java
@Retry(name = "productos-service", fallbackMethod = "fallbackProducto")
@CircuitBreaker(name = "productos-service", fallbackMethod = "fallbackProducto")
public Producto obtenerProducto(Long id) {
    return productosClient.obtenerProducto(id);
}
// Orden: Retry → CircuitBreaker → llamada real
// Primero reintenta, si agota reintentos, el CB cuenta el fallo
```

### TimeLimiter — límite de tiempo

```yaml
resilience4j:
  timelimiter:
    instances:
      productos-service:
        timeout-duration: 3s          # máximo tiempo de espera
        cancel-running-future: true   # cancela el Future si se supera el timeout
```

```java
@TimeLimiter(name = "productos-service")
@CircuitBreaker(name = "productos-service")
public CompletableFuture<Producto> obtenerProductoAsync(Long id) {
    return CompletableFuture.supplyAsync(() -> productosClient.obtenerProducto(id));
}
// TimeLimiter requiere que el método devuelva CompletableFuture o Mono/Flux
```

### Bulkhead — aislamiento de recursos

El Bulkhead limita la concurrencia de llamadas a un servicio, evitando que un servicio lento agote el pool de hilos completo.

```yaml
resilience4j:
  bulkhead:
    instances:
      productos-service:
        max-concurrent-calls: 10      # máximo de llamadas simultáneas
        max-wait-duration: 100ms      # tiempo máximo esperando un slot libre
```

```java
@Bulkhead(name = "productos-service", type = Bulkhead.Type.SEMAPHORE)
public Producto obtenerProducto(Long id) {
    return productosClient.obtenerProducto(id);
}
// Si ya hay 10 llamadas en curso, la siguiente espera 100ms y luego lanza BulkheadFullException
```

---

## 7.8 Integración con Spring Cloud Gateway

El Circuit Breaker se puede aplicar directamente en el Gateway para rutas completas:

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: pedidos-route
          uri: lb://pedidos-service
          predicates:
            - Path=/api/pedidos/**
          filters:
            - StripPrefix=1
            - name: CircuitBreaker
              args:
                name: pedidosCircuitBreaker
                fallbackUri: forward:/fallback/pedidos
                # Si el servicio falla, enruta a /fallback/pedidos del propio Gateway
```

```java
@RestController
public class FallbackController {

    @GetMapping("/fallback/pedidos")
    public ResponseEntity<Map<String, String>> pedidosFallback() {
        return ResponseEntity
            .status(HttpStatus.SERVICE_UNAVAILABLE)
            .body(Map.of(
                "error", "Servicio de pedidos temporalmente no disponible",
                "mensaje", "Por favor, inténtelo de nuevo en unos minutos"
            ));
    }
}
```

---

## 7.9 Monitoreo con Actuator y métricas

Resilience4j expone el estado de los circuit breakers a través de Spring Actuator:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, metrics, circuitbreakers, circuitbreakerevents
  health:
    circuitbreakers:
      enabled: true
  endpoint:
    health:
      show-details: always
```

### Endpoints de monitoreo

```bash
# Estado de todos los circuit breakers
GET /actuator/health
# Respuesta incluye estado CLOSED/OPEN/HALF_OPEN de cada CB

# Métricas de un CB específico
GET /actuator/metrics/resilience4j.circuitbreaker.state?tag=name:productos-service

# Eventos recientes del CB (aperturas, cierres, llamadas fallidas)
GET /actuator/circuitbreakerevents/productos-service
```

### Métricas disponibles en Prometheus/Grafana

| Métrica | Descripción |
|---|---|
| `resilience4j_circuitbreaker_state` | Estado actual (0=CLOSED, 1=OPEN, 2=HALF_OPEN) |
| `resilience4j_circuitbreaker_calls_total` | Total de llamadas por resultado |
| `resilience4j_circuitbreaker_failure_rate` | Tasa de fallo actual |
| `resilience4j_circuitbreaker_slow_call_rate` | Tasa de llamadas lentas |

---

← [Parte 6 — API Gateway](./06-api-gateway.md) | [Volver al índice](./README.md) | Siguiente: [Parte 8 — Comunicación](./08-comunicacion.md) →
