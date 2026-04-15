# 6.6.2 JwtAuthFilter con JJWT y propagación de identidad: X-User-Id, X-User-Roles

← [6.6.1 Flujo de validación JWT](./06-24-gateway-seguridad-flujo.md) | [Índice](./README.md) | [6.6.3 Spring Security OAuth2 →](./06-26-gateway-oauth2.md)

---

El filtro JJWT manual es la opción adecuada cuando el sistema de autenticación es propietario (no sigue el estándar OpenID Connect), cuando el formato del token tiene claims personalizados que no se mapean directamente con la convención OIDC, o cuando se quiere evitar la dependencia de Spring Security en el Gateway. A diferencia de Spring Security OAuth2 Resource Server, el filtro JJWT es transparente en cuanto a su implementación y no introduce el modelo de autenticación de Spring Security en el pipeline reactivo del Gateway.

## Dependencias

JJWT se distribuye en tres artefactos separados: `jjwt-api` contiene las interfaces públicas y es la única dependencia necesaria en tiempo de compilación; `jjwt-impl` aporta la implementación del parser y el generador de tokens, y se declara como `runtime` para que el código de la aplicación no dependa directamente de las clases internas; `jjwt-jackson` proporciona la serialización JSON a través de Jackson, también solo en runtime.

```xml
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.6</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.6</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.12.6</version>
    <scope>runtime</scope>
</dependency>
```

## Implementación completa del JwtAuthFilter

El filtro implementa `GlobalFilter` y `Ordered` para interceptar todas las peticiones antes de que alcancen los filtros de ruta. La lógica central sigue tres pasos: comprobar si la ruta es pública (y dejar pasar sin validar), extraer y verificar el token Bearer con JJWT, y mutar la petición para propagar los claims extraídos como headers internos eliminando previamente cualquier valor que el cliente pudiera haber inyectado.

```java
import io.jsonwebtoken.Claims;
import io.jsonwebtoken.JwtException;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.security.Keys;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.core.Ordered;
import org.springframework.http.HttpStatus;
import org.springframework.http.server.reactive.ServerHttpRequest;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

import javax.crypto.SecretKey;
import java.nio.charset.StandardCharsets;
import java.util.List;

@Component
public class JwtAuthFilter implements GlobalFilter, Ordered {

    private static final String BEARER_PREFIX = "Bearer ";
    private static final List<String> PUBLIC_PATHS = List.of(
        "/actuator/health",
        "/actuator/info",
        "/api/public/",
        "/api/auth/"
    );

    private final SecretKey signingKey;

    public JwtAuthFilter(@Value("${jwt.secret}") String secret) {
        // La clave debe tener al menos 256 bits (32 bytes) para HS256
        this.signingKey = Keys.hmacShaKeyFor(secret.getBytes(StandardCharsets.UTF_8));
    }

    @Override
    public int getOrder() {
        return -1; // antes de los filtros de ruta
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String path = exchange.getRequest().getURI().getPath();

        // Rutas públicas: pasar sin validar
        if (isPublicPath(path)) {
            return chain.filter(exchange);
        }

        String authHeader = exchange.getRequest().getHeaders()
            .getFirst("Authorization");

        if (authHeader == null || !authHeader.startsWith(BEARER_PREFIX)) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }

        String token = authHeader.substring(BEARER_PREFIX.length());

        try {
            Claims claims = Jwts.parser()
                .verifyWith(signingKey)
                .build()
                .parseSignedClaims(token)
                .getPayload();

            String userId = claims.getSubject();
            String roles = extractRoles(claims);
            String tenantId = claims.get("tenant_id", String.class);

            // Eliminar headers externos que pudieran suplantar identidad
            // y propagar los claims validados del JWT
            ServerHttpRequest mutatedRequest = exchange.getRequest().mutate()
                .headers(headers -> {
                    headers.remove("X-User-Id");
                    headers.remove("X-User-Roles");
                    headers.remove("X-Tenant-Id");
                    if (userId != null) headers.add("X-User-Id", userId);
                    if (roles != null) headers.add("X-User-Roles", roles);
                    if (tenantId != null) headers.add("X-Tenant-Id", tenantId);
                })
                .build();

            return chain.filter(exchange.mutate().request(mutatedRequest).build());

        } catch (JwtException e) {
            // JwtException cubre: firma inválida, token expirado, token malformado
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }
    }

    private boolean isPublicPath(String path) {
        return PUBLIC_PATHS.stream().anyMatch(path::startsWith);
    }

    @SuppressWarnings("unchecked")
    private String extractRoles(Claims claims) {
        Object rolesObj = claims.get("roles");
        if (rolesObj instanceof List<?> roleList) {
            return String.join(",", (List<String>) roleList);
        }
        if (rolesObj instanceof String rolesStr) {
            return rolesStr;
        }
        return null;
    }
}
```

## Configuración de la clave de firma

La clave secreta no se escribe directamente en el YAML de la aplicación sino que se inyecta desde una variable de entorno en tiempo de ejecución. Esto evita que la clave quede expuesta en el repositorio de código o en los logs del sistema de CI/CD. El valor de `JWT_SECRET` debe contener al menos 32 caracteres para que JJWT acepte la clave con el algoritmo HS256 sin lanzar `WeakKeyException` al arrancar.

