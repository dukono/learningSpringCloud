# 8.7 Autorización por URL y métodos con Spring Security

← [8.6 Token Relay en Spring Cloud Gateway](sc-security-token-relay.md) | [Índice (README.md)](README.md) | [8.8 Propagación de identidad entre microservicios síncronos](sc-security-propagation.md) →

---

Spring Security 6.x proporciona dos capas de autorización que se complementan: la autorización por URL (`authorizeHttpRequests`) que actúa a nivel de filtro HTTP antes de que la petición llegue al controlador, y la autorización por método (`@PreAuthorize`, `@PostAuthorize`, `@PostFilter`) que actúa dentro de la capa de negocio. La autorización por URL es adecuada para reglas simples basadas en path y rol; la autorización por método permite expresiones SpEL con acceso a los argumentos del método y al valor de retorno. En microservicios, la práctica correcta es usar ambas capas: URL para una barrera rápida de acceso (evita que peticiones no autorizadas lleguen a la capa de negocio), y método para la lógica de autorización específica de negocio (acceso a recursos propios del usuario, filtrado por tenant, etc.).

> [PREREQUISITO] Requiere `spring-boot-starter-security` con `@EnableMethodSecurity` para la autorización a nivel de método. El Resource Server JWT del fichero 8.2 es prerequisito para que las authorities del token estén correctamente configuradas.

## Ejemplo central: autorización por URL y método con expresiones SpEL

### Dependencias Maven

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

### SecurityConfig con reglas URL y activación de method security

```java
package com.example.security;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableWebSecurity
@EnableMethodSecurity(prePostEnabled = true, securedEnabled = true)
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(jwt -> {}))
            .authorizeHttpRequests(auth -> auth
                // Endpoints públicos
                .requestMatchers("/actuator/health", "/actuator/info").permitAll()
                .requestMatchers("/api/public/**").permitAll()

                // Endpoints de admin: requiere rol ADMIN a nivel URL
                .requestMatchers("/api/admin/**").hasRole("ADMIN")

                // Endpoints de métricas: solo actuator interno
                .requestMatchers("/actuator/**").hasRole("MONITORING")

                // Todo lo demás: autenticado (la lógica fina va en @PreAuthorize)
                .anyRequest().authenticated()
            );

        return http.build();
    }
}
```

### Controlador con autorización a nivel de método

```java
package com.example.security;

import org.springframework.security.access.prepost.PostFilter;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/orders")
public class OrderController {

    private final OrderService orderService;

    public OrderController(OrderService orderService) {
        this.orderService = orderService;
    }

    // Requiere scope orders:read O rol ADMIN
    @GetMapping
    @PreAuthorize("hasAuthority('SCOPE_orders:read') or hasRole('ADMIN')")
    public List<Order> listOrders(@AuthenticationPrincipal Jwt jwt) {
        return orderService.getOrdersForUser(jwt.getSubject());
    }

    // Solo el propietario del pedido o un ADMIN puede verlo
    // @AuthenticationPrincipal se inyecta en la expresión SpEL
    @GetMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN') or @orderService.isOwner(#id, authentication.name)")
    public Order getOrder(@PathVariable String id) {
        return orderService.findById(id);
    }

    // Crear pedido: requiere scope orders:write
    @PostMapping
    @PreAuthorize("hasAuthority('SCOPE_orders:write')")
    public Order createOrder(@RequestBody OrderRequest request,
                              @AuthenticationPrincipal Jwt jwt) {
        return orderService.create(request, jwt.getSubject());
    }

    // PostFilter: filtra el resultado para que solo se devuelvan los pedidos propios
    // (alternativa a filtrar en la consulta de base de datos)
    @GetMapping("/all")
    @PreAuthorize("hasRole('ADMIN')")
    @PostFilter("hasRole('ADMIN') or filterObject.ownerId == authentication.name")
    public List<Order> getAllOrders() {
        return orderService.findAll();
    }
}
```

### Bean de servicio con lógica de autorización reutilizable

