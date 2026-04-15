# 6.6.3 Spring Security OAuth2 Resource Server con issuer-uri y JWKS

← [6.6.2 JwtAuthFilter con JJWT](./06-25-gateway-jwt-filter.md) | [Índice](./README.md) | [6.6.4 TokenRelay →](./06-27-gateway-token-relay.md)

---

Cuando el sistema de autenticación es un Authorization Server estándar compatible con OpenID Connect (Keycloak, Auth0, Okta, Spring Authorization Server), la opción recomendada es Spring Security OAuth2 Resource Server. La integración es declarativa: con una sola propiedad YAML (`issuer-uri`), Spring Security descubre automáticamente el endpoint JWKS del Authorization Server, descarga las claves públicas y las refresca periódicamente sin necesidad de reinicios. Esto elimina el problema de la rotación manual de claves que requiere un filtro JJWT personalizado.

## Flujo de discovery OIDC y validación JWT

El proceso se divide en dos fases temporales distintas: la fase de arranque, en la que el Gateway consulta el endpoint de discovery del Authorization Server para obtener la URL del JWKS y descarga las claves públicas; y la fase de petición, en la que cada token entrante se valida contra esas claves en caché usando el `kid` del header del JWT. Cuando el Authorization Server rota sus claves, el `kid` desconocido activa un refresco automático del JWKS sin necesidad de reiniciar el Gateway.

```
ARRANQUE DEL GATEWAY:

  Gateway
    │  GET https://auth.miempresa.com/.well-known/openid-configuration
    ▼
  Authorization Server
    │  Responde con JSON: { "jwks_uri": "https://auth.miempresa.com/oauth2/jwks", ... }
    ▼
  Gateway
    │  GET https://auth.miempresa.com/oauth2/jwks
    ▼
  Authorization Server
    │  Responde con: { "keys": [ { "kid": "key-1", "n": "...", "e": "AQAB" } ] }
    ▼
  Gateway cachea las claves públicas (JWKS)


CADA PETICIÓN AUTENTICADA:

  Cliente
    │  Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6ImtleS0xIn0...
    ▼
  Gateway
    │  1. Lee kid="key-1" del header del token
    │  2. Busca clave en caché JWKS → encontrada
    │  3. Verifica firma RSA con la clave pública
    │  4. Valida claims: exp, iss, aud
    │  ¿válido? ── NO ──► 401 Unauthorized
    │       │
    │       SÍ
    ▼
  Microservicio (con headers X-User-Id, X-User-Roles propagados)


ROTACIÓN DE CLAVES (sin reinicio del Gateway):

  Authorization Server publica kid="key-2" en JWKS
  Cliente obtiene token firmado con kid="key-2"
    │
    ▼
  Gateway: kid="key-2" no está en caché
    │  GET https://auth.miempresa.com/oauth2/jwks  ← refresco automático
    ▼
  Caché actualizada con kid="key-2" → validación exitosa
```

## Dependencias

Spring Security OAuth2 Resource Server requiere dos dependencias: el starter de seguridad base de Spring y el módulo de resource server, que incluye la lógica de validación JWT, el soporte para JWKS endpoint y el cliente de OIDC discovery. Sin la segunda dependencia, Spring no reconoce las propiedades `spring.security.oauth2.resourceserver.*`.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
```

## Configuración YAML

Spring Security OAuth2 Resource Server admite tres modos de obtener la clave de verificación: mediante `issuer-uri` cuando el Authorization Server expone el endpoint OIDC de discovery (opción recomendada en producción), mediante `jwk-set-uri` cuando se conoce directamente la URL del JWKS pero no hay discovery, y mediante `public-key-location` con un fichero PEM estático para entornos de desarrollo sin Authorization Server activo.

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          # Opción 1: issuer-uri — Spring descubre el JWKS automáticamente
          # via /.well-known/openid-configuration
          issuer-uri: https://auth.miempresa.com

          # Opción 2: jwk-set-uri — cuando no hay discovery endpoint OIDC
          # jwk-set-uri: https://auth.miempresa.com/oauth2/jwks

          # Opción 3: clave pública estática (solo para desarrollo)
          # public-key-location: classpath:public.pem
```

