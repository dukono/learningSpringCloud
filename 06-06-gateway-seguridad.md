# 6.6 Seguridad: JWT y Spring Security OAuth2

← [6.5 Rate Limiting](./06-05-gateway-rate-limiting.md) | [Índice](./README.md) | [6.7 Testing del Gateway →](./06-07-gateway-testing.md)

---

El Gateway es el punto de entrada único a la arquitectura, lo que lo convierte en el lugar natural para centralizar la validación de tokens. Sin seguridad en el Gateway, cada microservicio tendría que validar el JWT de forma independiente: añadir la dependencia de JJWT o Spring Security, conocer la clave de firma, y mantener esa lógica actualizada en cada servicio. Centralizarla en el Gateway significa que los microservicios confían en los headers que el Gateway propaga (`X-User-Id`, `X-User-Roles`) sin necesidad de revalidar el token.

> [PREREQUISITO] La lógica de seguridad se implementa como `GlobalFilter` (ver [6.4](./06-04-gateway-filtros-custom.md)). Si se usa Spring Security con OAuth2, los microservicios deben estar en una red privada inaccesible directamente desde fuera del Gateway.

---

## 6.6.1 Flujo de validación JWT centralizada y propagación de identidad

```
Cliente
   │
   │  Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
   ▼
Gateway — JwtAuthFilter (GlobalFilter, order -200)
   │
   ├── ¿tiene header Authorization?  ──NO──▶  401 Unauthorized
   │
   ├── ¿empieza por "Bearer "?        ──NO──▶  401 Unauthorized
   │
   ├── ¿firma válida? ¿no expirado?   ──NO──▶  401 Unauthorized
   │
   └── ✅ Token válido
         │
         │  propaga headers al microservicio:
         │  X-User-Id: user-123
         │  X-User-Roles: ROLE_USER,ROLE_ADMIN
         ▼
   Microservicio destino
   (lee X-User-Id, no valida el token)
```

> [CONCEPTO] El microservicio **confía en los headers propagados por el Gateway** porque la petición llega desde la red interna, no desde internet. Si un microservicio es accesible directamente desde fuera (sin pasar por el Gateway), debe validar el token de forma independiente también.

---

## 6.6.2 JwtAuthFilter con JJWT (clave simétrica HS256)

Esta implementación valida tokens JWT firmados con clave simétrica (HS256) y propaga la identidad del usuario como headers al microservicio downstream.

```xml
<!-- pom.xml -->
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.11.5</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.11.5</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.11.5</version>
    <scope>runtime</scope>
</dependency>
```

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
import java.nio.charset.StandardCharsets;
import java.util.List;

@Component
public class JwtAuthFilter implements GlobalFilter, Ordered {

    @Value("${jwt.secret}")
    private String jwtSecret;

    private static final List<String> RUTAS_PUBLICAS = List.of(
        "/auth/login", "/auth/register", "/actuator/health"
    );

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String path = exchange.getRequest().getURI().getPath();

        if (RUTAS_PUBLICAS.stream().anyMatch(path::startsWith)) {
            return chain.filter(exchange);   // ruta pública: no validar token
        }

        String authHeader = exchange.getRequest().getHeaders().getFirst("Authorization");

        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            return rechazar(exchange, HttpStatus.UNAUTHORIZED);
        }

        String token = authHeader.substring(7);

        try {
            Claims claims = Jwts.parserBuilder()
                .setSigningKey(Keys.hmacShaKeyFor(jwtSecret.getBytes(StandardCharsets.UTF_8)))
                .build()
                .parseClaimsJws(token)
                .getBody();

            // Propagar identidad al microservicio como headers
            // Los microservicios confían en estos headers porque vienen de la red interna
            ServerHttpRequest mutatedRequest = exchange.getRequest().mutate()
                .header("X-User-Id", claims.getSubject())
                .header("X-User-Roles", claims.get("roles", String.class))
                .build();

            return chain.filter(exchange.mutate().request(mutatedRequest).build());

        } catch (JwtException e) {
            return rechazar(exchange, HttpStatus.UNAUTHORIZED);
        }
    }

    private Mono<Void> rechazar(ServerWebExchange exchange, HttpStatus status) {
        exchange.getResponse().setStatusCode(status);
        return exchange.getResponse().setComplete();
    }

    @Override
    public int getOrder() {
        return -200;   // primero de todos los filtros: si falla, no llega a los demás
    }
}
```

```yaml
# application.yml
jwt:
  secret: ${JWT_SECRET}   # mínimo 32 caracteres para HS256; en producción usar variable de entorno
```

---

## 6.6.3 Propagación de identidad: `X-User-Id`, `X-User-Roles`

Los microservicios pueden necesitar la identidad del usuario y sus roles para aplicar lógica de negocio. En lugar de validar el token JWT ellos mismos, que sería redundante y costoso, el Gateway puede propagar esta información como headers HTTP.

### 6.6.3.1 Ejemplo de implementación

El siguiente ejemplo muestra cómo el `JwtAuthFilter` propaga la identidad del usuario como headers:

```java
// Dentro de JwtAuthFilter.java

