# 8.14 Configuración avanzada de seguridad y producción

← [8.13 CORS, CSRF y cabeceras de seguridad HTTP](sc-security-cors-csrf.md) | [Índice (README.md)](README.md) | [8.15 Testing / Verificación de Spring Cloud Security](sc-security-testing.md) →

---

La configuración de seguridad en producción va más allá de la autenticación y autorización: incluye el hardening del servidor (deshabilitar endpoints de información, limitar métodos HTTP, configurar timeouts de token), la gestión segura de secretos (no hardcodear client secrets), el rate limiting para prevenir abuso, y el logging de auditoría para trazabilidad forense. Estos aspectos no aparecen en los tutoriales básicos pero son los que distinguen una configuración apta para producción de una de desarrollo. En microservicios Spring Cloud, los puntos más críticos son la exposición del Actuator (que puede revelar configuración sensible), la rotación de claves JWK, y la configuración de timeouts de token que equilibre seguridad y experiencia de usuario.

> [PREREQUISITO] Requiere los ficheros 8.1-8.13. Esta sección sintetiza configuraciones de hardening que aplican sobre las configuraciones previas.

## Ejemplo central: hardening de producción

### Asegurar Actuator — exponer solo health y info

```yaml
# application-production.yml
management:
  endpoints:
    web:
      exposure:
        # Solo exponer los endpoints necesarios en producción
        include: health, info, prometheus
        # NO incluir: env, configprops, beans, mappings, heapdump, threaddump
  endpoint:
    health:
      # Mostrar detalles solo a usuarios autenticados con rol MONITORING
      show-details: when-authorized
      roles: MONITORING
    info:
      enabled: true
  # Cambiar el path base de Actuator para dificultar discovery
  web:
    base-path: /internal/management
```

```java
package com.example.security;

import org.springframework.boot.actuate.autoconfigure.security.servlet.EndpointRequest;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Profile;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@Profile("production")
public class ProductionSecurityConfig {

    @Bean
    public SecurityFilterChain productionSecurityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(jwt -> {}))
            .authorizeHttpRequests(auth -> auth
                // Permitir health check sin autenticación (para load balancers y k8s probes)
                .requestMatchers(EndpointRequest.to("health")).permitAll()
                // Prometheus: solo desde red interna o con rol MONITORING
                .requestMatchers(EndpointRequest.to("prometheus")).hasRole("MONITORING")
                // Resto de Actuator: solo MONITORING
                .requestMatchers(EndpointRequest.toAnyEndpoint()).hasRole("MONITORING")
                .anyRequest().authenticated()
            )
            // Rate limiting con bucket4j (ver configuración abajo)
            // .addFilterBefore(rateLimitFilter, UsernamePasswordAuthenticationFilter.class)
            ;

        return http.build();
    }
}
```

### Gestión segura de secretos con Spring Cloud Config + Vault

```yaml
# application.yml — referencias a secretos en Vault
spring:
  cloud:
    vault:
      uri: https://vault.example.com:8200
      authentication: kubernetes  # autenticación mediante service account de K8s
      kubernetes:
        role: order-service
      kv:
        enabled: true
        default-context: order-service

  security:
    oauth2:
      client:
        registration:
          internal-client:
            client-id: order-service
            # El secret viene de Vault, no hardcodeado
            client-secret: ${vault.secret.client-secret}
            authorization-grant-type: client_credentials
```

### Configuración de JWK Set con rotación de claves

```java
package com.example.security;

import com.nimbusds.jose.jwk.JWK;
import com.nimbusds.jose.jwk.JWKSet;
import com.nimbusds.jose.jwk.RSAKey;
import com.nimbusds.jose.jwk.source.JWKSource;
import com.nimbusds.jose.proc.SecurityContext;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.security.KeyStore;
import java.util.List;

@Configuration
public class JwkRotationConfig {

    // Cargar claves desde KeyStore (PKCS12/JKS) — permite rotación sin reinicio
    @Bean
    public JWKSource<SecurityContext> jwkSource(
            @org.springframework.beans.factory.annotation.Value("${keystore.path}")
            String keystorePath,
            @org.springframework.beans.factory.annotation.Value("${keystore.password}")
            String keystorePassword) throws Exception {

        KeyStore keyStore = KeyStore.getInstance("PKCS12");
        try (var stream = new java.io.FileInputStream(keystorePath)) {
            keyStore.load(stream, keystorePassword.toCharArray());
        }

        // Cargar múltiples claves para soportar rotación gradual
        // La primera clave es la activa (firma nuevos tokens)
        // Las demás son claves anteriores (validan tokens emitidos antes de la rotación)
        RSAKey activeKey = RSAKey.load(keyStore, "active-key", keystorePassword.toCharArray());
        RSAKey previousKey = RSAKey.load(keyStore, "previous-key", keystorePassword.toCharArray());

        JWKSet jwkSet = new JWKSet(List.of(activeKey, previousKey.toPublicJWK()));
        return new com.nimbusds.jose.jwk.source.ImmutableJWKSet<>(jwkSet);
    }
}
```

