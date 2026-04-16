# 8.3 Resource Server avanzado — claims, opaque tokens y multi-tenancy

← [8.2 Resource Server JWT — configuración y personalización](sc-security-resource-server-jwt.md) | [Índice (README.md)](README.md) | [8.4 OAuth2 Client servlet — registro, gestión de tokens y Client Credentials](sc-security-oauth2-client.md) →

---

Más allá de la configuración básica de Resource Server JWT, existen tres necesidades avanzadas que aparecen en producción: acceder a claims anidadas en el payload del JWT (como `realm_access.roles` de Keycloak o claims personalizados en namespaces), soportar tokens opacos cuando el JWT no es viable, y configurar multi-tenancy cuando el microservicio debe aceptar tokens de múltiples Authorization Servers. Cada una requiere una extensión diferente del modelo de Spring Security, y confundirlas es la causa de la mayoría de los errores 403 avanzados en proyectos con múltiples realms o proveedores de identidad.

> [PREREQUISITO] Requiere `spring-boot-starter-oauth2-resource-server`. Familiaridad con el fichero 8.2 (Resource Server básico y `JwtAuthenticationConverter`).

## Ejemplo central: claims Keycloak anidadas, opaque tokens y multi-tenancy

### Claims anidadas de Keycloak — `realm_access.roles`

Keycloak distribuye los roles en claims anidadas. `realm_access.roles` contiene roles del realm y `resource_access.[clientId].roles` contiene roles del cliente.

```java
package com.example.security;

import org.springframework.core.convert.converter.Converter;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.oauth2.jwt.Jwt;

import java.util.Collection;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;
import java.util.stream.Stream;

public class KeycloakAuthoritiesConverter implements Converter<Jwt, Collection<GrantedAuthority>> {

    private final String clientId;

    public KeycloakAuthoritiesConverter(String clientId) {
        this.clientId = clientId;
    }

    @Override
    public Collection<GrantedAuthority> convert(Jwt jwt) {
        // 1. Roles del realm: jwt["realm_access"]["roles"]
        Map<String, Object> realmAccess = jwt.getClaim("realm_access");
        List<String> realmRoles = realmAccess != null
            ? (List<String>) realmAccess.get("roles")
            : List.of();

        // 2. Roles del cliente: jwt["resource_access"][clientId]["roles"]
        Map<String, Object> resourceAccess = jwt.getClaim("resource_access");
        List<String> clientRoles = List.of();
        if (resourceAccess != null && resourceAccess.containsKey(clientId)) {
            Map<String, Object> clientAccess = (Map<String, Object>) resourceAccess.get(clientId);
            clientRoles = (List<String>) clientAccess.getOrDefault("roles", List.of());
        }

        return Stream.concat(realmRoles.stream(), clientRoles.stream())
            .map(role -> new SimpleGrantedAuthority("ROLE_" + role))
            .collect(Collectors.toList());
    }
}
```

```java
@Bean
public JwtAuthenticationConverter jwtAuthenticationConverter(
        @Value("${spring.security.oauth2.resourceserver.keycloak.client-id}") String clientId) {
    JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
    converter.setJwtGrantedAuthoritiesConverter(new KeycloakAuthoritiesConverter(clientId));
    converter.setPrincipalClaimName("preferred_username");
    return converter;
}
```

### Tokens opacos con introspección

Cuando el Authorization Server emite tokens opacos (reference tokens, no JWT), Spring Security debe validarlos mediante el endpoint de introspección del AS en cada petición.

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        opaquetoken:
          introspection-uri: https://auth.example.com/oauth2/introspect
          client-id: order-service
          client-secret: ${INTROSPECTION_CLIENT_SECRET}
```

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http
        .csrf(csrf -> csrf.disable())
        .oauth2ResourceServer(oauth2 -> oauth2
            // Usar opaqueToken() en lugar de jwt()
            .opaqueToken(opaque -> opaque
                .introspectionUri("https://auth.example.com/oauth2/introspect")
                .introspectionClientCredentials("order-service", "${secret}")
            )
        )
        .authorizeHttpRequests(auth -> auth.anyRequest().authenticated());

    return http.build();
}
```

> [ADVERTENCIA] Los tokens opacos requieren una llamada al AS en cada petición HTTP (la introspección es síncrona). Esto añade latencia y crea una dependencia de disponibilidad del AS. Usar caché de introspección (`OpaqueTokenIntrospector` personalizado con Caffeine) en producción para mitigar este impacto.

### Multi-tenancy — múltiples Authorization Servers

Cuando el microservicio debe aceptar tokens de varios AS (ej: clientes en diferentes realms de Keycloak), se implementa un `AuthenticationManagerResolver` que selecciona el validador según el `iss` del token.

