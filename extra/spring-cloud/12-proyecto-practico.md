# Parte 12 — Proyecto Práctico Completo

← [Parte 11 — Kubernetes](./11-kubernetes.md) | [Volver al índice](./README.md) | Siguiente: [Parte 13 — Referencia Rápida](./13-referencia-rapida.md) →

---

## 12.1 Diseño del sistema de ejemplo

Construiremos un sistema de e-commerce simplificado con los siguientes microservicios:

```
┌─────────────────────────────────────────────────────────────┐
│                      INFRAESTRUCTURA                         │
│  Config Server (:8888)   Eureka Server (:8761)              │
│  Zipkin (:9411)          (opcional) RabbitMQ (:5672)        │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                       SERVICIOS                              │
│                                                             │
│  API Gateway (:8080)                                        │
│  ├── /api/productos/**  → productos-service (:8081)         │
│  └── /api/pedidos/**   → pedidos-service (:8082)            │
│                               ↓ (llama vía Feign + CB)      │
│                          productos-service                   │
└─────────────────────────────────────────────────────────────┘
```

### Estructura de módulos Maven

```
spring-cloud-ecommerce/
├── pom.xml                    (parent POM con BOM)
├── config-server/
├── eureka-server/
├── api-gateway/
├── productos-service/
└── pedidos-service/
```

### Parent POM

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.miempresa</groupId>
    <artifactId>spring-cloud-ecommerce</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>pom</packaging>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.5</version>
    </parent>

    <properties>
        <java.version>21</java.version>
        <spring-cloud.version>2023.0.1</spring-cloud.version>
    </properties>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <modules>
        <module>config-server</module>
        <module>eureka-server</module>
        <module>api-gateway</module>
        <module>productos-service</module>
        <module>pedidos-service</module>
    </modules>
</project>
```

---

## 12.2 Config Server con repositorio Git

### pom.xml del config-server

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-config-server</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
</dependencies>
```

### ConfigServerApplication.java

```java
@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}
```

### application.yml del config-server

```yaml
server:
  port: 8888

spring:
  application:
    name: config-server
  cloud:
    config:
      server:
        git:
          uri: https://github.com/mi-org/config-ecommerce
          default-label: main
          clone-on-start: true

management:
  endpoints:
    web:
      exposure:
        include: health, info, busrefresh
```

### Estructura del repositorio de configuración

```
config-ecommerce/  (repo Git separado)
├── application.yml              ← config global
├── productos-service.yml
├── productos-service-prod.yml
├── pedidos-service.yml
└── api-gateway.yml
```

```yaml
# application.yml (global)
spring:
  datasource:
    driver-class-name: org.postgresql.Driver

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/

management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics, prometheus

# pedidos-service.yml
pedidos:
  max-por-pagina: 20
  tiempo-espera-pago-minutos: 30

spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/pedidos
    username: pedidos_user
    password: '{cipher}AQBxxx...'   # cifrado
```

---

## 12.3 Eureka Server

### pom.xml del eureka-server

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-config</artifactId>
    </dependency>
</dependencies>
```

### EurekaServerApplication.java

```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

### application.yml del eureka-server

```yaml
server:
  port: 8761

spring:
  application:
    name: eureka-server

eureka:
  instance:
    hostname: localhost
  client:
    register-with-eureka: false
    fetch-registry: false
```

---

## 12.4 Microservicio de productos

### pom.xml del productos-service

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-config</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-tracing-bridge-brave</artifactId>
    </dependency>
    <dependency>
        <groupId>io.zipkin.reporter2</groupId>
        <artifactId>zipkin-reporter-brave</artifactId>
    </dependency>
</dependencies>
```

### application.yml del productos-service

```yaml
spring:
  application:
    name: productos-service
  config:
    import: "configserver:http://localhost:8888"
  profiles:
    active: ${SPRING_PROFILES_ACTIVE:dev}

server:
  port: 8081

management:
  tracing:
    sampling:
      probability: 1.0
  zipkin:
    tracing:
      endpoint: http://localhost:9411/api/v2/spans
```

### Entidad y Repositorio

```java
@Entity
@Table(name = "productos")
public class Producto {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String nombre;
    private String descripcion;
    private BigDecimal precio;
    private Integer stock;
    private boolean activo;
    // getters, setters, constructores
}

