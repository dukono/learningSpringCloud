# 8.15 Testing / Verificación de Spring Cloud Security

← [8.14 Configuración avanzada de seguridad y producción](sc-security-production.md) | [Índice (README.md)](README.md) | [9.1 Setup y starters de Spring Cloud Kubernetes](sc-kubernetes-setup.md) →

---

Testear la seguridad en Spring Boot 4.x/Spring Security 6.x requiere tres herramientas específicas: `@WithMockUser` para simular un usuario autenticado sin JWT real, `@WithSecurityContext` para construir un contexto de seguridad personalizado con claims JWT específicas, y `SecurityMockMvcRequestPostProcessors.jwt()` para enviar peticiones con un token JWT mockeado directamente en el request. El error más común es usar `@WithMockUser` en un Resource Server JWT y asumir que simula correctamente el contexto — `@WithMockUser` usa `UsernamePasswordAuthenticationToken`, no `JwtAuthenticationToken`, y si el controlador usa `@AuthenticationPrincipal Jwt jwt`, el objeto JWT será null aunque la petición no dé 401.

> [PREREQUISITO] Requiere `spring-boot-starter-test`, `spring-security-test` (incluido automáticamente con el test starter cuando Spring Security está presente) y `@SpringBootTest` con `MockMvc`.

## Estrategias de testing de seguridad

| Estrategia | Simula | Cuándo usar |
|---|---|---|
| `@WithMockUser` | Usuario básico con roles | Pruebas de autorización por URL o rol, sin necesidad de claims JWT |
| `.with(jwt())` (MockMvc postprocessor) | JWT completo con claims personalizables | Controladores que usan `@AuthenticationPrincipal Jwt jwt` |
| `@WithSecurityContext` | Cualquier Authentication personalizada | Flujos especiales: multi-tenancy, JwtAuthenticationToken con claims Keycloak |
| `MockMvc` sin auth | Sin autenticación | Verificar que endpoints devuelven 401 sin token |

## Ejemplo central: testing de Resource Server con JWT mockeado

### Dependencias Maven

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
<!-- spring-security-test se incluye automáticamente con spring-boot-starter-test
     cuando Spring Security está en el classpath -->
```

### Test con `.with(jwt())` — simular JWT con claims específicas

```java
package com.example.security;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors;
import org.springframework.test.web.servlet.MockMvc;

import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.jwt;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest
@AutoConfigureMockMvc
class OrderControllerSecurityTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    void whenNoToken_thenReturns401() throws Exception {
        mockMvc.perform(get("/api/orders"))
            .andExpect(status().isUnauthorized());
    }

    @Test
    void whenValidJwtWithScope_thenReturns200() throws Exception {
        mockMvc.perform(get("/api/orders")
            // Simular JWT con sub y scope; no requiere conexión al Authorization Server
            .with(jwt()
                .jwt(builder -> builder
                    .subject("user-123")
                    .claim("preferred_username", "alice@example.com")
                    .claim("scope", "orders:read")
                    .claim("roles", java.util.List.of("USER"))
                )
            ))
            .andExpect(status().isOk());
    }

    @Test
    void whenJwtWithoutRequiredScope_thenReturns403() throws Exception {
        mockMvc.perform(post("/api/orders")
            .contentType("application/json")
            .content("{\"productId\": \"p1\", \"quantity\": 1}")
            .with(jwt()
                .jwt(builder -> builder
                    .subject("user-123")
                    .claim("scope", "orders:read")  // solo read, no write
                )
            ))
            .andExpect(status().isForbidden());
    }

    @Test
    void whenAdminRole_thenCanDeleteOrder() throws Exception {
        mockMvc.perform(delete("/api/orders/order-1")
            .with(jwt()
                .jwt(builder -> builder
                    .subject("admin-456")
                    .claim("roles", java.util.List.of("ADMIN"))
                )
                // Configurar authorities explícitamente para @PreAuthorize hasRole("ADMIN")
                .authorities(new org.springframework.security.core.authority.SimpleGrantedAuthority("ROLE_ADMIN"))
            ))
            .andExpect(status().isNoContent());
    }

    @Test
    void whenAuthenticationPrincipalJwtInjected_thenSubjectIsAccessible() throws Exception {
        // Verificar que @AuthenticationPrincipal Jwt jwt funciona correctamente
        mockMvc.perform(get("/api/orders/owner-info")
            .with(jwt()
                .jwt(builder -> builder
                    .subject("user-789")
                    .claim("preferred_username", "bob@example.com")
                    .claim("scope", "orders:read")
                )
            ))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.userId").value("user-789"))
            .andExpect(jsonPath("$.username").value("bob@example.com"));
    }
}
```

### Test reactivo WebFlux con `mockJwt()`

```java
package com.example.security;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.reactive.AutoConfigureWebTestClient;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.web.reactive.server.WebTestClient;

import static org.springframework.security.test.web.reactive.server.SecurityMockServerConfigurers.mockJwt;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureWebTestClient
class ReactiveOrderControllerSecurityTest {

    @Autowired
    private WebTestClient webTestClient;

