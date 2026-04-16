# 8.13 CORS, CSRF y cabeceras de seguridad HTTP

← [8.12 Seguridad en microservicios reactivos con WebFlux](sc-security-reactive.md) | [Índice (README.md)](README.md) | [8.14 Configuración avanzada de seguridad y producción](sc-security-production.md) →

---

CORS, CSRF y las cabeceras de seguridad HTTP son los tres mecanismos de defensa perimetral que protegen las aplicaciones web de ataques desde el browser. En microservicios con APIs REST protegidas por JWT, CSRF puede deshabilitarse con seguridad (los JWT son stateless y no usan cookies de sesión), pero CORS debe configurarse cuidadosamente para permitir solo los orígenes legítimos. Las cabeceras de seguridad HTTP (Content-Security-Policy, HSTS, X-Frame-Options) las añade Spring Security automáticamente con valores conservadores, pero requieren ajuste en aplicaciones con requisitos específicos. Confundir CORS con seguridad de autenticación es el error más frecuente: CORS es un mecanismo del browser, no del servidor — un servidor sin CORS configurado sigue siendo vulnerable a llamadas desde servidores non-browser.

> [PREREQUISITO] Requiere `spring-boot-starter-security`. Para WebFlux: `spring-boot-starter-webflux` con `@EnableWebFluxSecurity`.

## Ejemplo central: CORS, CSRF y cabeceras HTTP en servlet y WebFlux

### Configuración CORS en SecurityConfig (servlet)

```java
package com.example.security;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.web.cors.CorsConfiguration;
import org.springframework.web.cors.CorsConfigurationSource;
import org.springframework.web.cors.UrlBasedCorsConfigurationSource;

import java.util.List;

@Configuration
@EnableWebSecurity
public class CorsSecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            // 1. CSRF: deshabilitado para APIs REST con JWT (stateless)
            // Justificación: JWT en Authorization header no es enviado automáticamente
            // por el browser en cross-site requests (CSRF explota cookies de sesión)
            .csrf(csrf -> csrf.disable())

            // 2. CORS: delegar al CorsConfigurationSource definido abajo
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))

            // 3. Headers de seguridad HTTP: habilitados por defecto, personalizar según necesidad
            .headers(headers -> headers
                // HSTS: obligar HTTPS durante 1 año, incluir subdominios
                .httpStrictTransportSecurity(hsts -> hsts
                    .includeSubDomains(true)
                    .maxAgeInSeconds(31536000)
                )
                // X-Frame-Options: prevenir clickjacking
                .frameOptions(frame -> frame.deny())
                // Content-Security-Policy: restringir fuentes de scripts/estilos
                .contentSecurityPolicy(csp -> csp
                    .policyDirectives("default-src 'self'; script-src 'self'; style-src 'self'")
                )
            )

            .oauth2ResourceServer(oauth2 -> oauth2.jwt(jwt -> {}))
            .authorizeHttpRequests(auth -> auth.anyRequest().authenticated());

        return http.build();
    }

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();

        // Orígenes permitidos (NUNCA usar "*" con credenciales)
        config.setAllowedOrigins(List.of(
            "https://app.example.com",
            "https://admin.example.com"
        ));

        // Métodos HTTP permitidos
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS"));

        // Headers permitidos en la petición
        config.setAllowedHeaders(List.of(
            "Authorization",
            "Content-Type",
            "X-Requested-With",
            "X-Correlation-ID"
        ));

        // Headers expuestos al browser en la respuesta
        config.setExposedHeaders(List.of("X-Total-Count", "X-Correlation-ID"));

        // Cache del preflight OPTIONS (10 minutos)
        config.setMaxAge(600L);

        // No usar allowCredentials(true) con allowedOrigins("*")
        // Si se necesitan cookies cross-origin: especificar orígenes concretos
        // config.setAllowCredentials(true);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", config);

        // Endpoints públicos con política CORS más permisiva
        CorsConfiguration publicConfig = new CorsConfiguration();
        publicConfig.setAllowedOrigins(List.of("*"));
        publicConfig.setAllowedMethods(List.of("GET"));
        source.registerCorsConfiguration("/api/public/**", publicConfig);

        return source;
    }
}
```

### Configuración CORS en WebFlux