public interface ProductoRepository extends JpaRepository<Producto, Long> {
    List<Producto> findByActivoTrue();
}
```

### Controller REST

```java
@RestController
@RequestMapping("/api/productos")
public class ProductoController {

    @Autowired
    private ProductoRepository productoRepository;

    @GetMapping
    public List<Producto> listar() {
        return productoRepository.findByActivoTrue();
    }

    @GetMapping("/{id}")
    public ResponseEntity<Producto> obtener(@PathVariable Long id) {
        return productoRepository.findById(id)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public Producto crear(@RequestBody @Valid Producto producto) {
        return productoRepository.save(producto);
    }
}
```

---

## 12.5 Microservicio de pedidos (con Feign + Circuit Breaker)

### pom.xml del pedidos-service

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-config</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-openfeign</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
    </dependency>
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-tracing-bridge-brave</artifactId>
    </dependency>
    <dependency>
        <groupId>io.zipkin.reporter2</groupId>
        <artifactId>zipkin-reporter-brave</artifactId>
    </dependency>
</dependencies>
```

### Cliente Feign para productos-service

```java
@FeignClient(
    name = "productos-service",
    fallbackFactory = ProductosFallbackFactory.class
)
public interface ProductosClient {
    @GetMapping("/api/productos/{id}")
    Producto obtenerProducto(@PathVariable("id") Long id);
}

@Component
public class ProductosFallbackFactory implements FallbackFactory<ProductosClient> {
    @Override
    public ProductosClient create(Throwable cause) {
        return id -> {
            log.warn("Fallback para producto {}: {}", id, cause.getMessage());
            return new Producto(id, "Producto no disponible", BigDecimal.ZERO, false);
        };
    }
}
```

### Servicio de pedidos con Circuit Breaker

```java
@Service
public class PedidoService {

    @Autowired
    private PedidoRepository pedidoRepository;
    @Autowired
    private ProductosClient productosClient;

    public PedidoDetalleDTO crearPedido(PedidoRequest request) {
        // Obtener producto (con circuit breaker vía Feign)
        Producto producto = productosClient.obtenerProducto(request.getProductoId());

        if (!producto.isActivo()) {
            throw new ProductoNoDisponibleException(request.getProductoId());
        }

        Pedido pedido = new Pedido();
        pedido.setProductoId(request.getProductoId());
        pedido.setCantidad(request.getCantidad());
        pedido.setTotal(producto.getPrecio().multiply(new BigDecimal(request.getCantidad())));
        pedido.setEstado(EstadoPedido.PENDIENTE);
        pedido = pedidoRepository.save(pedido);

        return new PedidoDetalleDTO(pedido, producto);
    }
}
```

### application.yml del pedidos-service

```yaml
spring:
  application:
    name: pedidos-service
  config:
    import: "configserver:http://localhost:8888"

server:
  port: 8082

spring.cloud.openfeign.circuitbreaker.enabled: true

resilience4j:
  circuitbreaker:
    instances:
      productos-service:
        failure-rate-threshold: 50
        wait-duration-in-open-state: 10s
        sliding-window-size: 10
        minimum-number-of-calls: 5
```

---

## 12.6 API Gateway con rutas y filtros

### pom.xml del api-gateway

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-gateway</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-config</artifactId>
    </dependency>
</dependencies>
<!-- IMPORTANTE: NO incluir spring-boot-starter-web aquí -->
```

### application.yml del api-gateway

```yaml
spring:
  application:
    name: api-gateway
  config:
    import: "configserver:http://localhost:8888"
  cloud:
    gateway:
      routes:
        - id: productos-route
          uri: lb://productos-service
          predicates:
            - Path=/api/productos/**
          filters:
            - AddRequestHeader=X-Gateway-Source, spring-cloud-gateway

        - id: pedidos-route
          uri: lb://pedidos-service
          predicates:
            - Path=/api/pedidos/**
          filters:
            - name: CircuitBreaker
              args:
                name: pedidosCircuitBreaker
                fallbackUri: forward:/fallback/pedidos

      default-filters:
        - DedupeResponseHeader=Access-Control-Allow-Credentials Access-Control-Allow-Origin
      globalcors:
        cors-configurations:
          '[/**]':
            allowedOrigins: "*"
            allowedMethods: "*"
            allowedHeaders: "*"

server:
  port: 8080
```

### FallbackController en el Gateway

```java
@RestController
public class FallbackController {

    @GetMapping("/fallback/pedidos")
    public ResponseEntity<Map<String, String>> pedidosFallback() {
        return ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE)
            .body(Map.of(
                "error", "SERVICE_UNAVAILABLE",
                "mensaje", "El servicio de pedidos está temporalmente no disponible. Inténtelo de nuevo."
            ));
    }
}
```

---

## 12.7 Trazabilidad con Zipkin

### Todos los servicios incluyen:

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-brave</artifactId>
</dependency>
<dependency>
    <groupId>io.zipkin.reporter2</groupId>
    <artifactId>zipkin-reporter-brave</artifactId>
</dependency>
```

```yaml
# En todos los application.yml
management:
  tracing:
    sampling:
      probability: 1.0
  zipkin:
    tracing:
      endpoint: http://localhost:9411/api/v2/spans

logging:
  pattern:
    level: "%5p [${spring.application.name:},%X{traceId:-},%X{spanId:-}]"
```

Con esto, una petición a `GET /api/pedidos/1` produce trazas visibles en `http://localhost:9411` mostrando todos los spans a través de Gateway → Pedidos → Productos.

---

## 12.8 Docker Compose del ecosistema completo

```yaml
# docker-compose.yml
version: '3.8'

services:

  # ─── BASES DE DATOS ────────────────────────────────────────
  postgres-productos:
    image: postgres:16
    environment:
      POSTGRES_DB: productos
      POSTGRES_USER: productos_user
      POSTGRES_PASSWORD: productos_pass
    ports:
      - "5432:5432"

  postgres-pedidos:
    image: postgres:16
    environment:
      POSTGRES_DB: pedidos
      POSTGRES_USER: pedidos_user
      POSTGRES_PASSWORD: pedidos_pass
    ports:
      - "5433:5432"

  # ─── INFRAESTRUCTURA SPRING CLOUD ──────────────────────────
  config-server:
    image: miempresa/config-server:1.0.0
    ports:
      - "8888:8888"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8888/actuator/health"]
      interval: 10s
      retries: 5

  eureka-server:
    image: miempresa/eureka-server:1.0.0
    ports:
      - "8761:8761"
    depends_on:
      config-server:
        condition: service_healthy

  zipkin:
    image: openzipkin/zipkin
    ports:
      - "9411:9411"

  # ─── SERVICIOS DE NEGOCIO ──────────────────────────────────
  productos-service:
    image: miempresa/productos-service:1.0.0
    environment:
      SPRING_PROFILES_ACTIVE: dev
      SPRING_CONFIG_IMPORT: "configserver:http://config-server:8888"
    ports:
      - "8081:8081"
    depends_on:
      - config-server
      - eureka-server
      - postgres-productos

  pedidos-service:
    image: miempresa/pedidos-service:1.0.0
    environment:
      SPRING_PROFILES_ACTIVE: dev
      SPRING_CONFIG_IMPORT: "configserver:http://config-server:8888"
    ports:
      - "8082:8082"
    depends_on:
      - config-server
      - eureka-server
      - postgres-pedidos

  api-gateway:
    image: miempresa/api-gateway:1.0.0
    environment:
      SPRING_CONFIG_IMPORT: "configserver:http://config-server:8888"
    ports:
      - "8080:8080"
    depends_on:
      - config-server
      - eureka-server
      - productos-service
      - pedidos-service
```

### Orden de arranque

```
1. postgres-productos, postgres-pedidos, zipkin  (sin dependencias)
2. config-server  (lee del repo Git)
3. eureka-server  (lee config del config-server)
4. productos-service, pedidos-service  (leen config, se registran en eureka)
5. api-gateway  (descubre servicios en eureka)
```

### Probar el sistema completo

```bash
# Arrancar todo
docker-compose up -d

# Crear un producto
curl -X POST http://localhost:8080/api/productos \
  -H "Content-Type: application/json" \
  -d '{"nombre":"Laptop","precio":999.99,"stock":50,"activo":true}'

# Crear un pedido (gateway → pedidos → productos vía Feign)
curl -X POST http://localhost:8080/api/pedidos \
  -H "Content-Type: application/json" \
  -d '{"productoId":1,"cantidad":2}'

# Ver trazas en Zipkin
open http://localhost:9411

# Ver servicios en Eureka
open http://localhost:8761
```

---

← [Parte 11 — Kubernetes](./11-kubernetes.md) | [Volver al índice](./README.md) | Siguiente: [Parte 13 — Referencia Rápida](./13-referencia-rapida.md) →
