# 13.6 API Composition con Spring Cloud Gateway y OpenFeign

← [13.5 Strangler Fig con Spring Cloud Gateway](sc-patrones-strangler.md) | [Índice (README.md)](README.md) | [13.7 Sidecar Pattern con Spring Cloud Netflix Sidecar](sc-patrones-sidecar.md) →

---

API Composition es el patrón en el que un servicio agrega datos de múltiples microservicios para servir una respuesta completa al cliente, evitando que el cliente tenga que hacer N llamadas secuenciales. En Spring Cloud, se implementa típicamente en dos lugares: en el API Gateway (para composición simple basada en routing y transformación) o en un servicio de Backend-for-Frontend (BFF) que usa OpenFeign o WebClient para llamar a los microservicios en paralelo y componer la respuesta. El Gateway es adecuado para agregaciones estáticas configurables; el BFF es necesario cuando la lógica de composición es compleja, condicional o require transformaciones significativas de los datos.

> [PREREQUISITO] Requiere `spring-cloud-starter-gateway` para composición en Gateway o `spring-cloud-starter-openfeign` para composición en BFF. Para llamadas en paralelo: WebFlux o `CompletableFuture` con `@Async`.

## Ejemplo central: API Composition en BFF con OpenFeign en paralelo

### Dependencias Maven del BFF

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>2025.1.1</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-openfeign</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <!-- Resiliencia en las llamadas a microservicios downstream -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
    </dependency>
</dependencies>
```

### Clientes Feign para los microservicios downstream

```java
package com.example.bff;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;

@FeignClient(name = "order-service", fallbackFactory = OrderClientFallback.class)
public interface OrderServiceClient {

    @GetMapping("/orders/{orderId}")
    OrderDto getOrder(@PathVariable String orderId);

    @GetMapping("/orders/user/{userId}")
    java.util.List<OrderDto> getUserOrders(@PathVariable String userId);
}

@FeignClient(name = "product-service", fallbackFactory = ProductClientFallback.class)
public interface ProductServiceClient {

    @GetMapping("/products/{productId}")
    ProductDto getProduct(@PathVariable String productId);
}

@FeignClient(name = "user-service")
public interface UserServiceClient {

    @GetMapping("/users/{userId}")
    UserDto getUser(@PathVariable String userId);
}
```

### BFF Service — composición en paralelo con CompletableFuture

```java
package com.example.bff;

import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.concurrent.CompletableFuture;
import java.util.stream.Collectors;

@Service
public class OrderDashboardCompositionService {

    private final OrderServiceClient orderClient;
    private final ProductServiceClient productClient;
    private final UserServiceClient userClient;

    public OrderDashboardCompositionService(OrderServiceClient orderClient,
                                             ProductServiceClient productClient,
                                             UserServiceClient userClient) {
        this.orderClient = orderClient;
        this.productClient = productClient;
        this.userClient = userClient;
    }

    // Composición de vista completa del dashboard de pedidos
    // Las 3 llamadas se ejecutan en paralelo
    public OrderDashboardView getOrderDashboard(String orderId) {
        // Lanzar las 3 llamadas en paralelo
        CompletableFuture<OrderDto> orderFuture =
            CompletableFuture.supplyAsync(() -> orderClient.getOrder(orderId));

        // El productId se necesita del pedido → esperar a orderFuture primero
        OrderDto order = orderFuture.join();  // bloquear solo aquí

        // Una vez tenemos el pedido, lanzar las otras dos llamadas en paralelo
        CompletableFuture<ProductDto> productFuture =
            CompletableFuture.supplyAsync(() ->
                productClient.getProduct(order.getProductId()));

        CompletableFuture<UserDto> userFuture =
            CompletableFuture.supplyAsync(() ->
                userClient.getUser(order.getUserId()));

        // Esperar a que ambas terminen
        ProductDto product = productFuture.join();
        UserDto user = userFuture.join();

        return new OrderDashboardView(order, product, user);
    }

    // Composición de pedidos del usuario: orden + productos en paralelo
    public UserOrdersView getUserOrdersDashboard(String userId) {
        List<OrderDto> orders = orderClient.getUserOrders(userId);

        // Obtener los productos de todos los pedidos en paralelo
        List<CompletableFuture<ProductDto>> productFutures = orders.stream()
            .map(order -> CompletableFuture.supplyAsync(() ->
                productClient.getProduct(order.getProductId())))
            .collect(Collectors.toList());

        // Combinar pedidos con sus productos
        List<OrderWithProduct> enrichedOrders = new java.util.ArrayList<>();
        for (int i = 0; i < orders.size(); i++) {
            enrichedOrders.add(new OrderWithProduct(orders.get(i), productFutures.get(i).join()));
        }

        return new UserOrdersView(userId, enrichedOrders);
    }
}
```

### Controlador del BFF

```java
package com.example.bff;