```java
package com.example.security;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.AuthenticationManagerResolver;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.oauth2.jwt.JwtDecoders;
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationProvider;
import org.springframework.security.web.SecurityFilterChain;

import jakarta.servlet.http.HttpServletRequest;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

@Configuration
@EnableWebSecurity
public class MultiTenantSecurityConfig {

    // Issuers aceptados: tenant-id → issuer-uri
    private static final Map<String, String> TRUSTED_ISSUERS = Map.of(
        "tenant-a", "https://auth.tenant-a.example.com",
        "tenant-b", "https://auth.tenant-b.example.com"
    );

    // Cache de AuthenticationManagers por issuer
    private final Map<String, AuthenticationManager> managers = new ConcurrentHashMap<>();

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .oauth2ResourceServer(oauth2 -> oauth2
                // El resolver selecciona el AuthenticationManager según el issuer del token
                .authenticationManagerResolver(multiTenantResolver())
            )
            .authorizeHttpRequests(auth -> auth.anyRequest().authenticated());

        return http.build();
    }

    @Bean
    public AuthenticationManagerResolver<HttpServletRequest> multiTenantResolver() {
        return request -> {
            // Extraer el issuer del token sin validar la firma
            String issuer = extractIssuerFromRequest(request);

            // Verificar que el issuer está en la lista de confianza
            if (!TRUSTED_ISSUERS.containsValue(issuer)) {
                throw new IllegalArgumentException("Unknown issuer: " + issuer);
            }

            // Crear o recuperar el AuthenticationManager para este issuer
            return managers.computeIfAbsent(issuer, iss -> {
                var decoder = JwtDecoders.fromIssuerLocation(iss);
                var provider = new JwtAuthenticationProvider(decoder);
                return provider::authenticate;
            });
        };
    }

    private String extractIssuerFromRequest(HttpServletRequest request) {
        // Extraer Bearer token del header
        String authorization = request.getHeader("Authorization");
        if (authorization == null || !authorization.startsWith("Bearer ")) {
            throw new IllegalArgumentException("No Bearer token");
        }
        String token = authorization.substring(7);
        // Decodificar el payload sin verificar firma para leer el issuer
        String[] parts = token.split("\\.");
        if (parts.length != 3) throw new IllegalArgumentException("Invalid JWT format");
        String payload = new String(java.util.Base64.getUrlDecoder().decode(parts[1]));
        // Parsear el issuer del payload JSON
        try {
            var node = new com.fasterxml.jackson.databind.ObjectMapper().readTree(payload);
            return node.get("iss").asText();
        } catch (Exception e) {
            throw new IllegalArgumentException("Cannot extract issuer", e);
        }
    }
}
```

> [CONCEPTO] El patrón multi-tenancy con `AuthenticationManagerResolver` es "trust on first decode": el issuer se lee del token sin verificar la firma, se valida contra la lista de issuers conocidos, y solo entonces se verifica criptográficamente con el JWK Set del issuer correcto. Un issuer desconocido se rechaza antes de la validación criptográfica.

## Tabla de elementos clave

| Componente / Técnica | Descripción |
|---|---|
| `realm_access.roles` | Path de claim en tokens Keycloak para roles del realm; acceso: `jwt.getClaim("realm_access")` retorna `Map<String, Object>` |
| `resource_access.[clientId].roles` | Path de claim en tokens Keycloak para roles del cliente específico |
| `opaqueToken()` | Configura Resource Server para tokens opacos; requiere endpoint de introspección |
| `introspection-uri` | URL del endpoint OAuth2 Token Introspection (RFC 7662) del AS |
| `OpaqueTokenIntrospector` | Interfaz para personalizar la introspección; implementar para añadir caché |
| `AuthenticationManagerResolver<HttpServletRequest>` | Selecciona el `AuthenticationManager` por petición; clave para multi-tenancy |
| `JwtDecoders.fromIssuerLocation(issuer)` | Crea un `JwtDecoder` descargando el JWK Set desde el OIDC discovery del issuer |

## Buenas y malas prácticas

**Hacer:**
- Implementar un `OpaqueTokenIntrospector` con caché (Caffeine, Guava) para tokens opacos en producción; sin caché, cada petición HTTP genera una llamada al AS, lo que añade ~50-200ms de latencia por petición.
- Usar una lista de `TRUSTED_ISSUERS` inmutable y configurada externamente (no hardcodeada) en implementaciones multi-tenancy; facilita añadir o revocar tenants sin recompilación.
- Extraer el converter de claims Keycloak a una librería compartida entre microservicios del mismo proyecto; evita duplicar la lógica de extracción de `realm_access.roles` en cada servicio.

**Evitar:**
- Parsear el JWT manualmente con `String.split("\\.")` más allá del resolver de issuer; para cualquier uso de claims en lógica de negocio, usar `@AuthenticationPrincipal Jwt jwt` que garantiza que el token ya fue validado.
- Confiar en el campo `iss` del token para seleccionar el tenant sin una lista de issuers de confianza explícita; un atacante podría construir un JWT con un issuer arbitrario y apuntar a un AS controlado por él.
- Usar tokens opacos cuando el rendimiento es crítico sin haber dimensionado el impacto en latencia; en servicios con > 1000 RPS, la introspección sin caché puede sobrecargar el AS.

---

← [8.2 Resource Server JWT — configuración y personalización](sc-security-resource-server-jwt.md) | [Índice (README.md)](README.md) | [8.4 OAuth2 Client servlet — registro, gestión de tokens y Client Credentials](sc-security-oauth2-client.md) →
