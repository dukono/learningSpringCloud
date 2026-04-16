# 7.6 Seguridad del endpoint actuator de Spring Cloud Bus

← [7.5 Eventos personalizados de Spring Cloud Bus](sc-bus-eventos-personalizados.md) | [Índice](README.md) | [7.7 Tracing y observabilidad de Spring Cloud Bus](sc-bus-observabilidad.md) →

## Introducción

El endpoint `/actuator/busrefresh` es uno de los más peligrosos de cualquier microservicio: un único `POST` sin autenticación permite a cualquier actor externo forzar la re-instanciación de todos los beans con `@RefreshScope` en todos los nodos del clúster simultáneamente. En producción con cien instancias, esto genera cien peticiones concurrentes al Config Server, puede provocar timeouts en cascada si el Config Server no está dimensionado para ese pico, y puede usarse como vector de denegación de servicio. El control de acceso a este endpoint tiene dos dimensiones: la exposición (qué endpoints son accesibles vía HTTP) y la autorización (quién puede llamarlos). Ambas deben configurarse explícitamente porque Spring Boot 4.x no expone ni protege los endpoints de actuator por defecto en producción.

> [ADVERTENCIA] No exponer `/actuator/busrefresh` en el mismo puerto que la API de negocio si el servicio es accesible desde internet. Configurar `management.server.port` a un puerto separado accesible solo desde la red interna o VPN.

## Representación visual

El diagrama siguiente muestra las tres capas de defensa para el endpoint de Bus y cómo se relacionan entre sí.

```
Internet / CI-CD Pipeline
        │
        │  POST /actuator/busrefresh
        ▼
┌─────────────────────────────────────────────────────┐
│  Capa 1: Exposición                                 │
│  management.endpoints.web.exposure.include          │
│  → solo endpoints declarados son accesibles         │
│  → si busrefresh no está en la lista, retorna 404   │
└──────────────────────┬──────────────────────────────┘
                       │ (endpoint expuesto)
┌──────────────────────▼──────────────────────────────┐
│  Capa 2: Spring Security FilterChain                │
│  .requestMatchers("/actuator/busrefresh")           │
│    .hasRole("BUS_ADMIN")                            │
│  → credenciales inválidas → 401/403                 │
│  → credenciales válidas → continúa                  │
└──────────────────────┬──────────────────────────────┘
                       │ (autorizado)
┌──────────────────────▼──────────────────────────────┐
│  Capa 3: Puerto separado (opcional pero recomendado)│
│  management.server.port=8081                        │
│  → puerto 8081 no expuesto en el load balancer      │
│  → solo accesible desde red interna                 │
└──────────────────────┬──────────────────────────────┘
                       │
               BusAutoConfiguration
               publica RefreshRemoteApplicationEvent
```

## Ejemplo central

El ejemplo siguiente configura la protección completa del endpoint de Bus con Spring Security, incluyendo credenciales para llamadas automatizadas desde webhooks de Git y la separación del puerto de management.

```xml
<!-- pom.xml -->
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-bus-kafka</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>
```

```yaml
# application.yml
spring:
  application:
    name: config-server
  cloud:
    bus:
      enabled: true
      id: ${spring.application.name}:${spring.profiles.active:default}:${random.value}
  kafka:
    bootstrap-servers: kafka:9092
  security:
    user:
      name: busadmin
      password: ${BUS_ADMIN_PASSWORD:changeme-in-production}
      roles: BUS_ADMIN

# Puerto separado para actuator: no expuesto en el load balancer externo
management:
  server:
    port: 8081
  endpoints:
    web:
      exposure:
        include: busrefresh, busenv, health, info
  endpoint:
    health:
      show-details: when-authorized
```

```java
// src/main/java/com/example/config/security/ActuatorSecurityConfig.java
package com.example.config.security;

import org.springframework.boot.actuate.autoconfigure.security.servlet.EndpointRequest;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableWebSecurity
public class ActuatorSecurityConfig {

    // SecurityFilterChain para los endpoints de actuator
    // Se aplica al puerto de management (8081 en este ejemplo)
    @Bean
    public SecurityFilterChain actuatorSecurityFilterChain(HttpSecurity http)
            throws Exception {

        http
            .securityMatcher(EndpointRequest.toAnyEndpoint())
            .authorizeHttpRequests(authorize -> authorize
                // busrefresh y busenv requieren rol BUS_ADMIN
                .requestMatchers(
                    EndpointRequest.to("busrefresh", "busenv")
                ).hasRole("BUS_ADMIN")
                // health e info son públicos
                .requestMatchers(
                    EndpointRequest.to("health", "info")
                ).permitAll()
                // cualquier otro endpoint de actuator requiere autenticación
                .anyRequest().authenticated()
            )
            // Autenticación básica para llamadas automatizadas desde webhook
            .httpBasic(basic -> {})
            // Sin sesión: cada petición lleva sus propias credenciales
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )
            // CSRF deshabilitado para endpoints de API (REST)
            .csrf(csrf -> csrf.disable());

        return http.build();
    }

    // SecurityFilterChain para el tráfico de negocio (puerto 8080)
    @Bean
    public SecurityFilterChain appSecurityFilterChain(HttpSecurity http)
            throws Exception {
        http
            .authorizeHttpRequests(authorize -> authorize
                .requestMatchers("/api/**").authenticated()
                .anyRequest().permitAll()
            )
            .httpBasic(basic -> {});

        return http.build();
    }
}
```