> [CONCEPTO] Con `issuer-uri`, Spring Security llama al endpoint `https://auth.miempresa.com/.well-known/openid-configuration` en el arranque y obtiene automáticamente la URL del JWKS, los algoritmos soportados y el emisor esperado. Con `jwk-set-uri`, se especifica directamente la URL del JWKS sin discovery. Usar `issuer-uri` siempre que el Authorization Server soporte OIDC discovery.

> [ADVERTENCIA] En el arranque, Spring Security intenta conectar con el `issuer-uri` para hacer el discovery. Si el Authorization Server no está disponible cuando arranca el Gateway, el contexto de Spring falla. Para arranques desconectados (pruebas, desarrollo), usar `jwk-set-uri` apuntando a un mock o `public-key-location` con una clave estática.

## SecurityWebFilterChain para el Gateway reactivo

El Gateway usa WebFlux, por lo que la configuración de seguridad usa `ServerHttpSecurity` (no `HttpSecurity` del Stack MVC):

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.reactive.EnableWebFluxSecurity;
import org.springframework.security.config.web.server.ServerHttpSecurity;
import org.springframework.security.oauth2.server.resource.authentication.JwtGrantedAuthoritiesConverter;
import org.springframework.security.oauth2.server.resource.authentication.ReactiveJwtAuthenticationConverterAdapter;
import org.springframework.security.web.server.SecurityWebFilterChain;

@Configuration
@EnableWebFluxSecurity
public class GatewaySecurityConfig {

    @Bean
    public SecurityWebFilterChain springSecurityFilterChain(ServerHttpSecurity http) {
        return http
            .csrf(ServerHttpSecurity.CsrfSpec::disable) // el Gateway es stateless
            .authorizeExchange(exchanges -> exchanges
                // Rutas públicas: sin autenticación
                .pathMatchers("/actuator/health", "/actuator/info").permitAll()
                .pathMatchers("/api/public/**").permitAll()
                .pathMatchers("/api/auth/**").permitAll()
                // Rutas admin: solo usuarios con rol ADMIN
                .pathMatchers("/api/admin/**").hasRole("ADMIN")
                // El resto: cualquier usuario autenticado
                .anyExchange().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt
                    .jwtAuthenticationConverter(jwtAuthenticationConverter())
                )
            )
            .build();
    }

    private ReactiveJwtAuthenticationConverterAdapter jwtAuthenticationConverter() {
        // Extrae los roles del claim "roles" en lugar del claim estándar "scope"
        JwtGrantedAuthoritiesConverter converter = new JwtGrantedAuthoritiesConverter();
        converter.setAuthoritiesClaimName("roles");
        converter.setAuthorityPrefix("ROLE_");

        return new ReactiveJwtAuthenticationConverterAdapter(
            new org.springframework.security.oauth2.server.resource
                .authentication.JwtAuthenticationConverter() {{
                    setJwtGrantedAuthoritiesConverter(converter);
                }}
        );
    }
}
```

> [EXAMEN] `@EnableWebFluxSecurity` es la anotación correcta para proyectos WebFlux (incluyendo Spring Cloud Gateway). `@EnableWebSecurity` es para proyectos Spring MVC. Mezclarlas provoca un error de contexto al arrancar.

## Propagación de la identidad autenticada como headers

Con Spring Security, la identidad autenticada está en el `SecurityContext`. Para propagarla como headers hacia los microservicios, se añade un `GlobalFilter` que lee el contexto de seguridad reactivo:

```java
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.core.Ordered;
import org.springframework.security.core.context.ReactiveSecurityContextHolder;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationToken;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

@Component
public class SecurityContextPropagationFilter implements GlobalFilter, Ordered {

