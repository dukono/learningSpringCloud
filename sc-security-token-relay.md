# 8.6 Token Relay en Spring Cloud Gateway

← [8.5 OAuth2 Client avanzado — WebClient, ReactiveManager y logout OIDC](sc-security-oauth2-client-advanced.md) | [Índice (README.md)](README.md) | [8.7 Autorización por URL y métodos con Spring Security](sc-security-authorization.md) →

---

Token Relay es el patrón en el que el API Gateway recibe un token del cliente externo y lo reenvía, sin modificaciones, a los servicios downstream. En Spring Cloud Gateway, el filtro `TokenRelayGatewayFilterFactory` automatiza este proceso: extrae el Bearer token del request entrante y lo adjunta al request que Gateway envía al microservicio destino. Sin este filtro, los microservicios downstream reciben requests sin Authorization header y rechazan la petición con 401. El Token Relay es el mecanismo más simple de propagación de identidad — el token del usuario viaja intacto por toda la cadena de llamadas.

> [PREREQUISITO] Requiere `spring-cloud-starter-gateway` y `spring-boot-starter-oauth2-client`. El Gateway debe estar configurado como OAuth2 Client para que `TokenRelayGatewayFilterFactory` tenga acceso al token del contexto de seguridad.

## Flujo de Token Relay en Gateway

```
Cliente externo
  Authorization: Bearer <token-usuario>
        │
        ▼
┌─────────────────────────────────────────────┐
│  Spring Cloud Gateway                        │
│                                              │
│  1. SecurityWebFilterChain valida el token  │
│     (Gateway como Resource Server)           │
│  2. TokenRelayGatewayFilterFactory extrae   │
│     el token del SecurityContext reactivo    │
│  3. Añade Authorization: Bearer <token>     │
│     al request hacia el servicio downstream │
└─────────────────────────────────────────────┘
        │
        ▼
Microservicio A
  Authorization: Bearer <token-usuario> ← mismo token
```

## Ejemplo central: Gateway como Resource Server con Token Relay

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
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-gateway</artifactId>
    </dependency>
    <!-- OAuth2 Client necesario para TokenRelayGatewayFilterFactory -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-oauth2-client</artifactId>
    </dependency>
    <!-- Resource Server para validar el token entrante en el Gateway -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
    </dependency>
</dependencies>
```

### application.yml — Gateway con Token Relay

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          # Gateway valida el JWT entrante del cliente
          issuer-uri: https://auth.example.com
      client:
        registration:
          # Necesario para que TokenRelayGatewayFilterFactory funcione
          gateway-client:
            provider: my-auth-server
            client-id: api-gateway
            client-secret: ${GATEWAY_CLIENT_SECRET}
            authorization-grant-type: client_credentials
            scope: openid
        provider:
          my-auth-server:
            issuer-uri: https://auth.example.com

  cloud:
    gateway:
      routes:
        - id: order-service
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
          filters:
            # TokenRelay propaga el token del usuario al servicio downstream
            - TokenRelay=
            - StripPrefix=1

        - id: product-service
          uri: lb://product-service
          predicates:
            - Path=/api/products/**
          filters:
            - TokenRelay=
            - StripPrefix=1

        # Ruta sin Token Relay: endpoint público que no requiere autenticación
        - id: public-health
          uri: lb://order-service
          predicates:
            - Path=/api/public/**
          # Sin TokenRelay: no se añade token
```

### SecurityConfig del Gateway

```java
package com.example.gateway;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.reactive.EnableWebFluxSecurity;
import org.springframework.security.config.web.server.ServerHttpSecurity;
import org.springframework.security.web.server.SecurityWebFilterChain;

@Configuration
@EnableWebFluxSecurity
public class GatewaySecurityConfig {

    @Bean
    public SecurityWebFilterChain securityWebFilterChain(ServerHttpSecurity http) {
        return http
            .csrf(ServerHttpSecurity.CsrfSpec::disable)
            // Validar JWT entrante (Gateway como Resource Server)
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> {})
            )
            .authorizeExchange(exchange -> exchange
                .pathMatchers("/api/public/**", "/actuator/health").permitAll()
                .anyExchange().authenticated()
            )
            .build();
    }
}
```

