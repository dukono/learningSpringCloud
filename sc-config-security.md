# 1.6 Seguridad del Config Server — autenticación HTTP Basic y cifrado

← [1.5 Refresh de configuración](sc-config-refresh.md) | [Índice](README.md) | [1.7 Operación y alta disponibilidad](sc-config-operacion.md) →

---

## Introducción

El Config Server es una pieza crítica de seguridad en la arquitectura de microservicios: centraliza todas las propiedades de todos los servicios, incluyendo contraseñas de bases de datos, API keys, tokens de autenticación y certificados. Sin protección, cualquier proceso en la red puede hacer `GET http://config-server:8888/order-service/prod` y obtener todas las credenciales de producción. Spring Cloud Config proporciona dos capas de seguridad: autenticación HTTP Basic (quién puede acceder al servidor) y cifrado de propiedades con `{cipher}` (cómo se protegen los datos sensibles en reposo en el repositorio).

> [CONCEPTO] El prefijo `{cipher}` en el valor de una propiedad indica que el valor está cifrado y el servidor lo descifrará antes de entregarlo al cliente. Ejemplo: `db.password={cipher}AQBKi4P6oYJBMs9T...`. El descifrado ocurre en el servidor por defecto.

> [PREREQUISITO] La seguridad se añade sobre un Config Server ya funcional. Completar el setup básico (1.1) antes de securizar.

## Autenticación HTTP Basic

La forma más directa de proteger el Config Server es añadir Spring Security al classpath y configurar un usuario con contraseña. Todos los clientes deberán incluir las credenciales en sus peticiones al servidor.

```
FLUJO DE AUTENTICACIÓN HTTP BASIC
═══════════════════════════════════════════════════════
  Config Client                      Config Server
       │                                    │
       │── GET /order-service/prod ─────────▶│
       │                                    │── 401 Unauthorized (sin credenciales)
       │◀───────────────────────────────────│
       │                                    │
       │── GET /order-service/prod ─────────▶│
       │   Authorization: Basic dXNlcjpwYXNz│
       │                                    │── Valida credenciales con Spring Security
       │◀─── PropertySource[] ──────────────│
═══════════════════════════════════════════════════════
```

## Ejemplo central — Config Server securizado con Basic Auth y cifrado

El siguiente ejemplo muestra el Config Server completo con autenticación HTTP Basic, gestión de claves simétricas y asimétricas, y el flujo de cifrado/descifrado.

**pom.xml — dependencias de seguridad**:

```xml
<dependencies>
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
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

**application.yml del Config Server — Basic Auth y cifrado simétrico**:

```yaml
server:
  port: 8888

spring:
  application:
    name: config-server
  security:
    user:
      name: ${CONFIG_SERVER_USER:configuser}
      password: ${CONFIG_SERVER_PASSWORD:changeme}
      roles: ADMIN
  cloud:
    config:
      server:
        git:
          uri: https://github.com/myorg/config-repo
          default-label: main
          search-paths: '{application}'
        encrypt:
          enabled: true   # Descifrado en servidor (por defecto true)

# Clave simétrica para cifrado (AES)
encrypt:
  key: ${ENCRYPT_KEY:my-32-char-secret-key-for-aes256}
```

**ConfigServerApplication.java**:

```java
package com.example.configserver;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.config.server.EnableConfigServer;

@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}
```

**SecurityConfig.java — permite acceso a /encrypt y /decrypt solo autenticados**:

```java
package com.example.configserver.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health").permitAll()
                .anyRequest().authenticated()
            )
            .httpBasic(httpBasic -> {});
        return http.build();
    }
}
```

**Configuración del cliente con credenciales**:

```yaml
# application.yml del Config Client
spring:
  application:
    name: order-service
  config:
    import: "configserver:http://localhost:8888"
  cloud:
    config:
      username: ${CONFIG_SERVER_USER:configuser}
      password: ${CONFIG_SERVER_PASSWORD:changeme}
```

**Flujo de cifrado: cifrar una contraseña antes de subirla al repositorio**:

```bash
# 1. Cifrar un valor con el endpoint /encrypt del servidor
curl -X POST http://configuser:changeme@localhost:8888/encrypt \
     -d "mi-password-de-base-de-datos"
# Respuesta: AQBKi4P6oYJBMs9TwlGYE7kZ8R...

# 2. Usar el valor cifrado en el repositorio Git (order-service-prod.yml)
# spring:
#   datasource:
#     password: '{cipher}AQBKi4P6oYJBMs9TwlGYE7kZ8R...'
#
# NOTA: Comillas simples son necesarias en YAML para el prefijo {cipher}