La llamada desde el webhook de GitHub queda así con autenticación básica:

```bash
# Webhook de GitHub configurado con:
# Payload URL: http://config-server:8081/actuator/busrefresh
# Content-Type: application/json
# Secret: (configurar HMAC si se usa GitHub App o Actions)
# Para tests manuales:
curl -X POST http://config-server:8081/actuator/busrefresh \
  -u busadmin:changeme-in-production \
  -H "Content-Type: application/json"

# Refresh selectivo autenticado:
curl -X POST "http://config-server:8081/actuator/busrefresh/pricing-service:**" \
  -u busadmin:changeme-in-production
```

## Tabla de elementos clave

La tabla siguiente recoge las propiedades de seguridad relevantes para el endpoint de Bus.

| Parámetro | Tipo | Default | Descripción |
|---|---|---|---|
| `management.endpoints.web.exposure.include` | String list | `health,info` | Endpoints expuestos vía HTTP. Debe incluir `busrefresh` explícitamente. |
| `management.endpoints.web.exposure.exclude` | String list | — | Endpoints explícitamente excluidos. Tiene precedencia sobre `include`. |
| `management.server.port` | Int | mismo que `server.port` | Puerto del servidor de management. Separar de negocio en producción. |
| `management.server.address` | String | — | Dirección de escucha de management. Puede restringirse a la red interna. |
| `spring.security.user.name` | String | `user` | Usuario de autenticación básica para actuator. |
| `spring.security.user.password` | String | (generado) | Password del usuario de actuator. Fijar en producción con variable de entorno. |
| `spring.security.user.roles` | String list | `USER` | Roles del usuario de actuator. Debe incluir `BUS_ADMIN` si se usa ese rol. |

## Buenas y malas prácticas

**Hacer:**

- Configurar `management.server.port` diferente al puerto de la aplicación y no exponerlo en el balanceador de carga externo. En Kubernetes, el Service de management debe ser de tipo `ClusterIP` (no `LoadBalancer` ni `NodePort`) para que solo sea accesible desde dentro del clúster.
- Usar variables de entorno o secretos de Kubernetes para la contraseña de actuator. Hardcodear credenciales en `application.yml` expone el acceso al endpoint en el repositorio Git del Config Server, que es precisamente el sistema que el endpoint controla.
- Validar el secreto HMAC del webhook de GitHub antes de procesar la petición. GitHub envía un header `X-Hub-Signature-256` que permite verificar que la petición viene realmente de GitHub y no de un atacante que conoce la URL del endpoint.

**Evitar:**

- No exponer `/actuator/busrefresh` sin autenticación incluso en entornos internos. Un atacante con acceso a la red interna puede usarlo para generar denegación de servicio en el Config Server. La autenticación básica con credenciales fuertes es el mínimo aceptable.
- No usar las mismas credenciales del endpoint de negocio para el endpoint de actuator. La separación de credenciales permite rotar las del bus sin afectar a los usuarios de la API de negocio, y viceversa.
- No confiar en el firewall como única capa de seguridad. El firewall puede tener reglas incorrectas, y un servicio comprometido dentro del perímetro puede llamar al endpoint sin restricciones. Spring Security añade una capa de defensa en profundidad que no depende de la configuración de red.
- No deshabilitar CSRF para toda la aplicación si los formularios web usan la misma SecurityFilterChain que los endpoints de actuator. La configuración mostrada en el ejemplo usa `securityMatcher` para aplicar la deshabilitación de CSRF solo a los endpoints de actuator, manteniendo la protección CSRF en el resto de la aplicación.

---

← [7.5 Eventos personalizados de Spring Cloud Bus](sc-bus-eventos-personalizados.md) | [Índice](README.md) | [7.7 Tracing y observabilidad de Spring Cloud Bus](sc-bus-observabilidad.md) →
