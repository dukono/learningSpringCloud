# 8.2 Resource Server JWT — configuración y personalización

← [8.1 Fundamentos OAuth2 y JWT para microservicios](sc-security-oauth2-fundamentals.md) | [Índice (README.md)](README.md) | [8.3 Resource Server avanzado — claims, opaque tokens y multi-tenancy](sc-security-resource-server-advanced.md) →

---

La configuración por defecto de un Resource Server JWT en Spring Security 6.x valida la firma y la expiración del token, pero no extrae roles ni scopes de forma que sean directamente utilizables con `@PreAuthorize`. Para que las claims del JWT se conviertan en `GrantedAuthority` que Spring Security reconozca, es necesario personalizar el `JwtAuthenticationConverter`. Esta personalización es el paso más frecuente que los equipos omiten y que produce el síntoma de endpoints que devuelven 403 aunque el token es válido: el token tiene el rol `ROLE_ADMIN` en la claim `roles`, pero Spring Security no lo encuentra porque está buscando en `scope` o con un path de claim diferente.

> [PREREQUISITO] Requiere `spring-boot-starter-oauth2-resource-server`. Haber comprendido el modelo JWT del fichero 8.1.

## Flujo de conversión de JWT a Authentication

Spring Security convierte el JWT en un `JwtAuthenticationToken` mediante un pipeline de conversión personalizable.

```
JWT (string)
  → NimbusJwtDecoder (verifica firma, expiración, issuer)
  → JwtAuthenticationConverter
      → JwtGrantedAuthoritiesConverter (claims → GrantedAuthority)
  → JwtAuthenticationToken (Authentication en SecurityContext)
```

## Ejemplo central: Resource Server con extracción de roles personalizada

### Dependencias Maven

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
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>
```

### application.yml

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth.example.com
          # Opcional: audiencias que este Resource Server acepta
          # audiences: order-service
```

### SecurityConfig con JwtAuthenticationConverter personalizado

```java
package com.example.security;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.convert.converter.Converter;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationConverter;
import org.springframework.security.oauth2.server.resource.authentication.JwtGrantedAuthoritiesConverter;
import org.springframework.security.web.SecurityFilterChain;

import java.util.Collection;
import java.util.List;
import java.util.stream.Collectors;
import java.util.stream.Stream;

@Configuration
@EnableWebSecurity
@EnableMethodSecurity  // habilita @PreAuthorize, @PostAuthorize
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt
                    .jwtAuthenticationConverter(jwtAuthenticationConverter())
                )
            )
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health", "/actuator/info").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            );

        return http.build();
    }

    @Bean
    public JwtAuthenticationConverter jwtAuthenticationConverter() {
        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
        // Configurar cómo extraer authorities del JWT
        converter.setJwtGrantedAuthoritiesConverter(customAuthoritiesConverter());
        // Opcional: qué claim del JWT usar como "name" del principal
        converter.setPrincipalClaimName("preferred_username");
        return converter;
    }

    // Extrae authorities de TANTO "scope" (OAuth2 estándar) COMO "roles" (Keycloak/custom)
    private Converter<Jwt, Collection<GrantedAuthority>> customAuthoritiesConverter() {
        // Converter para el claim "scope" → SCOPE_xxx
        JwtGrantedAuthoritiesConverter scopeConverter = new JwtGrantedAuthoritiesConverter();
        scopeConverter.setAuthorityPrefix("SCOPE_");

        // Converter personalizado para el claim "roles" → ROLE_xxx
        return jwt -> {
            // Authorities de "scope"
            Collection<GrantedAuthority> scopeAuthorities =
                scopeConverter.convert(jwt);

            // Authorities de "roles" (lista de strings en el JWT)
            List<String> roles = jwt.getClaimAsStringList("roles");
            Collection<GrantedAuthority> roleAuthorities = roles == null
                ? List.of()
                : roles.stream()
                    .map(role -> new SimpleGrantedAuthority(
                        role.startsWith("ROLE_") ? role : "ROLE_" + role))
                    .collect(Collectors.toList());

            return Stream.concat(
                scopeAuthorities != null ? scopeAuthorities.stream() : Stream.empty(),
                roleAuthorities.stream()
            ).collect(Collectors.toList());
        };
    }
}
```

