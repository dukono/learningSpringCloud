# Parte 5 — Load Balancing (Balanceo de carga)

← [Parte 4 — Service Discovery](./04-service-discovery.md) | [Volver al índice](./README.md) | Siguiente: [Parte 6 — API Gateway](./06-api-gateway.md) →

---

## 5.1 Qué es el balanceo de carga en microservicios

Cuando un servicio escala horizontalmente tiene **múltiples instancias** ejecutándose en paralelo:

```
[pedidos-service] quiere llamar a [productos-service]

Eureka le informa que hay 3 instancias:
  - 10.0.0.10:8082  (instancia 1)
  - 10.0.0.11:8082  (instancia 2)
  - 10.0.0.12:8082  (instancia 3)

¿A cuál llama? → lo decide el Load Balancer
```

El **balanceo de carga** distribuye el tráfico entre las instancias disponibles para:
- Maximizar el uso de recursos (no sobrecargar una sola instancia)
- Aumentar la disponibilidad (si una cae, las demás siguen atendiendo)
- Reducir la latencia de respuesta

### Tipos de balanceo de carga

| Tipo | Dónde vive | Ejemplo |
|---|---|---|
| **Server-side** | Componente externo al servicio | AWS ALB, NGINX, HAProxy |
| **Client-side** | Dentro del propio servicio llamante | Spring Cloud LoadBalancer |

Spring Cloud usa **client-side load balancing**: el propio microservicio decide a qué instancia llamar consultando el registro de Eureka.

---

## 5.2 Spring Cloud LoadBalancer (sucesor de Ribbon)

### Historia: Ribbon → Spring Cloud LoadBalancer

| Componente | Estado | Versión |
|---|---|---|
| **Netflix Ribbon** | Deprecado / en mantenimiento desde 2018 | Spring Cloud Hoxton y anteriores |
| **Spring Cloud LoadBalancer** | Activo, reemplaza a Ribbon | Spring Cloud 2020.x en adelante |

**Spring Cloud LoadBalancer** es la implementación oficial y activa. Es más ligero que Ribbon y está diseñado para integrarse con el ecosistema Spring Boot 3 / WebFlux.

### Dependencia

Con la dependencia de Eureka Client ya incluye Spring Cloud LoadBalancer automáticamente. De forma explícita:

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-loadbalancer</artifactId>
</dependency>
```

> Si se tiene Ribbon en el classpath junto con Spring Cloud LoadBalancer, Ribbon tiene prioridad por compatibilidad. Para forzar el uso de Spring Cloud LoadBalancer con Ribbon presente: `spring.cloud.loadbalancer.ribbon.enabled=false`

### Cómo funciona internamente

```
1. @FeignClient("productos-service") o @LoadBalanced RestTemplate
         ↓
2. Spring Cloud LoadBalancer intercepta la llamada
         ↓
3. Consulta ServiceInstanceListSupplier
   → que consulta Eureka (u otro discovery)
   → obtiene la lista de instancias activas
         ↓
4. Aplica el algoritmo de selección (Round Robin por defecto)
         ↓
5. Sustituye "productos-service" por la IP:puerto real elegida
         ↓
6. Realiza la llamada HTTP
```

---

## 5.3 Algoritmos: Round Robin y Random

Spring Cloud LoadBalancer incluye dos algoritmos de selección listos para usar:

### Round Robin (por defecto)

Recorre las instancias en orden circular. Si hay 3 instancias:

```
Petición 1 → instancia A
Petición 2 → instancia B
Petición 3 → instancia C
Petición 4 → instancia A  (vuelve al inicio)
Petición 5 → instancia B
...
```

Garantiza una distribución uniforme de la carga. Es el algoritmo recomendado en la mayoría de casos.

Clase: `RoundRobinLoadBalancer`

### Random

Elige una instancia aleatoriamente. Útil en escenarios donde se quiere evitar la sincronización de ciclos de petición.

Clase: `RandomLoadBalancer`

---

## 5.4 Configuración y personalización del balanceador

### Cambiar el algoritmo a Random

```java
// Clase de configuración — NO debe ser @Component ni estar en el paquete raíz
// para evitar que Spring la recoja automáticamente para todos los servicios
public class RandomLBConfig {