```java
package com.example.security;

import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.stereotype.Service;

import java.util.List;

@Service("orderService")  // El nombre del bean se usa en SpEL: @orderService.isOwner(...)
public class OrderService {

    private final OrderRepository orderRepository;

    public OrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    // Método auxiliar para autorización: llamado desde @PreAuthorize
    public boolean isOwner(String orderId, String username) {
        return orderRepository.findById(orderId)
            .map(order -> order.getOwnerId().equals(username))
            .orElse(false);
    }

    @PreAuthorize("hasAuthority('SCOPE_orders:read')")
    public List<Order> getOrdersForUser(String userId) {
        return orderRepository.findByOwnerId(userId);
    }

    public Order findById(String id) {
        return orderRepository.findById(id).orElseThrow();
    }

    @PreAuthorize("hasAuthority('SCOPE_orders:read')")
    public List<Order> findAll() {
        return orderRepository.findAll();
    }

    public Order create(OrderRequest request, String userId) {
        Order order = new Order(request.productId(), request.quantity(), userId);
        return orderRepository.save(order);
    }
}
```

> [CONCEPTO] `@PreAuthorize("@orderService.isOwner(#id, authentication.name)")` usa Spring Bean EL (`@beanName`) para llamar a un método del contexto de Spring dentro de la expresión de autorización. `#id` referencia el parámetro del método anotado, y `authentication.name` el `getName()` del principal actual. Esta combinación permite autorización dinámica sin acceso a base de datos en el filtro HTTP.

## Tabla de elementos clave

| Elemento | Descripción |
|---|---|
| `@EnableMethodSecurity` | Activa `@PreAuthorize`, `@PostAuthorize`, `@PostFilter`; reemplaza `@EnableGlobalMethodSecurity` (deprecated en Spring Security 6) |
| `@PreAuthorize("expr")` | Evalúa la expresión SpEL antes de ejecutar el método; lanza `AccessDeniedException` si es false |
| `@PostAuthorize("expr")` | Evalúa la expresión después del método; acceso al resultado con `returnObject`; útil para validar propiedad del recurso devuelto |
| `@PostFilter("expr")` | Filtra una colección devuelta; `filterObject` referencia cada elemento |
| `hasRole('ADMIN')` | Equivale a `hasAuthority('ROLE_ADMIN')`; el prefijo se añade automáticamente |
| `hasAuthority('SCOPE_orders:read')` | Verifica la authority exacta, incluyendo prefijo `SCOPE_` |
| `#paramName` en SpEL | Referencia al parámetro del método anotado por nombre |
| `authentication.name` | `getName()` del principal; con JWT configurado como `preferred_username`, el username del usuario |
| `@beanName.method(...)` en SpEL | Llama a un método de un bean de Spring desde la expresión de autorización |

## Buenas y malas prácticas

**Hacer:**
- Combinar autorización por URL (barrera rápida) y por método (lógica fina): la capa URL rechaza peticiones obvias sin coste de procesamiento; la capa método maneja la lógica de propietario o tenant.
- Centralizar la lógica de autorización de negocio en métodos de servicio anotados con `@PreAuthorize` en lugar de en los controladores; los servicios son reutilizables desde eventos, schedulers y tests.
- Usar `@PostAuthorize("returnObject.ownerId == authentication.name")` para verificar la propiedad del recurso devuelto sin hacer una consulta extra antes del método; recupera el recurso una sola vez y valida en la respuesta.

**Evitar:**
- Implementar lógica de autorización dentro de la lógica de negocio del método (ej: `if (!user.isAdmin()) throw new AccessDeniedException()`); mezcla responsabilidades y hace los tests de negocio dependientes del contexto de seguridad.
- Usar `@PostFilter` en colecciones grandes: filtra elemento por elemento en memoria después de cargar todos los registros de la base de datos; usar predicados en la consulta JPA/JPQL para filtrar en la capa de datos.
- Confiar en la autorización por URL como única capa de seguridad: un cambio de rutas o un error de configuración del `requestMatcher` puede exponer endpoints; la defensa en profundidad con `@PreAuthorize` garantiza que el método está protegido independientemente del path.

---

← [8.6 Token Relay en Spring Cloud Gateway](sc-security-token-relay.md) | [Índice (README.md)](README.md) | [8.8 Propagación de identidad entre microservicios síncronos](sc-security-propagation.md) →