### Token Relay programático en código Java (alternativa)

```java
package com.example.gateway;

import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.core.Ordered;
import org.springframework.http.HttpHeaders;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

import org.springframework.security.core.context.ReactiveSecurityContextHolder;
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationToken;

// Alternativa programática a TokenRelay= cuando se necesita transformar el token
@Component
public class TokenRelayFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        return ReactiveSecurityContextHolder.getContext()
            .map(ctx -> ctx.getAuthentication())
            .filter(auth -> auth instanceof JwtAuthenticationToken)
            .cast(JwtAuthenticationToken.class)
            .map(jwtAuth -> jwtAuth.getToken().getTokenValue())
            .map(tokenValue -> exchange.mutate()
                .request(r -> r.headers(headers ->
                    headers.set(HttpHeaders.AUTHORIZATION, "Bearer " + tokenValue)))
                .build())
            .defaultIfEmpty(exchange)
            .flatMap(chain::filter);
    }

    @Override
    public int getOrder() {
        return -1; // ejecutar antes de los filtros de routing
    }
}
```

> [CONCEPTO] `TokenRelay=` en YAML y la implementación manual con `ReactiveSecurityContextHolder` son equivalentes. La diferencia es que `TokenRelay=` puede configurarse por ruta (algunos endpoints propagan token, otros no), mientras que un `GlobalFilter` aplica a todas las rutas. Usar `TokenRelay=` por ruta cuando algunas rutas son públicas y otras requieren autenticación.

## Tabla de elementos clave

| Componente / Config | Descripción |
|---|---|
| `TokenRelay=` | Filtro de ruta de Gateway que extrae el Bearer token del SecurityContext y lo añade al request downstream |
| `spring-boot-starter-oauth2-client` | Dependencia requerida por `TokenRelayGatewayFilterFactory`; aunque Gateway actúe solo como Resource Server |
| `ReactiveSecurityContextHolder.getContext()` | Acceso programático al SecurityContext en contextos WebFlux; proporciona el `Authentication` del usuario |
| `JwtAuthenticationToken.getToken().getTokenValue()` | Extrae el token JWT string del `Authentication` para añadirlo manualmente al request |
| `exchange.mutate().request(...)` | Crea un `ServerWebExchange` inmutable modificado; patrón estándar para modificar headers en filtros reactivos |
| `StripPrefix=1` | Elimina el primer segmento de path antes del routing; ej: `/api/orders/1` → `/orders/1` |

## Buenas y malas prácticas

**Hacer:**
- Configurar el Gateway como Resource Server (además de OAuth2 Client) para validar el token antes de hacer Token Relay; sin la validación, el Gateway relayaría tokens inválidos o expirados a los servicios downstream.
- Aplicar `TokenRelay=` solo a las rutas que requieren autenticación; las rutas públicas no deben propagar tokens aunque el cliente los envíe (evitar leaks de información).
- Usar `ReactiveSecurityContextHolder` para el Token Relay manual en lugar de leer el header `Authorization` directamente; garantiza que el token ya fue validado por Spring Security antes de propagarlo.

**Evitar:**
- Propagar el token del Gateway añadiendo el header `Authorization` antes de la validación de Spring Security; el token debe haber pasado por el `JwtDecoder` antes de ser propagado para garantizar que es válido.
- Omitir la dependencia `spring-boot-starter-oauth2-client` asumiendo que solo se necesita `oauth2-resource-server`; `TokenRelayGatewayFilterFactory` la requiere aunque el Gateway no actúe como cliente OAuth2 en ningún flujo.
- Usar Token Relay en combinación con autorización en el Gateway sin entender el modelo: el Token Relay propaga el token del usuario — si el Gateway añade restricciones de autorización, el servicio downstream puede recibir tokens con permisos mayores que los que el Gateway habría concedido.

---

← [8.5 OAuth2 Client avanzado — WebClient, ReactiveManager y logout OIDC](sc-security-oauth2-client-advanced.md) | [Índice (README.md)](README.md) | [8.7 Autorización por URL y métodos con Spring Security](sc-security-authorization.md) →