    @Bean
    ReactorLoadBalancer<ServiceInstance> randomLoadBalancer(
            Environment environment,
            LoadBalancerClientFactory loadBalancerClientFactory) {

        String name = environment.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
        return new RandomLoadBalancer(
            loadBalancerClientFactory.getLazyProvider(name, ServiceInstanceListSupplier.class),
            name
        );
    }
}
```

```java
// Aplicar la configuración SOLO para el servicio "productos-service"
@FeignClient(name = "productos-service", configuration = RandomLBConfig.class)
public interface ProductosClient {
    @GetMapping("/productos/{id}")
    Producto obtenerProducto(@PathVariable Long id);
}
```

### Configuración global via properties

```yaml
spring:
  cloud:
    loadbalancer:
      # Algoritmo por defecto para todos los servicios
      configurations: random   # o round-robin (default)

      # Cache de instancias (reduce llamadas a Eureka)
      cache:
        enabled: true
        ttl: 35s          # tiempo de vida del caché
        capacity: 256     # número máximo de entradas en caché

      # Health check — excluir instancias no saludables
      health-check:
        initial-delay: 0
        interval: 25s
```

### Implementar un algoritmo personalizado

```java
public class WeightedLoadBalancer implements ReactorServiceInstanceLoadBalancer {

    private final ObjectProvider<ServiceInstanceListSupplier> supplierProvider;
    private final String serviceId;

    public WeightedLoadBalancer(
            ObjectProvider<ServiceInstanceListSupplier> supplierProvider,
            String serviceId) {
        this.supplierProvider = supplierProvider;
        this.serviceId = serviceId;
    }

    @Override
    public Mono<Response<ServiceInstance>> choose(Request request) {
        ServiceInstanceListSupplier supplier = supplierProvider.getIfAvailable();
        return supplier.get(request)
            .next()
            .map(instances -> {
                // lógica personalizada de selección
                ServiceInstance chosen = selectByWeight(instances);
                return new DefaultResponse(chosen);
            });
    }

    private ServiceInstance selectByWeight(List<ServiceInstance> instances) {
        // implementar lógica de peso por metadatos de instancia
        return instances.stream()
            .max(Comparator.comparingInt(i ->
                Integer.parseInt(i.getMetadata().getOrDefault("weight", "1"))))
            .orElse(instances.get(0));
    }
}
```

---

## 5.5 Integración con Eureka y OpenFeign

### Con @LoadBalanced RestTemplate

```java
@Configuration
public class RestTemplateConfig {

    @Bean
    @LoadBalanced   // ← esta anotación activa el load balancing
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

```java
@Service
public class ProductosService {

    @Autowired
    private RestTemplate restTemplate;

    public Producto obtenerProducto(Long id) {
        // Usar el nombre lógico del servicio — Spring lo resuelve con Eureka + LoadBalancer
        return restTemplate.getForObject(
            "http://productos-service/productos/" + id,
            Producto.class
        );
    }
}
```

### Con OpenFeign (recomendado)

OpenFeign integra automáticamente el LoadBalancer cuando está en el classpath:

```java
@FeignClient(name = "productos-service")
public interface ProductosClient {

    @GetMapping("/productos/{id}")
    Producto obtenerProducto(@PathVariable Long id);

    @GetMapping("/productos")
    List<Producto> listarProductos();
}
```

```java
@Service
@RequiredArgsConstructor
public class PedidosService {

    private final ProductosClient productosClient;

    public DetallePedido obtenerDetalle(Long pedidoId) {
        Pedido pedido = pedidoRepository.findById(pedidoId).orElseThrow();
        // Feign + LoadBalancer + Eureka: resuelve automáticamente la instancia
        Producto producto = productosClient.obtenerProducto(pedido.getProductoId());
        return new DetallePedido(pedido, producto);
    }
}
```

### Con WebClient (reactivo)

```java
@Configuration
public class WebClientConfig {

    @Bean
    @LoadBalanced
    public WebClient.Builder webClientBuilder() {
        return WebClient.builder();
    }
}
```

```java
@Service
@RequiredArgsConstructor
public class ProductosReactiveService {

    private final WebClient.Builder webClientBuilder;

    public Mono<Producto> obtenerProducto(Long id) {
        return webClientBuilder.build()
            .get()
            .uri("http://productos-service/productos/{id}", id)
            .retrieve()
            .bodyToMono(Producto.class);
    }
}
```

### Resumen: cuándo usar cada opción

| Opción | Estilo | Recomendado para |
|---|---|---|
| `@FeignClient` | Declarativo | La mayoría de casos — más limpio |
| `@LoadBalanced RestTemplate` | Imperativo | Apps con Spring MVC sin Feign |
| `@LoadBalanced WebClient` | Reactivo | Apps con Spring WebFlux |

---

← [Parte 4 — Service Discovery](./04-service-discovery.md) | [Volver al índice](./README.md) | Siguiente: [Parte 6 — API Gateway](./06-api-gateway.md) →