import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/bff/orders")
public class OrderBffController {

    private final OrderDashboardCompositionService compositionService;

    public OrderBffController(OrderDashboardCompositionService compositionService) {
        this.compositionService = compositionService;
    }

    // El cliente hace UNA llamada → el BFF hace N llamadas a los microservicios
    @GetMapping("/{orderId}/dashboard")
    public OrderDashboardView getOrderDashboard(@PathVariable String orderId) {
        return compositionService.getOrderDashboard(orderId);
    }

    @GetMapping("/my-orders")
    public UserOrdersView getMyOrders(@AuthenticationPrincipal Jwt jwt) {
        return compositionService.getUserOrdersDashboard(jwt.getSubject());
    }
}
```

### Fallback factories para resiliencia en la composición

```java
package com.example.bff;

import org.springframework.cloud.openfeign.FallbackFactory;
import org.springframework.stereotype.Component;

@Component
public class ProductClientFallback implements FallbackFactory<ProductServiceClient> {

    @Override
    public ProductServiceClient create(Throwable cause) {
        return productId -> {
            // Si el product-service no responde, devolver datos parciales
            // para que la composición devuelva una respuesta degradada
            ProductDto partial = new ProductDto();
            partial.setId(productId);
            partial.setName("Producto no disponible");
            partial.setAvailable(false);
            return partial;
        };
    }
}
```

> [CONCEPTO] La composición en paralelo con `CompletableFuture` reduce el tiempo de respuesta del BFF de la suma de los tiempos de cada llamada a el máximo de los tiempos de las llamadas paralelas. Si order-service tarda 50ms y product-service tarda 80ms (en paralelo), el BFF responde en ~80ms en lugar de ~130ms. El número óptimo de llamadas paralelas depende del límite del executor de threads y de la capacidad de los servicios downstream.

## Tabla de elementos clave

| Componente | Descripción |
|---|---|
| BFF (Backend-for-Frontend) | Servicio que agrega APIs de múltiples microservicios tailored para un cliente específico (mobile, web, etc.) |
| `CompletableFuture.supplyAsync()` | Ejecución asíncrona de una llamada Feign; varias instancias en paralelo reducen la latencia total |
| `.join()` | Espera el resultado del `CompletableFuture`; bloquea hasta que la llamada completa |
| `FallbackFactory` | Feign fallback que recibe la excepción original; permite incluir el motivo del fallo en la respuesta degradada |
| Composición en Gateway | Alternativa para agregaciones simples; usar `AggregateResponseGatewayFilterFactory` para combinar respuestas de múltiples rutas |
| Respuesta degradada | Respuesta parcial cuando un microservicio downstream no responde; mejora la experiencia de usuario respecto a un error completo |

## Buenas y malas prácticas

**Hacer:**
- Usar `FallbackFactory` en lugar de `@FeignClient(fallback=...)` cuando la respuesta degradada necesita contextualizar el error; la factory recibe la excepción que causó el fallo.
- Configurar timeouts independientes en cada cliente Feign del BFF; si el product-service es lento, no debería bloquear la respuesta del BFF más allá de su propio timeout.
- Cachear en el BFF los datos que cambian poco (datos de producto, catálogo): evita N llamadas a product-service por cada llamada al BFF para una lista de pedidos del usuario.

**Evitar:**
- Hacer las llamadas a los microservicios secuenciales cuando son independientes; si order-service, product-service y user-service no tienen dependencias entre sí para una composición, ejecutarlas en paralelo.
- Ignorar el tamaño de la respuesta del BFF; la composición de N microservicios puede producir respuestas muy grandes — aplicar proyecciones (solo los campos necesarios para el cliente) en el BFF.
- Usar el API Gateway para composición cuando la lógica requiere código Java (transformaciones condicionales, ordenación, filtrado); el Gateway está optimizado para routing, no para transformaciones de datos complejas — el BFF es la herramienta correcta.

---

← [13.5 Strangler Fig con Spring Cloud Gateway](sc-patrones-strangler.md) | [Índice (README.md)](README.md) | [13.7 Sidecar Pattern con Spring Cloud Netflix Sidecar](sc-patrones-sidecar.md) →