Claims claims = Jwts.parserBuilder()
    .setSigningKey(Keys.hmacShaKeyFor(jwtSecret.getBytes(StandardCharsets.UTF_8)))
    .build()
    .parseClaimsJws(token)
    .getBody();

// Propagar identidad al microservicio como headers
// Los microservicios confían en estos headers porque vienen de la red interna
ServerHttpRequest mutatedRequest = exchange.getRequest().mutate()
    .header("X-User-Id", claims.getSubject())
    .header("X-User-Roles", claims.get("roles", String.class))
    .build();

return chain.filter(exchange.mutate().request(mutatedRequest).build());
```

### 6.6.3.2 Consideraciones

- Los microservicios deben estar configurados para leer estos headers y aplicar la lógica de negocio correspondiente.
- Es importante que los headers `X-User-Id` y `X-User-Roles` no sean enviados por el cliente, ya que podrían ser fácilmente falsificados. El Gateway debe ser el único responsable de añadir estos headers.

---

## 6.6.4 Spring Security OAuth2 Resource Server con `issuer-uri` y JWKS

Cuando se utilizan tokens emitidos por un servidor de autorización externo (como Keycloak, Auth0 o Azure AD), es recomendable usar Spring Security con OAuth2 Resource Server. Esta configuración valida la firma del token contra el JWKS (JSON Web Key Set) del servidor de autorización, gestiona automáticamente la rotación de claves y rellena el `SecurityContext` para que `TokenRelay` pueda propagar el token a los microservicios.

### 6.6.4.1 Ejemplo de implementación

```xml
<!-- pom.xml — en lugar de jjwt -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.Customizer;
import org.springframework.security.config.annotation.web.reactive.EnableWebFluxSecurity;
import org.springframework.security.config.web.server.ServerHttpSecurity;
import org.springframework.security.web.server.SecurityWebFilterChain;

@Configuration
@EnableWebFluxSecurity
public class SecurityConfig {

    @Bean
    public SecurityWebFilterChain securityFilterChain(ServerHttpSecurity http) {
        http
            .authorizeExchange(exchanges -> exchanges
                .pathMatchers("/auth/**", "/actuator/health").permitAll()
                .pathMatchers("/admin/**").hasRole("ADMIN")
                .anyExchange().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()));