### Controlador usando authorities extraídas

```java
package com.example.security;

import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/orders")
public class OrderController {

    @GetMapping
    @PreAuthorize("hasAuthority('SCOPE_orders:read') or hasRole('ADMIN')")
    public List<String> listOrders(@AuthenticationPrincipal Jwt jwt) {
        // jwt.getSubject() → "user-123"
        // jwt.getClaimAsString("preferred_username") → "alice@example.com"
        return List.of("order-1", "order-2");
    }

    @PostMapping
    @PreAuthorize("hasAuthority('SCOPE_orders:write')")
    public String createOrder() {
        return "created";
    }

    @DeleteMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN')")
    public void deleteOrder(@PathVariable String id) {
        // Solo ROLE_ADMIN puede eliminar
    }
}
```

> [CONCEPTO] `@AuthenticationPrincipal Jwt jwt` inyecta el objeto JWT decodificado directamente en el método del controlador. Todas las claims del token están accesibles mediante `jwt.getClaim("claim-name")`. Es el mecanismo estándar para acceder a la identidad del usuario dentro de la lógica de negocio.

## Tabla de elementos clave

| Componente | Descripción |
|---|---|
| `JwtAuthenticationConverter` | Convierte el `Jwt` en `JwtAuthenticationToken`; personalizar para extraer roles de claims no estándar |
| `JwtGrantedAuthoritiesConverter` | Extrae authorities del claim `scope` con prefijo `SCOPE_`; configurable con `setAuthoritiesClaimName()` |
| `setPrincipalClaimName("preferred_username")` | Define qué claim del JWT se usa como `getName()` del principal; default: `sub` |
| `@EnableMethodSecurity` | Habilita `@PreAuthorize`, `@PostAuthorize`, `@Secured`; necesario para seguridad a nivel de método |
| `@AuthenticationPrincipal Jwt jwt` | Inyecta el JWT decodificado en métodos de controller; acceso directo a todas las claims |
| `hasAuthority('SCOPE_orders:read')` | Verifica una authority exacta incluyendo el prefijo; para scopes OAuth2 |
| `hasRole('ADMIN')` | Verifica `ROLE_ADMIN` (añade el prefijo automáticamente); para roles de negocio |
| `issuer-uri` | Spring Security descarga el JWK Set y valida que `iss` del JWT coincida con esta URI |

## Buenas y malas prácticas

**Hacer:**
- Centralizar el `JwtAuthenticationConverter` en un `@Bean` reutilizable; si varios microservicios tienen la misma estructura de claims, extraer la configuración a una librería compartida.
- Configurar `setPrincipalClaimName("preferred_username")` o el claim que identifica al usuario de negocio; el default (`sub`) es un ID opaco que no tiene significado para el logging de negocio.
- Validar `aud` (audience) en la configuración: `jwt.audiences(aud -> aud.add("order-service"))` en el DSL de Resource Server; previene que tokens emitidos para otros servicios sean aceptados.

**Evitar:**
- Hardcodear nombres de claims en varios lugares del código; definir constantes para los nombres de claims del JWT (`"roles"`, `"preferred_username"`, etc.) en una clase de configuración.
- Usar `hasRole()` y `hasAuthority()` de forma intercambiable sin entender el prefijo: `hasRole("ADMIN")` es equivalente a `hasAuthority("ROLE_ADMIN")`; mezclar sin consistencia produce bugs de autorización difíciles de diagnosticar.
- Omitir `@EnableMethodSecurity` y usar solo `requestMatchers` en `SecurityFilterChain`: la seguridad a nivel de URL no protege métodos de servicio llamados desde otros puntos (tareas programadas, eventos, etc.).

---

← [8.1 Fundamentos OAuth2 y JWT para microservicios](sc-security-oauth2-fundamentals.md) | [Índice (README.md)](README.md) | [8.3 Resource Server avanzado — claims, opaque tokens y multi-tenancy](sc-security-resource-server-advanced.md) →