### Rate limiting con Bucket4j en Spring Security

```java
package com.example.security;

import jakarta.servlet.*;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

// Rate limiter simple basado en IP para el endpoint /oauth2/token
@Component
public class TokenEndpointRateLimitFilter implements Filter {

    // En producción usar Bucket4j con Redis para rate limiting distribuido
    private final Map<String, java.util.concurrent.atomic.AtomicInteger> requestCounts =
        new ConcurrentHashMap<>();

    private static final int MAX_REQUESTS_PER_MINUTE = 60;

    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
                         FilterChain chain) throws IOException, ServletException {
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        HttpServletResponse httpResponse = (HttpServletResponse) response;

        if ("/oauth2/token".equals(httpRequest.getRequestURI())) {
            String clientIp = httpRequest.getRemoteAddr();
            var count = requestCounts.computeIfAbsent(
                clientIp, k -> new java.util.concurrent.atomic.AtomicInteger(0));

            if (count.incrementAndGet() > MAX_REQUESTS_PER_MINUTE) {
                httpResponse.setStatus(429);
                httpResponse.setHeader("Retry-After", "60");
                httpResponse.getWriter().write("{\"error\":\"rate_limit_exceeded\"}");
                return;
            }
        }

        chain.doFilter(request, response);
    }
}
```

> [CONCEPTO] La rotación de claves JWK requiere publicar la clave anterior en el JWK Set durante el periodo de transición: los Resource Servers cachean el JWK Set con un TTL (típicamente 5-60 minutos). Si se retira la clave anterior antes de que todos los Resource Servers hayan renovado su caché, los tokens firmados con la clave anterior dejarán de validarse. El periodo de transición debe ser al menos igual al TTL de caché más el tiempo máximo de vida de un access token.

## Tabla de elementos clave

| Elemento | Descripción |
|---|---|
| `EndpointRequest.to("health")` | Matcher de Spring Security para endpoints de Actuator específicos |
| `EndpointRequest.toAnyEndpoint()` | Matcher para cualquier endpoint de Actuator |
| `show-details: when-authorized` | Health endpoint muestra detalles solo a usuarios con el rol configurado |
| `management.web.base-path` | Cambia el prefijo de URLs de Actuator (`/actuator` por defecto) |
| KeyStore PKCS12 | Formato estándar para almacenar claves privadas de forma segura; soporta rotación sin recompilación |
| `ImmutableJWKSet` con múltiples claves | Publica la clave activa y las anteriores; necesario durante rotación gradual |
| Rate limiting por IP | Previene brute force en endpoints de token; usar Bucket4j + Redis en producción distribuida |
| `${vault.secret.client-secret}` | Referencia a secreto en Vault via Spring Cloud Vault; el secret no aparece en ningún fichero de config |

## Buenas y malas prácticas

**Hacer:**
- Revisar qué endpoints de Actuator están expuestos en producción antes del primer despliegue; `/actuator/env`, `/actuator/configprops` y `/actuator/beans` revelan configuración interna (incluyendo secretos si no están enmascarados correctamente).
- Implementar rate limiting en los endpoints de autenticación (`/oauth2/token`, `/login`) para prevenir ataques de credential stuffing; sin rate limiting, un atacante puede probar millones de combinaciones client_id/client_secret.
- Usar `@Profile("production")` para la `SecurityFilterChain` de producción; permite tener configuraciones más permisivas en desarrollo sin riesgo de que lleguen a producción por error.

**Evitar:**
- Hardcodear `client-secret` o contraseñas de keystore en `application.yml` versionado en Git; usar variables de entorno (`${ENV_VAR}`) o secretos de Vault/AWS Secrets Manager.
- Rotar la clave JWK sin un periodo de transición; los Resource Servers que aún tienen la clave anterior en caché empezarán a rechazar tokens válidos, causando una interrupción de servicio.
- Exponer `/actuator/heapdump` en producción; un heap dump puede contener tokens JWT, contraseñas y datos de usuarios en texto plano.

---

← [8.13 CORS, CSRF y cabeceras de seguridad HTTP](sc-security-cors-csrf.md) | [Índice (README.md)](README.md) | [8.15 Testing / Verificación de Spring Cloud Security](sc-security-testing.md) →