```yaml
jwt:
  secret: ${JWT_SECRET}   # variable de entorno; mínimo 32 caracteres para HS256
```

## Parámetros de configuración del filtro

La tabla siguiente describe las propiedades que controlan el comportamiento del `JwtAuthFilter`. Todas se leen en el arranque de la aplicación; un valor inválido (clave demasiado corta, lista de rutas mal formada) provoca un error en el arranque, no en tiempo de ejecución.

| Parámetro | Tipo | Valor por defecto | Descripción |
|---|---|---|---|
| `jwt.secret` | `String` | — (obligatorio) | Clave secreta para el algoritmo HS256. Debe tener al menos 32 caracteres (256 bits). Se recomienda inyectar desde variable de entorno. |
| `gateway.security.public-paths` | `List<String>` | `[]` | Rutas que se sirven sin requerir token JWT. Se evalúan con `startsWith()` sobre el path de la petición. |

> [ADVERTENCIA] La clave secreta para HS256 debe tener al menos 256 bits (32 bytes). JJWT lanza `WeakKeyException` si la clave es más corta. Para HS512, el mínimo es 512 bits (64 bytes). Una clave de desarrollo como `"secret"` o `"password"` falla al arrancar en producción o genera advertencias.

> [EXAMEN] `JwtException` es la clase base de todas las excepciones de JJWT: `ExpiredJwtException`, `MalformedJwtException`, `SignatureException`, `UnsupportedJwtException`. Capturar `JwtException` en el bloque catch cubre todos los casos de token inválido sin necesidad de múltiples catch separados.

## Cómo el microservicio lee los headers propagados

El microservicio receptor no valida el JWT; simplemente lee los headers que el Gateway propagó:

```java
// En un controlador Spring MVC/WebFlux del microservicio:
@GetMapping("/mis-pedidos")
public Mono<List<Pedido>> getMisPedidos(
        @RequestHeader("X-User-Id") String userId,
        @RequestHeader(value = "X-User-Roles", required = false) String roles) {
    return pedidoService.findByUserId(userId);
}
```

Para acceder desde cualquier capa del microservicio sin pasarlo por parámetros:

```java
// Alternativa reactiva: leer el header desde el ServerWebExchange
exchange.getRequest().getHeaders().getFirst("X-User-Id")
```

## Tabla de claims JWT

La tabla siguiente mapea los claims estándar de RFC 7519 y los claims personalizados más habituales con los headers internos que el `JwtAuthFilter` propaga al microservicio. Conocer esta correspondencia es esencial para que los microservicios lean la identidad del usuario sin necesidad de decodificar el token por sí mismos.

| Claim JWT | Estándar RFC 7519 | Header interno propagado | Ejemplo de valor |
|---|---|---|---|
| `sub` | Sí | `X-User-Id` | `usr-12345` |
| `exp` | Sí | No propagado (solo para validación) | `1735689600` |
| `iat` | Sí | No propagado | `1735686000` |
| `iss` | Sí | No propagado (para validación) | `https://auth.miempresa.com` |
| `roles` | No (custom) | `X-User-Roles` | `["ROLE_USER","ROLE_ADMIN"]` |
| `tenant_id` | No (custom) | `X-Tenant-Id` | `tenant-acme` |
| `email` | No (OIDC) | `X-User-Email` | `user@acme.com` |

## Buenas y malas prácticas

Hacer:
- Usar RS256 (RSA con clave pública/privada) en lugar de HS256 (clave simétrica) cuando el Authorization Server y el Gateway no son el mismo proceso. Con RS256, solo el Authorization Server conoce la clave privada para firmar; el Gateway solo necesita la clave pública para verificar. Si la clave pública del Gateway se expone accidentalmente, solo compromete la capacidad de verificación, no la de emisión de tokens.
- Establecer TTL cortos en los tokens de acceso (15-60 minutos) y usar refresh tokens para renovación. Los tokens de larga duración son difíciles de revocar.
- Para revocar tokens antes de su expiración, mantener una blacklist en Redis y consultarla en el filtro:

```java
// Consulta reactiva a Redis para verificar si el token está revocado
return redisTemplate.hasKey("jwt:blacklist:" + token)
    .flatMap(revoked -> {
        if (revoked) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }
        return chain.filter(mutatedExchange);
    });
```

Evitar:
- No eliminar los headers `X-User-Id`, `X-User-Roles` y `X-Tenant-Id` de la petición entrante antes de añadir los del JWT. Un cliente que conoce los nombres de los headers internos puede suplantar cualquier identidad si el Gateway no los elimina primero.
- Usar `HS256` con una clave corta o predecible. Las claves de menos de 32 bytes son vulnerables a ataques de fuerza bruta offline contra los tokens capturados.
- Loguear el token JWT completo en los logs de debug o de error. Los tokens son credenciales y su presencia en logs los expone a cualquier persona con acceso al sistema de logging.

---

← [6.6.1 Flujo de validación JWT](./06-24-gateway-seguridad-flujo.md) | [Índice](./README.md) | [6.6.3 Spring Security OAuth2 →](./06-26-gateway-oauth2.md)
