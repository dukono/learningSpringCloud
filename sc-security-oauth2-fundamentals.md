# 8.1 Fundamentos OAuth2 y JWT para microservicios

← [7.9 Testing de Spring Cloud Bus](sc-bus-testing.md) | [Índice (README.md)](README.md) | [8.2 Resource Server JWT — configuración y personalización](sc-security-resource-server-jwt.md) →

---

OAuth2 y JWT son los estándares de seguridad que dominan la autenticación y autorización en arquitecturas de microservicios: OAuth2 define el flujo de delegación de acceso (quién emite los tokens, quién los valida, quién los usa), y JWT es el formato más extendido para transportar las claims de identidad y permisos dentro del token. Spring Security 6.x integra ambos estándares a través de dos abstracciones centrales: `Resource Server` (el microservicio que protege sus endpoints validando el JWT) y `OAuth2 Client` (el microservicio que necesita obtener y usar tokens para llamar a otros servicios). Comprender la separación de roles entre Authorization Server, Resource Server y Client es el prerequisito para cualquier configuración de seguridad en microservicios Spring Cloud.

> [PREREQUISITO] No requiere código en este punto — es una sección conceptual. Los ficheros siguientes (8.2 en adelante) implementan la configuración concreta. Requiere conocimiento básico de HTTP y del protocolo HTTP Bearer.

## Roles en OAuth2 para microservicios

La arquitectura OAuth2 distribuye las responsabilidades entre tres componentes que se comunican mediante tokens.

```
┌─────────────────────────────────────────────────────────────────┐
│                     Arquitectura OAuth2                          │
│                                                                  │
│  ┌─────────────┐   (1) solicita token          ┌─────────────┐  │
│  │   Client    │ ──────────────────────────→   │  Auth       │  │
│  │  (frontend  │ ←──────────────────────────   │  Server     │  │
│  │  / svc A)  │   (2) devuelve JWT             │  (Keycloak/ │  │
│  └─────────────┘                               │  SAS/etc)   │  │
│         │                                      └─────────────┘  │
│         │ (3) petición + Bearer JWT                              │
│         ↓                                                        │
│  ┌─────────────┐   (4) valida JWT localmente   ┌─────────────┐  │
│  │  Resource   │ ──────────────────────────→   │  JWK Set    │  │
│  │  Server     │ ←──────────────────────────   │  Endpoint   │  │
│  │  (svc B)   │   (5) clave pública OK         │  del AS     │  │
│  └─────────────┘                               └─────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

## Ejemplo central: estructura de un JWT y flujo de validación

El JWT contiene tres partes separadas por puntos: header (algoritmo), payload (claims) y signature. Spring Security valida la firma usando la clave pública del Authorization Server.

### Estructura de un JWT típico para microservicios

```json
// Header (Base64url decodificado)
{
  "alg": "RS256",
  "typ": "JWT",
  "kid": "rsa-key-2025"
}

// Payload (Base64url decodificado)
{
  "iss": "https://auth.example.com",
  "sub": "user-123",
  "aud": ["order-service", "product-service"],
  "exp": 1735689600,
  "iat": 1735686000,
  "scope": "orders:read orders:write",
  "roles": ["ROLE_USER", "ROLE_ADMIN"],
  "preferred_username": "alice@example.com"
}

// Signature: RS256(base64url(header) + "." + base64url(payload), private_key)
```

### Dependencias Maven para un Resource Server

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
    <!-- Resource Server: valida JWT en cada petición -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
    </dependency>
    <!-- Incluye Spring Security 6.x -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
</dependencies>
```

### Flujo de validación JWT en Spring Security 6.x

```java
// application.yml — configuración mínima de un Resource Server JWT
// Spring Security descarga automáticamente la JWK Set del issuer
```

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          # Spring Security recupera el JWK Set de esta URL para validar firmas
          issuer-uri: https://auth.example.com
          # Alternativa sin OIDC discovery: especificar la URL del JWK Set directamente
          # jwk-set-uri: https://auth.example.com/oauth2/jwks
```

```java
package com.example.security;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            // Deshabilitar CSRF: los APIs REST con JWT no lo necesitan
            .csrf(csrf -> csrf.disable())
            // Configurar como Resource Server JWT
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> {})  // configuración mínima: valida firma y expiración
            )
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health").permitAll()
                .anyRequest().authenticated()
            );

        return http.build();
    }
}
```

> [CONCEPTO] `issuer-uri` hace que Spring Security ejecute OIDC Discovery (GET `{issuer}/.well-known/openid-configuration`) al arrancar para obtener el JWK Set URI. Si el Authorization Server no soporta OIDC Discovery, usar `jwk-set-uri` directamente. La validación de firma es local (no requiere llamada al AS en cada petición), lo que hace que los JWTs sean stateless y escalables.

## Tabla de elementos clave

| Concepto | Descripción |
|---|---|
| Authorization Server (AS) | Emite tokens JWT tras autenticar al usuario/cliente; ejemplos: Spring Authorization Server, Keycloak, Auth0, Azure AD |
| Resource Server | Microservicio que valida el JWT en cada petición HTTP; Spring Boot configura uno con `spring-boot-starter-oauth2-resource-server` |
| OAuth2 Client | Microservicio o frontend que obtiene tokens y los adjunta en llamadas a otros servicios |
| `iss` (issuer) | Claim del JWT que identifica quién emitió el token; Spring Security valida que coincida con `issuer-uri` |
| `sub` (subject) | Claim del JWT que identifica al usuario o cliente; accesible como `authentication.getName()` en el código |
| `aud` (audience) | Claim del JWT que identifica a qué servicios va dirigido el token; validar en cada Resource Server evita que un token de otro servicio sea aceptado |
| `scope` | Claim OAuth2 que lista los permisos delegados; en Spring Security se convierte en authorities con prefijo `SCOPE_` |
| JWK Set | Conjunto de claves públicas del AS; Spring Security lo descarga al iniciar para validar firmas RS256/ES256 |
| Bearer Token | El token JWT se envía en el header `Authorization: Bearer <token>` |

## Buenas y malas prácticas

**Hacer:**
- Validar el claim `aud` (audience) en cada Resource Server para rechazar tokens emitidos para otros servicios; sin esta validación, un token robado de otro servicio puede ser reutilizado.
- Usar algoritmos asimétricos (RS256, ES256) en lugar de HMAC (HS256): con HMAC, todos los servicios necesitan la clave secreta (riesgo de exposición); con RS256, solo el AS tiene la clave privada y los Resource Servers solo necesitan la clave pública.
- Configurar un tiempo de expiración corto (15-60 minutos) para access tokens junto con refresh tokens de larga duración; reduce la ventana de exposición ante robo de token.

**Evitar:**
- Almacenar tokens JWT en `localStorage` en aplicaciones SPA: son vulnerables a XSS; usar `httpOnly` cookies o Session Storage con estricto CSP.
- Poner información sensible (contraseñas, PII completo) en las claims del JWT: el payload es solo Base64url, no cifrado — cualquiera con el token puede leer el contenido.
- Aceptar tokens sin validar la firma: un JWT sin validación criptográfica no garantiza autenticidad — cualquiera puede construir un payload arbitrario.

---

← [7.9 Testing de Spring Cloud Bus](sc-bus-testing.md) | [Índice (README.md)](README.md) | [8.2 Resource Server JWT — configuración y personalización](sc-security-resource-server-jwt.md) →