    @Test
    void whenValidJwtWebFlux_thenReturns200() {
        webTestClient
            .mutateWith(mockJwt()
                .jwt(jwt -> jwt
                    .subject("user-123")
                    .claim("scope", "orders:read")
                    .claim("roles", java.util.List.of("USER"))
                )
            )
            .get()
            .uri("/api/orders")
            .exchange()
            .expectStatus().isOk();
    }

    @Test
    void whenNoAuth_thenReturns401() {
        webTestClient
            .get()
            .uri("/api/orders")
            .exchange()
            .expectStatus().isUnauthorized();
    }
}
```

### Custom SecurityContext con @WithSecurityContext para claims Keycloak

```java
package com.example.security;

import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.context.SecurityContext;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationToken;
import org.springframework.security.test.context.support.WithSecurityContextFactory;

import java.lang.annotation.*;
import java.time.Instant;
import java.util.List;
import java.util.Map;

// Anotación personalizada para simular un usuario Keycloak con realm_access.roles
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@org.springframework.security.test.context.support.WithSecurityContext(
    factory = WithKeycloakUser.Factory.class)
@interface WithKeycloakUser {
    String username() default "alice";
    String[] roles() default {"USER"};
    String tenantId() default "default";

    class Factory implements WithSecurityContextFactory<WithKeycloakUser> {
        @Override
        public SecurityContext createSecurityContext(WithKeycloakUser annotation) {
            Jwt jwt = Jwt.withTokenValue("test-token")
                .header("alg", "RS256")
                .subject(annotation.username())
                .issuer("https://auth.example.com")
                .issuedAt(Instant.now())
                .expiresAt(Instant.now().plusSeconds(300))
                .claim("preferred_username", annotation.username())
                .claim("tenant_id", annotation.tenantId())
                .claim("realm_access", Map.of("roles", List.of(annotation.roles())))
                .build();

            var authorities = java.util.Arrays.stream(annotation.roles())
                .map(role -> new SimpleGrantedAuthority("ROLE_" + role))
                .toList();

            var authentication = new JwtAuthenticationToken(jwt, authorities);

            SecurityContext context = SecurityContextHolder.createEmptyContext();
            context.setAuthentication(authentication);
            return context;
        }
    }
}
```

```java
// Uso de la anotación personalizada en tests
@Test
@WithKeycloakUser(username = "alice", roles = {"USER", "ADMIN"}, tenantId = "tenant-a")
void whenKeycloakAdminUser_thenCanAccessAdminEndpoint() throws Exception {
    mockMvc.perform(get("/api/admin/users"))
        .andExpect(status().isOk());
}
```

> [ADVERTENCIA] `.with(jwt())` en MockMvc no valida la firma del JWT ni consulta el JWK Set: es una simulación que bypasea la validación criptográfica. Esto es correcto en tests unitarios/integración — el propósito es testear la lógica de autorización, no la infraestructura de JWT. Para testear la validación de firma real, usar Testcontainers con un Authorization Server real.

## Tabla de elementos clave

| Componente | Descripción |
|---|---|
| `.with(jwt())` | MockMvc post-processor para simular JWT; de `SecurityMockMvcRequestPostProcessors` |
| `.jwt(builder -> ...)` en `jwt()` | Configura claims del JWT simulado: `subject()`, `claim()`, `issuer()` |
| `.authorities(...)` en `jwt()` | Configura las `GrantedAuthority` del contexto; necesario cuando `@PreAuthorize` usa `hasRole()` |
| `mockJwt()` | Equivalente reactivo de `.with(jwt())`; de `SecurityMockServerConfigurers` |
| `@WithMockUser` | Simula `UsernamePasswordAuthenticationToken`; NO es `JwtAuthenticationToken` |
| `@WithSecurityContext` | Permite crear cualquier `Authentication` personalizada para tests |
| `WithSecurityContextFactory` | Implementar para construir el `SecurityContext` de una anotación custom |

## Buenas y malas prácticas

**Hacer:**
- Usar `.with(jwt().authorities(new SimpleGrantedAuthority("ROLE_ADMIN")))` cuando el test verifica `hasRole()`: sin configurar `authorities`, el JWT simulado puede no tener la authority correctamente mapeada según la configuración del `JwtAuthenticationConverter`.
- Escribir tests negativos explícitos (sin autenticación → 401, sin scope correcto → 403): estos tests detectan regresiones cuando la configuración de seguridad cambia inadvertidamente.
- Crear una anotación `@WithKeycloakUser` (u otras custom) para los tests de proyectos con estructura de claims no estándar; evita repetir la construcción del JWT en cada test.

**Evitar:**
- Usar `@WithMockUser` en tests de controladores que usan `@AuthenticationPrincipal Jwt jwt`; el objeto JWT será null, el test puede fallar con `NullPointerException` en el controlador o con un error diferente al esperado.
- Desactivar Spring Security completamente en tests (`@WebMvcTest` sin `spring.security.enabled=false` real); los tests de seguridad deben ejecutarse con la configuración de seguridad real del proyecto.
- Testear solo los casos de acceso exitoso e ignorar los casos de denegación; los bugs de seguridad más críticos son los que dejan acceder a quien no debería, no los que niegan acceso a quien sí debería tenerlo.

---

← [8.14 Configuración avanzada de seguridad y producción](sc-security-production.md) | [Índice (README.md)](README.md) | [9.1 Setup y starters de Spring Cloud Kubernetes](sc-kubernetes-setup.md) →