# 3. Verificar que el servidor lo descifra correctamente
curl http://configuser:changeme@localhost:8888/order-service/prod/main
# La propiedad spring.datasource.password llegará ya descifrada al cliente
```

## Cifrado asimétrico con keystore JKS

Para entornos de producción, el cifrado asimétrico (par de claves RSA) es más seguro: la clave pública se puede distribuir ampliamente para cifrar, pero solo el servidor (que tiene la clave privada) puede descifrar.

```yaml
# application.yml del Config Server — Cifrado asimétrico
encrypt:
  key-store:
    location: classpath:/config-server-keystore.jks
    password: ${KEYSTORE_PASSWORD}
    alias: configserver
    secret: ${KEY_PASSWORD}
```

**Generar el keystore JKS**:

```bash
keytool -genkeypair \
  -alias configserver \
  -keyalg RSA \
  -keysize 2048 \
  -storetype JKS \
  -keystore config-server-keystore.jks \
  -validity 3650 \
  -storepass mykeystore \
  -keypass mykey
```

> [ADVERTENCIA] Los endpoints `/encrypt` y `/decrypt` del Config Server exponen la capacidad de cifrado/descifrado a cualquiera que tenga las credenciales HTTP Basic. En producción, restringir el acceso a estos endpoints a IPs internas o redes de administración.

## Tabla de propiedades de seguridad y cifrado

| Propiedad | Tipo | Default | Descripción |
|-----------|------|---------|-------------|
| `spring.security.user.name` | String | `user` | Usuario HTTP Basic del servidor |
| `spring.security.user.password` | String | (generado) | Contraseña HTTP Basic del servidor |
| `spring.cloud.config.server.encrypt.enabled` | boolean | `true` | Si el servidor descifra `{cipher}` antes de entregar al cliente |
| `encrypt.key` | String | — | Clave simétrica AES para cifrado |
| `encrypt.key-store.location` | Resource | — | Ubicación del keystore JKS/PKCS12 para cifrado asimétrico |
| `encrypt.key-store.password` | String | — | Contraseña del keystore |
| `encrypt.key-store.alias` | String | — | Alias de la clave en el keystore |
| `spring.cloud.config.username` | String | — | Usuario en el cliente para autenticarse con el servidor |
| `spring.cloud.config.password` | String | — | Contraseña en el cliente |

> [EXAMEN] `spring.cloud.config.server.encrypt.enabled=true` (por defecto) significa que el servidor descifra los valores `{cipher}` ANTES de enviarlos al cliente. Si se pone a `false`, el servidor envía los valores cifrados tal cual y el cliente debe descifrarlos con la misma clave.

## Buenas y malas prácticas

Hacer:
- Almacenar la clave de cifrado y las credenciales del servidor en variables de entorno o en un secrets manager (Vault), nunca en `application.yml`.
- Usar cifrado asimétrico (keystore RSA) en producción; la clave simétrica tiene el riesgo de que quien cifra también puede descifrar.
- Proteger los endpoints `/encrypt` y `/decrypt` con autenticación; por defecto son accesibles a cualquier usuario autenticado.
- Rotar las claves de cifrado periódicamente y re-cifrar los valores en el repositorio con la nueva clave.

Evitar:
- Poner la clave simétrica `encrypt.key` en el fichero `application.yml` que se sube a un repositorio público.
- Usar contraseñas débiles o el usuario `user` por defecto en producción; Spring Security genera una contraseña aleatoria al arranque pero desaparece con cada restart.
- Confundir `{cipher}` (cifrado por Config Server) con el cifrado a nivel de repositorio Git; son mecanismos completamente distintos.
- Deshabilitar CSRF sin protección adicional en entornos expuestos a Internet.

## Verificación y práctica

```bash
# Verificar que el servidor requiere autenticación
curl http://localhost:8888/order-service/prod
# Debe responder 401 Unauthorized

# Acceder con credenciales correctas
curl http://configuser:changeme@localhost:8888/order-service/prod/main

# Cifrar un valor
curl -X POST http://configuser:changeme@localhost:8888/encrypt -d "secret-value"

# Descifrar un valor
curl -X POST http://configuser:changeme@localhost:8888/decrypt \
     -d "AQBKi4P6oYJBMs9T..."
```

**Preguntas estilo examen VMware Spring Professional:**

1. ¿Qué prefijo se usa en el repositorio de configuración para indicar que un valor está cifrado por el Config Server?
2. ¿Qué ocurre cuando `spring.cloud.config.server.encrypt.enabled=false`? ¿Quién descifra en ese caso?
3. ¿Qué endpoints del Config Server permiten cifrar y descifrar valores interactivamente?
4. ¿Cómo se configura el cliente para que use credenciales HTTP Basic al conectar con el Config Server?
5. ¿Cuál es la diferencia de seguridad entre la clave simétrica (`encrypt.key`) y el keystore asimétrico?

---

← [1.5 Refresh de configuración](sc-config-refresh.md) | [Índice](README.md) | [1.7 Operación y alta disponibilidad](sc-config-operacion.md) →
