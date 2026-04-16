# 3.8 CORS y TLS/HTTPS en Gateway

← [3.7 Timeouts y configuración del cliente HTTP Netty](sc-gateway-timeouts.md) | [Índice](README.md) | [3.9 Rutas dinámicas y recarga en caliente](sc-gateway-rutas-dinamicas.md) →

---

## Introducción

CORS y TLS son las dos responsabilidades de seguridad de borde que deben configurarse en el gateway antes que en cada servicio individual. Sin CORS en el gateway, los navegadores rechazan las peticiones cross-origin desde frontends alojados en dominios distintos. Sin TLS, la comunicación entre cliente y gateway está expuesta a intercepción. La razón para configurar ambos en el mismo fichero es que interactúan entre sí: la cabecera `X-Forwarded-Proto` que propaga el gateway debe ser correcta para que los servicios backend generen URLs HTTPS en sus respuestas, lo que afecta a los redirects CORS cuando se usa `RewriteLocationResponseHeader`. El desarrollador necesita configurar CORS y TLS en Gateway para asegurar la comunicación desde el borde y habilitar peticiones cross-origin.

## Diagrama: interacción entre CORS, TLS y cabeceras X-Forwarded

El siguiente diagrama muestra cómo se relacionan la terminación TLS, las cabeceras X-Forwarded y la evaluación CORS.

```
Internet
  │  HTTPS (TLS terminado en Gateway)
  ▼
Gateway (Spring Cloud Gateway)
  ├── TLS: server.ssl.* (termina TLS, habla HTTP internamente con backends)
  │
  ├── CORS (antes del routing):
  │   spring.cloud.gateway.globalcors.cors-configurations
  │   ┌─────────────────────────────────────────────────────┐
  │   │ Petición preflight (OPTIONS):                       │
  │   │  → Gateway responde directamente (sin proxy)        │
  │   │  → Añade: Access-Control-Allow-Origin, etc.         │
  │   └─────────────────────────────────────────────────────┘
  │
  ├── X-Forwarded headers (XForwardedHeadersFilter):
  │   X-Forwarded-For:    IP real del cliente
  │   X-Forwarded-Proto:  https  ← crítico para URLs generadas por el backend
  │   X-Forwarded-Host:   api.example.com
  │   X-Forwarded-Prefix: /api
  │
  └── Proxy HTTP al backend
         │
         ▼
      Backend (recibe HTTP, genera URLs basadas en X-Forwarded-Proto=https)
```

## Ejemplo central

El siguiente ejemplo muestra la configuración completa de CORS global, CORS por ruta, TLS del servidor y TLS del cliente HTTP (para backends con mTLS).

### Configuración TLS del servidor Gateway (terminación HTTPS)

```yaml
# application.yml
# ── TLS del servidor Gateway (Spring Boot 4 / PEM nativo) ────────────
server:
  port: 443
  ssl:
    # Spring Boot 4: soporte nativo de PEM (sin necesidad de keystore JKS)
    certificate: classpath:certs/gateway.crt
    certificate-private-key: classpath:certs/gateway.key
    # CA de confianza (para mTLS con clientes que presenten certificado)
    # trust-certificate: classpath:certs/ca.crt
    # client-auth: NEED  # NONE (default), WANT, NEED

spring:
  cloud:
    gateway:
      # ── CORS global (aplica a todas las rutas) ─────────────────────
      globalcors:
        # add-to-simple-url-handler-mapping: true es necesario cuando
        # se usa Spring Security + Gateway para que los preflight OPTIONS
        # sean respondidos por Gateway y no bloqueados por Security.
        add-to-simple-url-handler-mapping: true
        cors-configurations:
          # Configuración para todas las rutas
          '[/**]':
            allowed-origins:
              - https://app.example.com
              - https://admin.example.com
            allowed-origin-patterns:
              - https://*.example.com    # cualquier subdominio
            allowed-methods:
              - GET
              - POST
              - PUT
              - DELETE
              - OPTIONS
            allowed-headers:
              - Authorization
              - Content-Type
              - X-Correlation-Id
              - X-API-Key
            exposed-headers:
              - X-Total-Count            # headers accesibles desde JavaScript
              - X-Correlation-Id
            allow-credentials: true      # necesario para cookies cross-origin
            max-age: 3600                # segundos de caché del preflight

          # Configuración más restrictiva para rutas de administración
          '[/admin/**]':
            allowed-origins:
              - https://admin.example.com
            allowed-methods:
              - GET
              - POST
            allow-credentials: false
            max-age: 300

      # ── TLS del cliente HTTP (para backends con HTTPS/mTLS) ────────
      httpclient:
        ssl:
          # NUNCA usar en producción; solo para desarrollo
          use-insecure-trust-manager: false
          # CA propia para backends con certificado autofirmado o CA interna
          trusted-x509-certificates:
            - classpath:certs/internal-ca.crt
          # Keystore para mTLS (Gateway se autentica ante el backend)
          key-store: classpath:certs/gateway-client.p12
          key-store-type: PKCS12
          key-store-password: ${KEYSTORE_PASSWORD}
          key-alias: gateway-client

      # ── Cabeceras X-Forwarded ────────────────────────────────────
      x-forwarded:
        enabled: true
        for-enabled: true            # X-Forwarded-For (IP del cliente)
        proto-enabled: true          # X-Forwarded-Proto (https/http)
        host-enabled: true           # X-Forwarded-Host
        port-enabled: true           # X-Forwarded-Port
        prefix-enabled: true         # X-Forwarded-Prefix
        # for-append: true → añade al header existente (multi-proxy)
        # for-append: false → sobreescribe (solo confiar en primer proxy)
        for-append: false
```

### CORS por ruta individual con AddResponseHeader

