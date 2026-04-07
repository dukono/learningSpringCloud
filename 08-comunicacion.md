# Parte 8 — Comunicación entre servicios

← [Parte 7 — Circuit Breaker](./07-circuit-breaker.md) | [Volver al índice](./README.md) | Siguiente: [Parte 9 — Observabilidad](./09-observabilidad.md) →

---

## 8.1 Comunicación síncrona vs asíncrona

### Comunicación síncrona (HTTP/REST, gRPC)

El servicio que llama **espera** la respuesta antes de continuar:

```
Servicio Pedidos → [llama y espera] → Servicio Productos
                 ←─ respuesta ───────
```

**Cuándo usarla:**
- El resultado es necesario para continuar el procesamiento
- Se necesita confirmación inmediata
- Operaciones de lectura (consultas)

**Herramientas:** OpenFeign, RestTemplate, WebClient

### Comunicación asíncrona (Mensajería)

El servicio que llama **no espera**; publica un mensaje y continúa:

```
Servicio Pagos → [publica evento "PagoConfirmado"] → Broker (Kafka)
                                                          ↓
                                      Servicio Notificaciones [consume y procesa]
                                      Servicio Facturación    [consume y procesa]
```

**Cuándo usarla:**
- No se necesita respuesta inmediata
- Múltiples consumidores del mismo evento
- Operaciones de escritura que pueden procesarse después
- Desacoplamiento temporal entre servicios

**Herramientas:** Spring Cloud Stream, Spring Cloud Bus (Kafka, RabbitMQ)

| Aspecto | Síncrona | Asíncrona |
|---|---|---|
| **Acoplamiento** | Alto (el cliente espera) | Bajo (fire and forget) |
| **Complejidad** | Baja | Mayor (broker, serialización) |
| **Latencia** | Baja si el servicio responde rápido | Variable (depende del broker) |
| **Disponibilidad** | El servicio destino debe estar UP | El broker puede guardar mensajes |
| **Casos de uso** | Consultas, datos en tiempo real | Eventos, notificaciones, workflows |

---

## 8.2 Spring Cloud OpenFeign — cliente HTTP declarativo

**OpenFeign** permite crear clientes HTTP como **interfaces Java**, sin escribir código de llamadas HTTP:

```java
// Sin Feign — código manual con RestTemplate
String url = "http://productos-service/api/productos/" + id;
Producto p = restTemplate.getForObject(url, Producto.class);

// Con Feign — solo una interfaz anotada
@FeignClient(name = "productos-service")
public interface ProductosClient {
    @GetMapping("/api/productos/{id}")
    Producto obtenerProducto(@PathVariable Long id);
}
// Feign genera automáticamente la implementación HTTP
```

### Dependencia Maven

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

### Habilitar Feign en la aplicación

```java
@SpringBootApplication
@EnableFeignClients   // ← escanea @FeignClient en el classpath
public class PedidosApplication {
    public static void main(String[] args) {
        SpringApplication.run(PedidosApplication.class, args);
    }
}
```

### Definir un cliente Feign completo

```java
@FeignClient(
    name = "productos-service",             // service ID en Eureka
    path = "/api/productos",                // path base (opcional)
    fallbackFactory = ProductosFallbackFactory.class  // fallback de circuit breaker
)
public interface ProductosClient {

    @GetMapping("/{id}")
    Producto obtenerProducto(@PathVariable("id") Long id);

    @GetMapping
    List<Producto> listarProductos(
        @RequestParam(value = "pagina", defaultValue = "0") int pagina,
        @RequestParam(value = "tamanio", defaultValue = "10") int tamanio
    );

    @PostMapping
    Producto crearProducto(@RequestBody ProductoRequest request);

    @PutMapping("/{id}")
    Producto actualizarProducto(@PathVariable("id") Long id,
                                 @RequestBody ProductoRequest request);

    @DeleteMapping("/{id}")
    void eliminarProducto(@PathVariable("id") Long id);
}
```

### FallbackFactory (fallback con acceso al error)

