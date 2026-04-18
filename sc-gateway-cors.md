# 5.8 CORS en Spring Cloud Gateway

← [5.7 Integración con Service Discovery](sc-gateway-discovery-integration.md) | [Índice](README.md) | [5.9 Seguridad y OAuth2](sc-gateway-seguridad-oauth2.md) →

---

## Introducción

CORS (Cross-Origin Resource Sharing) es el mecanismo del navegador para controlar qué orígenes pueden hacer peticiones HTTP a un dominio diferente. En una arquitectura de microservicios con API Gateway, la configuración CORS debe centralizarse en el Gateway para evitar duplicación y conflictos con los microservicios downstream. Spring Cloud Gateway ofrece dos niveles de configuración CORS: una política global (`globalCors`) aplicada a todas las rutas y configuración por ruta para casos especiales. Ignorar la configuración CORS en el Gateway —o configurarla incorrectamente en paralelo con los microservicios— es una fuente habitual de errores difíciles de diagnosticar.

> [ADVERTENCIA] Si configuras CORS tanto en el Gateway como en los microservicios downstream, los navegadores recibirán headers CORS duplicados y rechazarán las peticiones. La regla es: configura CORS en el Gateway y elimínalo de los servicios downstream (o usa `DedupeResponseHeader`).

## Cómo funciona CORS en Gateway (WebFlux)

Spring Cloud Gateway está construido sobre WebFlux, que implementa CORS mediante `CorsWebFilter` integrado en el pipeline reactivo. Cuando el Gateway recibe una petición con header `Origin`, el motor CORS compara el origen con la política configurada. Para peticiones preflight (`OPTIONS`), el Gateway responde directamente sin llegar al upstream.

> [CONCEPTO] El **preflight request** es una petición `OPTIONS` que el navegador envía automáticamente antes de peticiones cross-origin con métodos no simples (PUT, DELETE, PATCH) o headers personalizados. El Gateway debe responder a estos preflight con los headers CORS correctos para que el navegador permita la petición real.

La configuración global CORS en Gateway se expresa bajo `spring.cloud.gateway.globalcors.corsConfigurations` como un mapa donde la clave es el patrón de path y el valor es la configuración de CORS para ese patrón.

## Ejemplo central

El siguiente ejemplo muestra la configuración CORS completa: política global, política específica por path, y gestión de headers duplicados:

```yaml
# application.yml — Configuración CORS completa en Spring Cloud Gateway
spring:
  cloud:
    gateway:
      # ===== CORS GLOBAL =====
      globalcors:
        cors-configurations:
          # Política aplicada a TODOS los paths
          '[/**]':
            # Orígenes permitidos (usar lista explícita en producción, no '*')
            allowed-origins:
              - "https://app.example.com"
              - "https://admin.example.com"
            # O usar patrones de origen (Spring 5.3+)
            # allowed-origin-patterns:
            #   - "https://*.example.com"
            allowed-methods:
              - GET
              - POST
              - PUT
              - DELETE
              - PATCH
              - OPTIONS
            allowed-headers:
              - Authorization
              - Content-Type
              - X-Requested-With
              - X-Correlation-Id
            # Headers que el navegador puede leer desde JavaScript
            exposed-headers:
              - X-Total-Count
              - X-Correlation-Id
              - Location
            # Permite envío de cookies cross-origin
            allow-credentials: true
            # Tiempo en segundos que el navegador cachea el preflight
            max-age: 3600

          # Política más permisiva solo para /public/**
          '[/public/**]':
            allowed-origins: "*"
            allowed-methods:
              - GET
              - HEAD
            allow-credentials: false
            max-age: 86400

      routes:
        - id: api-service
          uri: lb://api-service
          predicates:
            - Path=/api/**
          filters:
            - StripPrefix=1
            # Evitar headers CORS duplicados si el upstream también los añade
            - DedupeResponseHeader=Access-Control-Allow-Credentials Access-Control-Allow-Origin

        - id: public-api
          uri: lb://public-service
          predicates:
            - Path=/public/**
          filters:
            - StripPrefix=1
```

```java
package com.example.gateway.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.cors.CorsConfiguration;
import org.springframework.web.cors.reactive.CorsWebFilter;
import org.springframework.web.cors.reactive.UrlBasedCorsConfigurationSource;

import java.util.Arrays;
import java.util.List;

/**
 * Configuración CORS programática (alternativa al YAML).
 * Útil cuando la política CORS necesita lógica dinámica o valores de propiedades.
 */
@Configuration
public class CorsGlobalConfig {

    /**
     * CorsWebFilter se registra como bean y aplica a todas las rutas del Gateway.
     * NOTA: Si usas spring.cloud.gateway.globalcors en YAML, NO declares este bean
     * para evitar conflictos. Usa uno u otro enfoque, no ambos.
     */
    @Bean
    public CorsWebFilter corsWebFilter() {
        CorsConfiguration corsConfig = new CorsConfiguration();

        // Orígenes permitidos
        corsConfig.setAllowedOrigins(Arrays.asList(
            "https://app.example.com",
            "https://admin.example.com"
        ));

        // Métodos HTTP permitidos
        corsConfig.setAllowedMethods(Arrays.asList(
            "GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS"
        ));

        // Headers permitidos en la petición
        corsConfig.setAllowedHeaders(Arrays.asList(
            "Authorization",
            "Content-Type",
            "X-Requested-With",
            "X-Correlation-Id"
        ));

        // Headers expuestos al JavaScript del navegador
        corsConfig.setExposedHeaders(Arrays.asList(
            "X-Total-Count",
            "X-Correlation-Id",
            "Location"
        ));

        // Permite cookies/credentials cross-origin
        corsConfig.setAllowCredentials(true);

        // Cacheo del preflight en el navegador (segundos)
        corsConfig.setMaxAge(3600L);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        // Aplicar a todos los paths
        source.registerCorsConfiguration("/**", corsConfig);

        return new CorsWebFilter(source);
    }
}
```