```java
package com.example.security;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.reactive.EnableWebFluxSecurity;
import org.springframework.security.config.web.server.ServerHttpSecurity;
import org.springframework.security.web.server.SecurityWebFilterChain;
import org.springframework.web.cors.CorsConfiguration;
import org.springframework.web.cors.reactive.CorsConfigurationSource;
import org.springframework.web.cors.reactive.UrlBasedCorsConfigurationSource;

import java.util.List;

@Configuration
@EnableWebFluxSecurity
public class ReactiveCorsCsrfConfig {

    @Bean
    public SecurityWebFilterChain securityWebFilterChain(ServerHttpSecurity http) {
        return http
            .csrf(ServerHttpSecurity.CsrfSpec::disable)
            // En WebFlux, cors() delega a CorsWebFilter que se configura mediante
            // un CorsConfigurationSource reactivo
            .cors(cors -> cors.configurationSource(reactiveCorsSource()))
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(jwt -> {}))
            .authorizeExchange(exchange -> exchange.anyExchange().authenticated())
            .build();
    }

    @Bean
    public CorsConfigurationSource reactiveCorsSource() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOrigins(List.of("https://app.example.com"));
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
        config.setAllowedHeaders(List.of("Authorization", "Content-Type"));
        config.setMaxAge(600L);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);
        return source;
    }
}
```

> [CONCEPTO] CORS es un mecanismo de seguridad del browser (Same-Origin Policy): el browser bloquea requests JavaScript a orígenes diferentes a menos que el servidor responda con los headers CORS adecuados. Configurar CORS solo afecta a llamadas desde browsers — las llamadas desde servidores, curl, Postman, etc. ignoran completamente los headers CORS. Por esto, CORS no es un mecanismo de autenticación ni autorización.

> [ADVERTENCIA] Si la aplicación usa Spring Security Y define un `@CrossOrigin` a nivel de controlador, puede haber conflictos: Spring Security procesa CORS antes de que llegue al controlador. La política de CORS debe definirse exclusivamente en `SecurityFilterChain` usando `CorsConfigurationSource`; los `@CrossOrigin` pueden quedar silenciados.

## Tabla de elementos clave

| Elemento | Descripción |
|---|---|
| `csrf.disable()` | Desactiva la protección CSRF; seguro solo cuando el estado no se guarda en cookies de sesión (APIs con JWT) |
| `CorsConfigurationSource` | Bean que define la política CORS por path pattern; registrar en `http.cors()` |
| `allowedOrigins(List.of(...))` | Orígenes permitidos; no usar `"*"` si `allowCredentials(true)` — el browser lo rechaza |
| `allowedHeaders(...)` | Headers que el browser puede enviar en la petición CORS; incluir `Authorization` para JWT |
| `exposedHeaders(...)` | Headers de respuesta que el browser puede leer; por defecto solo los headers "safe" |
| `maxAge(600L)` | Tiempo de caché del preflight OPTIONS en segundos; reduce el número de preflights |
| `X-Frame-Options: DENY` | Previene que la app sea embebida en un iframe (clickjacking) |
| `Content-Security-Policy` | Restringe qué recursos puede cargar la página; mitigación de XSS |
| `Strict-Transport-Security` | Obliga HTTPS por un período; el browser rechaza conexiones HTTP al dominio |

## Buenas y malas prácticas

**Hacer:**
- Configurar `allowedOrigins` con la lista exacta de dominios del frontend en producción; nunca usar `"*"` en producción para endpoints que requieren autenticación.
- Deshabilitar CSRF con `csrf.disable()` en APIs REST que usan JWT en el header `Authorization`; habilitar CSRF solo en aplicaciones que usan cookies de sesión (Spring MVC con login de formulario).
- Configurar CORS en `SecurityFilterChain` en lugar de en `@CrossOrigin` o `WebMvcConfigurer.addCorsMappings`; Spring Security procesa antes y su configuración tiene precedencia — definirla en el mismo lugar evita conflictos.

**Evitar:**
- Usar `allowedOrigins("*")` con `allowCredentials(true)`: los browsers modernos lo rechazan activamente (viola el estándar CORS).
- Configurar un `Content-Security-Policy` extremadamente restrictivo en APIs sin interfaz web (microservicio puro REST): las cabeceras CSP consumen espacio en cada respuesta pero son irrelevantes para consumidores no-browser; aplicarlas solo en aplicaciones con interfaz HTML.
- Olvidar incluir `Authorization` en `allowedHeaders`: si no está incluido, el browser bloqueará la petición preflight y el cliente recibirá un error de CORS aunque el token sea válido y el servidor responda correctamente.

---

← [8.12 Seguridad en microservicios reactivos con WebFlux](sc-security-reactive.md) | [Índice (README.md)](README.md) | [8.14 Configuración avanzada de seguridad y producción](sc-security-production.md) →
