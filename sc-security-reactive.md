# 8.12 Seguridad en microservicios reactivos con WebFlux

← [8.11 Spring Authorization Server — personalización avanzada y OIDC](sc-security-authorization-server-advanced.md) | [Índice (README.md)](README.md) | [8.13 CORS, CSRF y cabeceras de seguridad HTTP](sc-security-cors-csrf.md) →

---

La seguridad en aplicaciones WebFlux sigue los mismos principios que en aplicaciones servlet, pero con una implementación completamente diferente: en lugar de `HttpSecurity` y `SecurityFilterChain`, se usa `ServerHttpSecurity` y `SecurityWebFilterChain`; en lugar de `SecurityContextHolder` con `ThreadLocal`, se usa `ReactiveSecurityContextHolder` que propaga el contexto a través del `ReactorContext`. Esta diferencia es fundamental: en WebFlux no existe hilo de request que persista durante toda la llamada — el SecurityContext debe viajar en el `ReactorContext` del publisher. El 95% de la configuración es análoga al mundo servlet, pero las clases son distintas y no intercambiables.

> [PREREQUISITO] Requiere `spring-boot-starter-oauth2-resource-server` con `spring-boot-starter-webflux`. La dependencia de `spring-boot-starter-web` (servlet) no puede coexistir con `webflux` en el mismo proyecto.

## Equivalencias servlet / WebFlux en Spring Security

| Servlet | WebFlux | Descripción |
|---|---|---|
| `HttpSecurity` | `ServerHttpSecurity` | Builder de la cadena de seguridad |
| `SecurityFilterChain` | `SecurityWebFilterChain` | Cadena de filtros resultante |
| `@EnableWebSecurity` | `@EnableWebFluxSecurity` | Anotación de activación |
| `SecurityContextHolder` | `ReactiveSecurityContextHolder` | Acceso al contexto de seguridad |
| `Authentication` en `ThreadLocal` | `Authentication` en `ReactorContext` | Almacenamiento del principal |
| `UserDetailsService` | `ReactiveUserDetailsService` | Carga de usuarios |
| `AuthenticationManager` | `ReactiveAuthenticationManager` | Validación de credenciales |

## Ejemplo central: Resource Server JWT reactivo con autorización

### Dependencias Maven

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>
```

### SecurityConfig WebFlux

```java
package com.example.reactive;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.method.configuration.EnableReactiveMethodSecurity;
import org.springframework.security.config.annotation.web.reactive.EnableWebFluxSecurity;
import org.springframework.security.config.web.server.ServerHttpSecurity;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.security.oauth2.server.resource.authentication.ReactiveJwtAuthenticationConverter;
import org.springframework.security.web.server.SecurityWebFilterChain;

import reactor.core.publisher.Flux;

import java.util.List;

@Configuration
@EnableWebFluxSecurity
@EnableReactiveMethodSecurity  // habilita @PreAuthorize en WebFlux
public class ReactiveSecurityConfig {

    @Bean
    public SecurityWebFilterChain securityWebFilterChain(ServerHttpSecurity http) {
        return http
            .csrf(ServerHttpSecurity.CsrfSpec::disable)
            // Resource Server JWT reactivo
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt
                    .jwtAuthenticationConverter(reactiveJwtConverter())
                )
            )
            .authorizeExchange(exchange -> exchange
                .pathMatchers("/actuator/health").permitAll()
                .pathMatchers("/api/admin/**").hasRole("ADMIN")
                .anyExchange().authenticated()
            )
            .build();
    }

    @Bean
    public ReactiveJwtAuthenticationConverter reactiveJwtConverter() {
        ReactiveJwtAuthenticationConverter converter = new ReactiveJwtAuthenticationConverter();

        // Converter que devuelve Flux<GrantedAuthority> (reactivo)
        converter.setJwtGrantedAuthoritiesConverter(jwt -> {
            List<String> roles = jwt.getClaimAsStringList("roles");
            if (roles == null) return Flux.empty();
            return Flux.fromIterable(roles)
                .map(role -> new SimpleGrantedAuthority(
                    role.startsWith("ROLE_") ? role : "ROLE_" + role));
        });

        converter.setPrincipalClaimName("preferred_username");
        return converter;
    }
}
```

### Controlador reactivo con acceso al principal

```java
package com.example.reactive;

import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.web.bind.annotation.*;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

@RestController
@RequestMapping("/api/orders")
public class ReactiveOrderController {

    private final ReactiveOrderRepository orderRepository;