    @Override
    public int getOrder() {
        return -1;
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        return ReactiveSecurityContextHolder.getContext()
            .map(ctx -> ctx.getAuthentication())
            .filter(auth -> auth instanceof JwtAuthenticationToken)
            .cast(JwtAuthenticationToken.class)
            .flatMap(jwtAuth -> {
                Jwt jwt = jwtAuth.getToken();
                String userId = jwt.getSubject();
                String roles = jwtAuth.getAuthorities().stream()
                    .map(a -> a.getAuthority())
                    .collect(java.util.stream.Collectors.joining(","));

                var mutatedRequest = exchange.getRequest().mutate()
                    .headers(h -> {
                        h.remove("X-User-Id");
                        h.remove("X-User-Roles");
                        if (userId != null) h.add("X-User-Id", userId);
                        if (!roles.isBlank()) h.add("X-User-Roles", roles);
                    })
                    .build();

                return chain.filter(exchange.mutate().request(mutatedRequest).build());
            })
            .switchIfEmpty(chain.filter(exchange)); // ruta pública sin autenticación
    }
}
```

## Rotación automática de claves JWKS

Spring Security descarga las claves del JWKS endpoint al arrancar y las cachea. Cuando recibe un token firmado con una clave que no está en la caché, intenta refrescar el JWKS automáticamente antes de rechazar el token. Este mecanismo permite rotar las claves del Authorization Server sin necesidad de reconfigurar o reiniciar el Gateway.

> [CONCEPTO] La rotación de claves JWKS funciona porque los tokens JWT incluyen el campo `kid` (Key ID) en el header, que identifica qué clave del JWKS se usó para firmar. Cuando el Gateway ve un `kid` desconocido, descarga el JWKS actualizado. Si se implementara con `public-key-location` (clave estática), la rotación requeriría actualizar el fichero y reiniciar el Gateway.

## Tabla de parámetros de configuración

Cada propiedad del namespace `spring.security.oauth2.resourceserver.jwt` controla un aspecto distinto de la validación: cómo se obtienen las claves públicas, qué algoritmos se aceptan y qué claims del token se verifican obligatoriamente. La columna "Cuándo usar" orienta sobre el escenario que justifica cada opción, ya que algunas son mutuamente excluyentes (por ejemplo, `issuer-uri` y `jwk-set-uri` no deben declararse simultáneamente).

| Propiedad | Descripción | Cuándo usar |
|---|---|---|
| `jwt.issuer-uri` | URL base del Authorization Server para OIDC discovery | Authorization Server con endpoint `/.well-known/openid-configuration` |
| `jwt.jwk-set-uri` | URL directa del JWKS endpoint | Sin discovery OIDC, o para apuntar a un mock en tests |
| `jwt.public-key-location` | Ruta a la clave pública PEM (classpath o filesystem) | Desarrollo local, tests sin Authorization Server |
| `jwt.jws-algorithms` | Algoritmos de firma permitidos | Restringir a RS256 o ES256 explícitamente |
| `jwt.audiences` | Claims `aud` válidos | Validar que el token fue emitido para este servicio específico |

## Buenas y malas prácticas

Hacer:
- Validar el claim `aud` (audience) para asegurarse de que el token fue emitido para el Gateway y no para otro servicio:

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth.miempresa.com
          audiences: https://gateway.miempresa.com
```

- Deshabilitar CSRF solo en el Gateway (que es stateless). Si algún microservicio detrás del Gateway sirve aplicaciones web con sesiones de usuario, ese microservicio debe mantener la protección CSRF.

Evitar:
- No configurar `audiences`. Sin validación de audience, un token emitido para el servicio A puede usarse para autenticarse en el servicio B si comparten el mismo Authorization Server.
- No usar `@EnableWebFluxSecurity` en el Gateway y usar `@EnableWebSecurity` en su lugar. El Gateway usa WebFlux y el stack de seguridad MVC no es compatible.

---

← [6.6.2 JwtAuthFilter con JJWT](./06-25-gateway-jwt-filter.md) | [Índice](./README.md) | [6.6.4 TokenRelay →](./06-27-gateway-token-relay.md)