## Tabla de propiedades CORS

La siguiente tabla resume las propiedades de una `CorsConfiguration` y su significado:

| Propiedad YAML | Clase Java | Descripción |
|---|---|---|
| `allowed-origins` | `setAllowedOrigins(List)` | Orígenes exactos permitidos |
| `allowed-origin-patterns` | `setAllowedOriginPatterns(List)` | Patrones con `*` (p. ej. `https://*.example.com`) |
| `allowed-methods` | `setAllowedMethods(List)` | Métodos HTTP permitidos (`GET`, `POST`, etc.) |
| `allowed-headers` | `setAllowedHeaders(List)` | Headers de petición que el navegador puede enviar |
| `exposed-headers` | `setExposedHeaders(List)` | Headers de respuesta accesibles desde JavaScript |
| `allow-credentials` | `setAllowCredentials(Boolean)` | Permite cookies/Authorization header cross-origin |
| `max-age` | `setMaxAge(Long)` | Segundos de caché del preflight en el navegador |

## Incompatibilidad: allow-credentials + wildcard origin

El estándar CORS prohíbe combinar `allow-credentials: true` con `allowed-origins: "*"`. Si se activan ambos simultáneamente, el navegador rechaza la respuesta con un error de CORS. Esta es la causa más frecuente de errores CORS en producción.

> [ADVERTENCIA] Si necesitas `allow-credentials: true`, debes especificar los orígenes exactos en `allowed-origins` (lista explícita) o usar `allowed-origin-patterns` con un patrón que no sea `*`. Nunca uses `allowed-origins: "*"` con `allow-credentials: true`.

Para APIs públicas sin credenciales, `allowed-origins: "*"` es seguro y conveniente.

## DedupeResponseHeader y CORS

Cuando el Gateway tiene CORS configurado globalmente y los microservicios downstream también envían headers CORS en su respuesta, el cliente recibe headers duplicados como `Access-Control-Allow-Origin: https://app.example.com, https://app.example.com`. Esto rompe la validación CORS del navegador.

El filtro `DedupeResponseHeader` resuelve este problema eliminando los duplicados del header especificado. Las estrategias disponibles son `RETAIN_FIRST` (por defecto), `RETAIN_LAST` y `RETAIN_UNIQUE`.

## Buenas y malas prácticas

**Buenas prácticas:**
- Centralizar toda la configuración CORS en el Gateway y eliminarla de los microservicios downstream.
- Usar `allowed-origin-patterns: "https://*.example.com"` en lugar de listar todos los subdominios manualmente.
- Añadir `DedupeResponseHeader=Access-Control-Allow-Credentials Access-Control-Allow-Origin` a las rutas cuyo upstream también configura CORS.
- Usar listas explícitas de orígenes en producción; reservar `*` solo para APIs públicas sin credenciales.

**Malas prácticas:**
- Combinar `allow-credentials: true` con `allowed-origins: "*"`: el navegador rechazará todas las peticiones con credenciales.
- Configurar CORS en el Gateway y en el downstream simultáneamente sin `DedupeResponseHeader`: causa headers duplicados.
- Usar `max-age: 0` en producción: fuerza un preflight en cada petición, aumentando la latencia.
- Omitir `OPTIONS` en `allowed-methods`: las peticiones preflight fallan con 405 Method Not Allowed.

## Verificación y práctica

1. ¿Cómo configuras CORS globalmente en Spring Cloud Gateway para permitir `GET` y `POST` desde `https://app.example.com`? ¿Qué propiedad usarías?

2. ¿Por qué no puedes combinar `allow-credentials: true` con `allowed-origins: "*"`? ¿Cuál es la alternativa correcta?

3. ¿Qué es una petición preflight y qué método HTTP usa? ¿Por qué el Gateway debe incluir `OPTIONS` en `allowed-methods`?

4. Un microservicio downstream devuelve `Access-Control-Allow-Origin: https://app.example.com` en su respuesta, y el Gateway también añade el mismo header. ¿Qué problema causa esto y cómo lo resuelves?

5. ¿Cuál es la diferencia entre `allowed-headers` y `exposed-headers`? ¿Qué ocurre con un header de respuesta no listado en `exposed-headers`?

---

← [5.7 Integración con Service Discovery](sc-gateway-discovery-integration.md) | [Índice](README.md) | [5.9 Seguridad y OAuth2](sc-gateway-seguridad-oauth2.md) →