    public ReactiveOrderController(ReactiveOrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    @GetMapping
    @PreAuthorize("hasAuthority('SCOPE_orders:read')")
    public Flux<Order> listOrders(@AuthenticationPrincipal Jwt jwt) {
        // El JWT está disponible directamente en el método
        String userId = jwt.getSubject();
        return orderRepository.findByOwnerId(userId);
    }

    @GetMapping("/{id}")
    public Mono<Order> getOrder(@PathVariable String id,
                                 @AuthenticationPrincipal Mono<Jwt> jwtMono) {
        // Alternativamente, el principal puede inyectarse como Mono
        return jwtMono.flatMap(jwt ->
            orderRepository.findById(id)
                .filter(order -> order.getOwnerId().equals(jwt.getSubject()))
        );
    }
}
```

### Acceso programático al SecurityContext reactivo

```java
package com.example.reactive;

import org.springframework.security.core.context.ReactiveSecurityContextHolder;
import org.springframework.security.core.context.SecurityContext;
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationToken;
import org.springframework.stereotype.Service;
import reactor.core.publisher.Mono;

@Service
public class ReactiveAuditService {

    // Acceder al SecurityContext desde lógica de servicio reactiva
    public Mono<String> getCurrentUserId() {
        return ReactiveSecurityContextHolder.getContext()
            .map(SecurityContext::getAuthentication)
            .filter(auth -> auth instanceof JwtAuthenticationToken)
            .cast(JwtAuthenticationToken.class)
            .map(jwtAuth -> jwtAuth.getToken().getSubject())
            .defaultIfEmpty("anonymous");
    }

    public Mono<Void> auditAction(String action) {
        return getCurrentUserId()
            .doOnNext(userId ->
                System.out.println("Auditing: " + action + " by " + userId))
            .then();
    }
}
```

> [CONCEPTO] `ReactiveSecurityContextHolder.getContext()` retorna `Mono<SecurityContext>`. El contexto se propaga automáticamente a través del `ReactorContext` cuando el contexto de seguridad se establece en el filtro de entrada del request. Si se hace un `subscribeOn(Schedulers.boundedElastic())` u otro switch de scheduler, el contexto se propaga correctamente — a diferencia del `ThreadLocal` servlet que se pierde al cambiar de thread.

> [ADVERTENCIA] `@EnableReactiveMethodSecurity` es la anotación correcta para WebFlux; `@EnableMethodSecurity` (para servlet) no tiene efecto en aplicaciones WebFlux. Si se usa la anotación incorrecta, los `@PreAuthorize` son ignorados silenciosamente y todos los métodos son accesibles.

## Tabla de elementos clave

| Componente | Descripción |
|---|---|
| `@EnableWebFluxSecurity` | Anotación que activa la integración de Spring Security con WebFlux |
| `@EnableReactiveMethodSecurity` | Activa `@PreAuthorize` en métodos de componentes reactivos |
| `ServerHttpSecurity` | Builder reactivo equivalente a `HttpSecurity`; no son intercambiables |
| `SecurityWebFilterChain` | Cadena de filtros reactiva; equivalente a `SecurityFilterChain` servlet |
| `ReactiveJwtAuthenticationConverter` | Convierte el `Jwt` en `Authentication` en contexto reactivo; converter devuelve `Flux<GrantedAuthority>` |
| `ReactiveSecurityContextHolder.getContext()` | Acceso programático al `Mono<SecurityContext>` desde cadenas reactivas |
| `@AuthenticationPrincipal Mono<Jwt>` | Inyección del principal como `Mono`; equivalente reactivo al `@AuthenticationPrincipal Jwt` servlet |

## Buenas y malas prácticas

**Hacer:**
- Usar `@AuthenticationPrincipal Jwt jwt` (sin `Mono`) en métodos de controlador WebFlux cuando sea posible; Spring Security lo resuelve síncronamente del contexto reactivo y es más limpio que manejar `Mono<Jwt>`.
- Usar `ReactiveSecurityContextHolder` en lugar de `SecurityContextHolder` en cualquier servicio WebFlux; el `SecurityContextHolder` ThreadLocal siempre devuelve null en contextos reactivos.
- Configurar `@EnableReactiveMethodSecurity` y verificar que no está el equivalente servlet `@EnableMethodSecurity` en el mismo módulo; la presencia de ambas produce comportamiento indefinido.

**Evitar:**
- Mezclar `spring-boot-starter-web` y `spring-boot-starter-webflux` en el mismo módulo si el objetivo es WebFlux; Spring Boot elige el modo basándose en las dependencias y la presencia de ambas activa el modo servlet con compatibilidad limitada para WebFlux.
- Usar `block()` en cadenas reactivas dentro de un contexto de seguridad WebFlux; bloquear en un thread de Reactor Netty puede causar deadlocks cuando el SecurityContext necesita propagarse a través del `ReactorContext`.
- Guardar la autenticación del request en una variable estática o de instancia para reutilizarla en requests posteriores; en WebFlux, el `ReactorContext` es específico del pipeline del request — el contexto del request A nunca debe ser accesible desde el request B.

---

← [8.11 Spring Authorization Server — personalización avanzada y OIDC](sc-security-authorization-server-advanced.md) | [Índice (README.md)](README.md) | [8.13 CORS, CSRF y cabeceras de seguridad HTTP](sc-security-cors-csrf.md) →