```java
@Component
public class ProductosFallbackFactory implements FallbackFactory<ProductosClient> {

    @Override
    public ProductosClient create(Throwable cause) {
        return new ProductosClient() {
            @Override
            public Producto obtenerProducto(Long id) {
                log.error("Fallback obtenerProducto: {}", cause.getMessage());
                return Producto.noDisponible(id);
            }

            @Override
            public List<Producto> listarProductos(int pagina, int tamanio) {
                log.error("Fallback listarProductos: {}", cause.getMessage());
                return Collections.emptyList();
            }
            // ... resto de métodos
        };
    }
}
```

---

## 8.3 Configuración de Feign: timeouts, interceptors, logging

### Timeouts

```yaml
spring:
  cloud:
    openfeign:
      client:
        config:
          # Configuración global (aplica a todos los clientes)
          default:
            connect-timeout: 5000    # ms para establecer conexión
            read-timeout: 10000      # ms esperando respuesta

          # Configuración específica para un cliente (sobreescribe la global)
          productos-service:
            connect-timeout: 2000
            read-timeout: 5000
            logger-level: FULL       # NONE, BASIC, HEADERS, FULL
```

### Logging de peticiones Feign

```yaml
logging:
  level:
    com.miempresa.clientes.ProductosClient: DEBUG
```

| Nivel | Qué registra |
|---|---|
| `NONE` | Nada (defecto) |
| `BASIC` | Método, URL, código de respuesta, tiempo |
| `HEADERS` | Lo anterior + headers de petición y respuesta |
| `FULL` | Lo anterior + body completo de petición y respuesta |

### Request Interceptor — añadir headers a todas las peticiones

```java
@Component
public class FeignAuthInterceptor implements RequestInterceptor {

    @Override
    public void apply(RequestTemplate template) {
        // Propaga el token JWT de la petición entrante a las llamadas salientes
        ServletRequestAttributes attrs =
            (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();

        if (attrs != null) {
            String authHeader = attrs.getRequest().getHeader("Authorization");
            if (authHeader != null) {
                template.header("Authorization", authHeader);
            }
        }
    }
}
```

### Error Decoder personalizado

```java
@Component
public class CustomFeignErrorDecoder implements ErrorDecoder {

    @Override
    public Exception decode(String methodKey, Response response) {
        return switch (response.status()) {
            case 404 -> new RecursoNoEncontradoException("Recurso no encontrado");
            case 403 -> new AccesoDenegadoException("Sin permisos");
            case 503 -> new ServicioNoDisponibleException("Servicio no disponible");
            default -> new ErrorDecoder.Default().decode(methodKey, response);
        };
    }
}
```

---

## 8.4 Feign + LoadBalancer + Circuit Breaker (integración completa)

Esta es la combinación más usada en producción:

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
</dependency>
<!-- Eureka client ya incluye LoadBalancer -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

```yaml
spring:
  cloud:
    openfeign:
      circuitbreaker:
        enabled: true                    # activa integración Feign + CB
      client:
        config:
          productos-service:
            read-timeout: 5000

resilience4j:
  circuitbreaker:
    instances:
      productos-service:
        failure-rate-threshold: 50
        wait-duration-in-open-state: 10s
```

```java
// La integración es transparente: solo define el cliente y el fallback
@FeignClient(name = "productos-service", fallbackFactory = ProductosFallbackFactory.class)
public interface ProductosClient {
    @GetMapping("/api/productos/{id}")
    Producto obtenerProducto(@PathVariable Long id);
}

// Spring Cloud gestiona automáticamente:
// 1. Feign serializa la petición HTTP
// 2. LoadBalancer elige instancia de Eureka
// 3. Circuit Breaker monitoriza éxitos/fallos
// 4. Si CB está OPEN → llama al fallback directamente
```

---

## 8.5 Spring Cloud Stream — mensajería con Kafka y RabbitMQ

**Spring Cloud Stream** es una capa de abstracción sobre brokers de mensajes que permite escribir código de mensajería **independiente del broker** (Kafka, RabbitMQ, etc.).

### Conceptos fundamentales

| Concepto | Descripción |
|---|---|
| **Binder** | Adaptador al broker específico (Kafka, RabbitMQ) |
| **Binding** | Canal de entrada (input) o salida (output) asociado a un topic/queue |
| **Producer** | Produce mensajes al binding de salida |
| **Consumer** | Consume mensajes del binding de entrada |
| **Consumer Group** | Grupo de consumidores que comparten la carga (como en Kafka) |

