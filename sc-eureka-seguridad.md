# 2.9 Seguridad del Eureka Server

← [2.8 Alta disponibilidad: Eureka Server en cluster](sc-eureka-cluster.md) | [Índice (README.md)](README.md) | [2.10 Integración con Spring Cloud Config →](sc-eureka-integracion-config.md)

---

Sin seguridad, el Eureka Server expone la topología completa del sistema —nombres de servicios, IPs, puertos y metadata— a cualquier proceso con acceso a la red interna, y permite que cualquiera dé de baja instancias o inyecte entradas falsas en el registro. La forma estándar de protegerlo en Spring Cloud es HTTP Basic Authentication configurada con Spring Security. Esta solución cubre el registro y el dashboard sin introducir la complejidad de OAuth2 o JWT, que pertenecen al módulo Spring Cloud Security y van más allá del alcance de Eureka.

> [ADVERTENCIA] Esta sección cubre exclusivamente HTTP Basic auth para el Eureka Server. OAuth2, JWT y autorización por roles en el ecosistema de microservicios son responsabilidad del módulo Spring Cloud Security (dominio-sc-security.json) y no se tratan aquí.

## Diagrama: flujo de autenticación del cliente Eureka

El siguiente diagrama muestra cómo el cliente incluye las credenciales en la URL de registro y cómo el servidor Spring Security las valida antes de procesar el heartbeat o la consulta.

```
Microservicio (Eureka Client)
         │
         │  PUT http://admin:secreto@eureka-server:8761/eureka/apps/SERVICIO
         │  Authorization: Basic YWRtaW46c2VjcmV0bw==
         ▼
Spring Security (filtro en Eureka Server)
         │
         ├── Verificar cabecera Authorization: Basic
         │   ├── Usuario/password correcto → continuar
         │   └── Incorrecto → 401 Unauthorized
         │
         ▼ (autenticado)
CSRF Filter
         │
         ├── Petición sobre /eureka/** ?
         │   ├── SÍ → ignorar CSRF (PUT/DELETE sin token CSRF)
         │   └── NO → validar token CSRF normalmente
         │
         ▼
Eureka Server procesa la petición
(registro, heartbeat, consulta, baja de instancia)
```

## Implementación: Eureka Server con HTTP Basic Authentication

El siguiente ejemplo es un Eureka Server completo con Spring Security configurado para HTTP Basic, incluyendo la desactivación correcta de CSRF para los endpoints de Eureka.

**pom.xml (añadir Spring Security al Eureka Server)**

```xml
<dependencies>
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
  <!-- Spring Security para HTTP Basic -->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
  </dependency>
</dependencies>
```

**application.yml (credenciales del servidor)**

```yaml
server:
  port: 8761

spring:
  application:
    name: eureka-server
  security:
    user:
      # Credenciales para HTTP Basic. En producción, usar variables de entorno
      # o Spring Cloud Config para no hardcodear en el fichero.
      name: ${EUREKA_USER:admin}
      password: ${EUREKA_PASSWORD:secreto}

eureka:
  instance:
    hostname: localhost
  client:
    register-with-eureka: false
    fetch-registry: false
    service-url:
      defaultZone: http://${spring.security.user.name}:${spring.security.user.password}@${eureka.instance.hostname}:${server.port}/eureka/
```

**SecurityConfig.java — configuración de Spring Security para Eureka Server**

```java
package com.ejemplo.eurekaserver.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.Customizer;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            // CSRF debe desactivarse para los endpoints de Eureka porque los clientes
            // Eureka envían PUT y DELETE sin token CSRF. Activar CSRF para /eureka/**
            // bloquea todos los heartbeats y registros de clientes.
            .csrf(csrf -> csrf
                .ignoringRequestMatchers("/eureka/**")
            )
            // Todos los endpoints requieren autenticación
            .authorizeHttpRequests(auth -> auth
                .anyRequest().authenticated()
            )
            // HTTP Basic como mecanismo de autenticación
            .httpBasic(Customizer.withDefaults());

        return http.build();
    }
}
```

**application.yml del cliente — incluir credenciales en la URL**

```yaml
spring:
  application:
    name: servicio-pedidos

eureka:
  client:
    service-url:
      # Formato: http://usuario:contraseña@host:puerto/eureka/
      # Las credenciales se envían como HTTP Basic en cada heartbeat y registro.
      defaultZone: http://admin:secreto@eureka-server:8761/eureka/

  instance:
    prefer-ip-address: true
```

> [ADVERTENCIA] Las credenciales incluidas en `defaultZone` con formato `http://user:pass@host/eureka/` se transmiten en Base64 (HTTP Basic), no cifradas. En entornos de producción, el canal debe estar protegido con TLS (HTTPS) para evitar que las credenciales sean interceptadas en la red interna. Sin TLS, HTTP Basic ofrece autenticación pero no confidencialidad.

**Uso de variables de entorno para credenciales en Docker/Kubernetes**

```yaml
# docker-compose.yml (extracto)
services:
  eureka-server:
    image: eureka-server:latest
    environment:
      EUREKA_USER: admin
      EUREKA_PASSWORD: ${EUREKA_SECRET}  # Variable del entorno del host
    ports:
      - "8761:8761"

  servicio-pedidos:
    image: servicio-pedidos:latest
    environment:
      # El cliente lee la URL de la variable de entorno
      EUREKA_CLIENT_SERVICEURL_DEFAULTZONE: http://admin:${EUREKA_SECRET}@eureka-server:8761/eureka/
```

## Tabla de propiedades de seguridad

La siguiente tabla recoge los parámetros relevantes para la seguridad del Eureka Server.

| Parámetro | Tipo | Default | Descripción |
|---|---|---|---|
| `spring.security.user.name` | String | `user` | Usuario para HTTP Basic (autoconfiguración de Spring Security) |
| `spring.security.user.password` | String | (generado) | Contraseña para HTTP Basic; por defecto se genera y se imprime en el log |
| `eureka.client.service-url.defaultZone` | String | — | Incluir `http://user:pass@host:port/eureka/` para autenticación del cliente |

## Buenas y malas prácticas

Hacer:
- Externalizar las credenciales del Eureka Server con variables de entorno o Spring Cloud Config: las credenciales hardcodeadas en `application.yml` quedan expuestas en el repositorio de código y en los artefactos Docker publicados.
- Desactivar CSRF exclusivamente para `/eureka/**` y no para toda la aplicación: el dashboard web sí puede beneficiarse de la protección CSRF para sus propios formularios si se extiende con funcionalidad adicional.
- Habilitar HTTPS en el Eureka Server en producción: HTTP Basic sobre HTTP transmite credenciales en Base64 legible por cualquiera que capture el tráfico de red interno. TLS convierte esta autenticación en segura.

Evitar:
- Mezclar la configuración de seguridad de Eureka con la de OAuth2/JWT: los clientes Eureka solo soportan HTTP Basic nativo en la URL `defaultZone`; intentar usar tokens OAuth2 para el heartbeat requiere implementación custom fuera del alcance estándar del módulo.
- Aplicar `csrf.disable()` a toda la aplicación para resolver el problema del Eureka Server: la solución correcta es `csrf.ignoringRequestMatchers("/eureka/**")`, que mantiene CSRF activo para el resto de la aplicación.

---

← [2.8 Alta disponibilidad: Eureka Server en cluster](sc-eureka-cluster.md) | [Índice (README.md)](README.md) | [2.10 Integración con Spring Cloud Config →](sc-eureka-integracion-config.md)