Cuando se necesita CORS en una sola ruta (sin configuración global), se puede usar `AddResponseHeader`:

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: public-api-cors-route
          uri: lb://public-api-service
          predicates:
            - Path=/api/public/**
          filters:
            - AddResponseHeader=Access-Control-Allow-Origin, https://app.example.com
            - AddResponseHeader=Access-Control-Allow-Methods, GET, POST, OPTIONS
            - AddResponseHeader=Access-Control-Allow-Headers, Authorization, Content-Type
```

> [ADVERTENCIA] Cuando se configura CORS tanto en `globalcors` como con `AddResponseHeader` en la misma ruta, el resultado son cabeceras duplicadas `Access-Control-Allow-Origin` que el navegador rechaza con el error "Multiple values in 'Access-Control-Allow-Origin' header". Usar exclusivamente uno de los dos mecanismos.

### Configuración con Spring Security y CORS

Cuando Spring Security está activo en el gateway, la configuración CORS de Spring Security y la de Gateway pueden interferir. La práctica recomendada es delegar el CORS a Gateway:

```java
package com.example.gateway;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.reactive.EnableWebFluxSecurity;
import org.springframework.security.config.web.server.ServerHttpSecurity;
import org.springframework.security.web.server.SecurityWebFilterChain;

@Configuration
@EnableWebFluxSecurity
public class SecurityConfig {

    @Bean
    public SecurityWebFilterChain springSecurityFilterChain(ServerHttpSecurity http) {
        return http
            // Delegar CORS completamente a Gateway (spring.cloud.gateway.globalcors)
            // No configurar CORS en Spring Security para evitar duplicados
            .cors(cors -> cors.disable())  // o .cors(Customizer.withDefaults()) si se quiere Security CORS
            .csrf(csrf -> csrf.disable())  // Gateway no tiene estado; CSRF no aplica
            .authorizeExchange(exchanges -> exchanges
                .pathMatchers("/api/public/**").permitAll()
                .anyExchange().authenticated())
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(jwt -> {}))
            .build();
    }
}
```

## Tabla de elementos clave

La siguiente tabla recoge las propiedades de CORS y TLS más relevantes.

| Parámetro | Tipo | Default | Descripción |
|-----------|------|---------|-------------|
| `gateway.globalcors.cors-configurations` | `Map<String, CorsConfiguration>` | `{}` | Mapa path-pattern → configuración CORS |
| `gateway.globalcors.add-to-simple-url-handler-mapping` | `boolean` | `false` | Necesario para que preflight OPTIONS funcione con Spring Security |
| `server.ssl.certificate` | `Resource` | — | Certificado PEM del servidor (Spring Boot 4) |
| `server.ssl.certificate-private-key` | `Resource` | — | Clave privada PEM del servidor |
| `httpclient.ssl.use-insecure-trust-manager` | `boolean` | `false` | Acepta cualquier certificado del backend (solo desarrollo) |
| `httpclient.ssl.trusted-x509-certificates` | `List<Resource>` | `[]` | CAs de confianza para certificados del backend |
| `httpclient.ssl.key-store` | `Resource` | — | Keystore para mTLS del cliente |
| `gateway.x-forwarded.for-append` | `boolean` | `true` | Añade al header existente (`true`) o sobreescribe (`false`) |
| `cors.allowCredentials` | `boolean` | `false` | Permite cookies cross-origin; requiere `allowedOrigins` sin `*` |

> [EXAMEN] `allowedOriginPatterns` vs `allowedOrigins` con `*`: cuando `allowCredentials=true`, `allowedOrigins=*` es inválido (el spec CORS lo prohíbe). Usar `allowedOriginPatterns: - "https://*.example.com"` que soporta wildcards Y es compatible con `allowCredentials=true`.

> [ADVERTENCIA] `X-Forwarded-Proto: https` es crítico cuando el gateway termina TLS y reenvía por HTTP al backend. Si el backend genera URLs absolutas (ej: links de paginación, redirects OAuth2) sin leer `X-Forwarded-Proto`, generará URLs `http://` que los clientes rechazarán por estar en una página HTTPS (mixed content).

## Buenas y malas prácticas

**Hacer:**
- Usar `add-to-simple-url-handler-mapping=true` cuando Spring Security esté activo: sin esta propiedad, los preflight OPTIONS pueden ser rechazados por Spring Security antes de llegar al handler CORS de Gateway.
- Configurar `max-age` en la respuesta CORS (recomendado: 3600s para producción): reduce el número de peticiones preflight en el navegador, mejorando el tiempo percibido de carga.
- Usar `httpclient.ssl.trusted-x509-certificates` en lugar de `use-insecure-trust-manager` para backends con CA interna: el trust manager inseguro acepta cualquier certificado, incluyendo los de un atacante man-in-the-middle.

**Evitar:**
- Configurar `allowed-origins: "*"` con `allow-credentials: true`: el spec CORS lo prohíbe y el navegador ignorará las cabeceras CORS, produciendo errores oscuros en el frontend.
- Usar `use-insecure-trust-manager: true` fuera del entorno de desarrollo: en producción acepta certificados expirados, autofirmados o de atacantes, eliminando la protección TLS en la comunicación Gateway→Backend.
- Confiar ciegamente en `X-Forwarded-For` para control de acceso sin validar el proxy upstream: cualquier cliente puede falsificar el header `X-Forwarded-For`; usar `for-append=false` y confiar solo en el primer proxy de la cadena.

---

← [3.7 Timeouts y configuración del cliente HTTP Netty](sc-gateway-timeouts.md) | [Índice](README.md) | [3.9 Rutas dinámicas y recarga en caliente](sc-gateway-rutas-dinamicas.md) →