### Dependencias Maven

```xml
<!-- Para Kafka -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-stream-kafka</artifactId>
</dependency>

<!-- Para RabbitMQ -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-stream-rabbit</artifactId>
</dependency>
```

---

## 8.6 Binders, Bindings y canales en Spring Cloud Stream

### Productor de mensajes

```java
@Service
public class PagoService {

    @Autowired
    private StreamBridge streamBridge;

    public void procesarPago(Pago pago) {
        // lógica de negocio...
        pago.setEstado(EstadoPago.CONFIRMADO);
        pagoRepository.save(pago);

        // Publicar evento al topic "pago-confirmado"
        PagoConfirmadoEvent evento = new PagoConfirmadoEvent(
            pago.getId(), pago.getUsuarioId(), pago.getImporte()
        );
        streamBridge.send("pago-confirmado-out-0", evento);
        // "pago-confirmado-out-0" = nombre del binding de salida
    }
}
```

### Consumidor de mensajes

```java
@Service
public class NotificacionService {

    @Bean
    public Consumer<PagoConfirmadoEvent> procesarPagoConfirmado() {
        return evento -> {
            log.info("Pago confirmado recibido: {}", evento.getPagoId());
            // Enviar email, push notification, etc.
            emailService.enviarConfirmacion(evento.getUsuarioId(), evento.getImporte());
        };
    }
}
```

### Función (Producer + Consumer en uno)

```java
@Bean
public Function<PedidoCreado, InventarioReservado> procesarPedido() {
    return pedido -> {
        // consume un PedidoCreado, produce un InventarioReservado
        inventario.reservar(pedido.getProductoId(), pedido.getCantidad());
        return new InventarioReservado(pedido.getId(), pedido.getProductoId());
    };
}
```

### Configuración en application.yml

```yaml
spring:
  cloud:
    stream:
      # Configuración del binder Kafka
      kafka:
        binder:
          brokers: localhost:9092

      # Configuración de bindings
      bindings:
        # Binding de salida (productor)
        pago-confirmado-out-0:
          destination: pago-confirmado    # nombre del topic en Kafka
          content-type: application/json

        # Binding de entrada (consumidor)
        procesarPagoConfirmado-in-0:
          destination: pago-confirmado
          group: notificaciones-group     # consumer group (Kafka)
          content-type: application/json

        # Función (in y out)
        procesarPedido-in-0:
          destination: pedido-creado
        procesarPedido-out-0:
          destination: inventario-reservado

      function:
        definition: procesarPagoConfirmado;procesarPedido  # lista las funciones activas
```

---

## 8.7 Spring Cloud Bus — propagación de eventos

**Spring Cloud Bus** conecta todos los microservicios mediante un broker de mensajes para **propagar eventos de configuración** a todos a la vez.

### Caso de uso principal: refresco masivo de configuración

```
Cambio en repositorio Git
       ↓
POST /actuator/busrefresh (a cualquier nodo o al Config Server)
       ↓
Spring Cloud Bus publica RefreshRemoteApplicationEvent en Kafka/RabbitMQ
       ↓
TODOS los microservicios reciben el evento y refrescan sus @RefreshScope beans
```

### Dependencia

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-kafka</artifactId>
    <!-- o spring-cloud-starter-bus-amqp para RabbitMQ -->
</dependency>
```

### Configuración

```yaml
spring:
  cloud:
    bus:
      enabled: true
      id: ${spring.application.name}:${server.port}:${random.uuid}

management:
  endpoints:
    web:
      exposure:
        include: busrefresh, busenv
```

### Endpoints de Spring Cloud Bus

```bash
# Refresca la configuración en TODOS los servicios
POST /actuator/busrefresh

# Refresca solo un servicio específico
POST /actuator/busrefresh/pedidos-service:**

# Actualiza una propiedad en entorno de todos los servicios
POST /actuator/busenv
Body: {"name": "feature.nueva-ui", "value": "true"}
```

---

← [Parte 7 — Circuit Breaker](./07-circuit-breaker.md) | [Volver al índice](./README.md) | Siguiente: [Parte 9 — Observabilidad](./09-observabilidad.md) →