        return http.build();
    }
}
```

```yaml
# application.yml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth-server.miempresa.com
          # Spring descubre automáticamente el JWKS desde:
          # https://auth-server.miempresa.com/.well-known/openid-configuration
          # Alternativa directa si el servidor no tiene endpoint de discovery:
          # jwk-set-uri: https://auth-server.miempresa.com/.well-known/jwks.json

  cloud:
    gateway:
      routes:
        - id: pedidos-route
          uri: lb://pedidos-service
          predicates:
            - Path=/api/pedidos/**
          filters:
            - StripPrefix=1
            - TokenRelay=   # propaga el Bearer token al microservicio; requiere Spring Security activo
```

### 6.6.4.2 Consideraciones

- Usar `issuer-uri` en lugar de `jwk-set-uri` fija permite que Spring Security gestione automáticamente la rotación de claves cuando el servidor de autorización renueva sus claves.
- Asegurarse de que el servidor de autorización esté configurado correctamente y sea accesible desde el Gateway.

---

## 6.6.5 `TokenRelay` — propagación automática del Bearer token

Una vez que el token JWT es validado por Spring Security, se puede usar `TokenRelay` para propagar automáticamente el token a los microservicios. Esto simplifica la arquitectura, ya que los microservicios no necesitan preocuparse por la validación del token ni por la extracción de la información del usuario.

### 6.6.5.1 Ejemplo de implementación

```yaml
# application.yml
spring:
  cloud:
    gateway:
      routes:
        - id: pedidos-route
          uri: lb://pedidos-service
          predicates:
            - Path=/api/pedidos/**
          filters:
            - StripPrefix=1
            - TokenRelay=   # propaga el Bearer token al microservicio; requiere Spring Security activo
```

### 6.6.5.2 Consideraciones

- `TokenRelay` solo está disponible si se usa Spring Security con OAuth2 Resource Server.
- Asegurarse de que los microservicios estén configurados para aceptar el token en el formato esperado.

---

## 6.6.6 Parámetros de configuración de seguridad

| Parámetro | Tipo | Valor por defecto | Descripción |
|---|---|---|---|
| `jwt.secret` | String | — | Clave simétrica para JJWT (HS256); mínimo 32 caracteres |
| `spring.security.oauth2.resourceserver.jwt.issuer-uri` | String | — | URL del servidor de autorización; descubre JWKS automáticamente |
| `spring.security.oauth2.resourceserver.jwt.jwk-set-uri` | String | — | URL directa del endpoint JWKS (si no hay discovery) |

---

## 6.6.7 Buenas y malas prácticas

**Hacer:**
- Poner el `JwtAuthFilter` con `getOrder() = -200` para que sea el primer filtro ejecutado — si el token no es válido, no tiene sentido ejecutar rate limiting, circuit breaker ni ningún otro filtro.
- Propagar la identidad del usuario como headers (`X-User-Id`, `X-User-Roles`) para que los microservicios puedan aplicar lógica de negocio basada en el usuario sin necesidad de validar el token ellos mismos.
- Usar `issuer-uri` con OAuth2 Resource Server en lugar de `jwk-set-uri` fija — `issuer-uri` gestiona automáticamente la rotación de claves cuando el servidor de autorización las renueva.

**Evitar:**
- Guardar `jwt.secret` como texto plano en el `application.yml` del repositorio — usar variables de entorno (`${JWT_SECRET}`) o Spring Cloud Config con cifrado (ver [03-06](./03-06-config-avanzado.md)).
- Confiar en los headers `X-User-Id` enviados directamente por el cliente — cualquier cliente puede forjar esos headers; el Gateway debe sobreescribirlos siempre con los valores extraídos del token validado.
- Configurar Spring Security en los microservicios internos y también en el Gateway con la misma clave — la doble validación es redundante si los microservicios solo son accesibles a través del Gateway; añade latencia sin beneficio.

---

## 6.6.8 Comparación: JwtAuthFilter manual vs Spring Security OAuth2

| Aspecto | JwtAuthFilter con JJWT | Spring Security OAuth2 Resource Server |
|---|---|---|
| **Complejidad** | Baja — un solo bean | Media — configuración de Spring Security WebFlux |
| **Rotación de claves** | Manual — hay que actualizar `jwt.secret` | Automática — descarga el JWKS periódicamente |
| **Múltiples proveedores** | No soportado directamente | Soportado con configuración adicional |
| **`TokenRelay`** | No disponible (no hay `SecurityContext`) | Disponible — propaga el token automáticamente |
| **Soporte OpenID Connect** | No | Sí |
| **Cuándo usarlo** | Token JWT propio, clave simétrica, sin IDP externo | Keycloak, Auth0, Azure AD, cualquier IDP estándar |

---

## Ejemplo integrado: seguridad completa en producción

Una configuración real combina JWT + OAuth2 + propagación de identidad + rutas públicas. Este ejemplo muestra el application.yml completo de un Gateway de producción con todas las piezas conectadas:

```yaml
# application.yml — Gateway de producción
server:
  port: 8080

spring:
  application:
    name: gateway-service

  data:
    redis:
      host: ${REDIS_HOST:localhost}
      port: 6379

  cloud:
    gateway:
      default-filters:
        - DedupeResponseHeader=Access-Control-Allow-Origin RETAIN_FIRST
        - AddResponseHeader=X-Gateway-Version, 2.0

      globalcors:
        cors-configurations:
          '[/**]':
            allowed-origins: "${ALLOWED_ORIGINS:http://localhost:3000}"
            allowed-methods: GET,POST,PUT,DELETE,OPTIONS
            allowed-headers: Authorization,Content-Type,X-API-Key
            allow-credentials: true
            max-age: 3600

      routes:
        # Rutas públicas — sin JWT
        - id: auth-route
          uri: lb://auth-service
          predicates:
            - Path=/auth/**
          order: 1

        # Ruta protegida con JWT + rate limiting + circuit breaker
        - id: pedidos-route
          uri: lb://pedidos-service
          predicates:
            - Path=/api/pedidos/**
          filters:
            - StripPrefix=1
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 20
                redis-rate-limiter.burstCapacity: 40
                key-resolver: "#{@userKeyResolver}"
            - name: CircuitBreaker
              args:
                name: pedidosCB
                fallbackUri: forward:/fallback/pedidos
            - TokenRelay=
          order: 10

        # Admin — solo red interna, sin rate limiting externo
        - id: admin-route
          uri: lb://admin-service
          predicates:
            - Path=/admin/**
            - RemoteAddr=10.0.0.0/8
          filters:
            - AddRequestHeader=X-Admin-Origin, gateway-internal
          order: 5

      httpclient:
        connect-timeout: 2000
        response-timeout: 15s
      metrics:
        enabled: true

jwt:
  secret: ${JWT_SECRET}

resilience4j:
  circuitbreaker:
    instances:
      pedidosCB:
        slidingWindowSize: 10
        failureRateThreshold: 50
        waitDurationInOpenState: 10s

management:
  endpoints:
    web:
      exposure:
        include: health,gateway,prometheus
```

> [ADVERTENCIA] No configurar `spring.security.oauth2.resourceserver` y el `JwtAuthFilter` manual en el mismo Gateway — los dos mecanismos intentarán validar el token y el segundo siempre verá que el `SecurityContext` ya fue procesado por el primero, generando comportamiento impredecible. Elegir uno de los dos enfoques (ver tabla de comparación al final del fichero).

---

← [6.5 Rate Limiting](./06-05-gateway-rate-limiting.md) | [Índice](./README.md) | [6.7 Testing del Gateway →](./06-07-gateway-testing.md)
