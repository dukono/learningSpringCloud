# 6.6.1 Flujo de validación JWT centralizada y arquitectura de seguridad

← [6.5.3 Clúster Redis e in-memory](./06-23-gateway-rate-limiter-variantes.md) | [Índice](./README.md) | [6.6.2 JwtAuthFilter con JJWT →](./06-25-gateway-jwt-filter.md)

---

Sin un Gateway que centralice la validación de tokens, cada microservicio debe implementar su propia lógica de autenticación: parsear el JWT, verificar la firma, comprobar la expiración, extraer los claims. Esta duplicación introduce inconsistencias (distintas versiones de la librería JWT, distintos algoritmos de firma), dificulta la rotación de claves (habría que actualizar decenas de servicios simultáneamente) y expone la clave pública en todos los servicios. El Gateway como punto centralizado de validación resuelve estos problemas: valida el token una sola vez, extrae la identidad del usuario y la propaga como headers internos de confianza al resto de la plataforma.

## Diagrama: flujo completo de validación JWT centralizada

El diagrama muestra el recorrido de una petición desde el cliente externo hasta los microservicios internos: el `JwtAuthFilter` intercepta la petición en el Gateway, verifica el token y —si es válido— extrae los claims y los inyecta como headers internos antes de enrutar la petición. Los microservicios reciben esos headers ya validados y nunca procesan el JWT directamente.

```
Cliente externo
  │
  │  Authorization: Bearer eyJhbGc...
  │
  ▼
┌─────────────────────────────────────────────────────────┐
│                  Spring Cloud Gateway                   │
│                                                         │
│  JwtAuthFilter (GlobalFilter, order=-1)                 │
│  1. Extrae Bearer token del header Authorization        │
│  2. Verifica firma con clave pública/secreta            │
│  3. Comprueba expiración (claim `exp`)                  │
│  4. Extrae claims: sub, roles, tenant                   │
│  5. Propaga como headers internos                       │
│                                                         │
│  ¿Token válido? ── NO ──► 401 Unauthorized              │
│       │                                                 │
│       SÍ                                                │
│       │                                                 │
│  Petición mutada con headers internos:                  │
│    X-User-Id: "usr-12345"                               │
│    X-User-Roles: "ROLE_USER,ROLE_ADMIN"                 │
│    X-Tenant-Id: "tenant-acme"                           │
└──────────────────────────┬──────────────────────────────┘
                           │
           ┌───────────────┼───────────────┐
           ▼               ▼               ▼
    Microservicio A  Microservicio B  Microservicio C
    (lee X-User-Id)  (lee X-User-Roles) (lee X-Tenant-Id)
    NO valida el JWT NO valida el JWT  NO valida el JWT
```

## Arquitectura de seguridad en capas

La arquitectura de seguridad en un sistema con API Gateway se divide en tres capas con responsabilidades distintas:

**Capa 1 — Gateway (validación y extracción)**: verifica la firma del JWT, comprueba la expiración, extrae los claims relevantes y los convierte en headers HTTP internos. El token original puede o no reenviarse al microservicio (ver `TokenRelay` en 6.6.4).

**Capa 2 — Red interna**: los microservicios solo son accesibles desde la red interna del cluster. Cualquier petición que llega a un microservicio fue originada desde el Gateway o desde otro microservicio de confianza. Los headers `X-User-Id`, `X-User-Roles`, `X-Tenant-Id` son de confianza porque solo el Gateway puede inyectarlos (los clientes externos no pueden acceder directamente a los microservicios).

**Capa 3 — Microservicio (autorización)**: el microservicio lee los headers propagados para tomar decisiones de autorización (¿tiene el usuario el rol necesario? ¿pertenece al tenant correcto?), pero no vuelve a validar el JWT.

> [ADVERTENCIA] La seguridad de la capa 2 depende de que los microservicios no sean accesibles desde el exterior. Si un microservicio expone un puerto público directamente, un atacante puede añadir `X-User-Id` y `X-User-Roles` arbitrarios en la petición y suplantar cualquier identidad. En Kubernetes, esto se implementa con `NetworkPolicy` que restringe el acceso a los microservicios solo desde el namespace del Gateway.

## Rutas públicas vs rutas protegidas

No todas las rutas requieren autenticación. El `JwtAuthFilter` necesita distinguir entre ambas:

```
Rutas públicas (sin JWT requerido):
  /actuator/health
  /actuator/info
  /api/public/**
  /api/auth/login
  /api/auth/register

Rutas protegidas (JWT requerido):
  /api/productos/**
  /api/pedidos/**
  /api/usuarios/**
  /api/admin/**
```

La lista de rutas públicas se gestiona en configuración para permitir añadir o eliminar rutas sin recompilar:

```yaml
gateway:
  security:
    public-paths:
      - /actuator/health
      - /actuator/info
      - /api/public/**
      - /api/auth/**
```

## Propagación de identidad como headers internos

La identidad del usuario autenticado se propaga como headers HTTP que los microservicios pueden leer directamente sin parsear el JWT:

| Header interno | Origen en el JWT | Ejemplo de valor |
|---|---|---|
| `X-User-Id` | Claim `sub` (subject) | `usr-12345` |
| `X-User-Roles` | Claim `roles` o `authorities` | `ROLE_USER,ROLE_ADMIN` |
| `X-Tenant-Id` | Claim `tenant_id` (custom) | `tenant-acme` |
| `X-User-Email` | Claim `email` | `user@acme.com` |

> [CONCEPTO] Los headers propagados son cadenas de texto HTTP, no objetos tipados. Los microservicios que necesiten los roles como lista deben hacer `split(",")` sobre el valor de `X-User-Roles`. Esta limitación es intencional: los headers HTTP son el contrato entre el Gateway y los microservicios, y deben ser simples y universales.

Los headers externos del cliente que coincidan con los nombres de los headers internos deben eliminarse antes de propagar los del JWT, para evitar que un cliente malintencionado inyecte una identidad falsa:

```java
// En el JwtAuthFilter, antes de propagar:
ServerHttpRequest request = exchange.getRequest().mutate()
    .headers(headers -> {
        headers.remove("X-User-Id");      // eliminar cualquier valor externo
        headers.remove("X-User-Roles");
        headers.remove("X-Tenant-Id");
        headers.add("X-User-Id", userId);  // añadir el valor validado del JWT
        headers.add("X-User-Roles", roles);
        headers.add("X-Tenant-Id", tenantId);
    })
    .build();
```

## Opciones de implementación

Spring Cloud Gateway ofrece dos enfoques para la validación de JWT, con diferencias importantes en cuándo elegir cada uno:

### Filtro personalizado con JJWT

Adecuado cuando el formato del JWT es propietario, cuando se necesita lógica personalizada de extracción de claims, o cuando no se integra con un Authorization Server compatible con OpenID Connect.

### Spring Security OAuth2 Resource Server

Adecuado cuando se tiene un Authorization Server estándar (Keycloak, Auth0, Okta, Spring Authorization Server) que expone un endpoint JWKS. La integración es declarativa (solo YAML), gestiona la rotación de claves pública automáticamente y soporta el estándar OIDC completo.

## Tabla comparativa: filtro JJWT vs Spring Security OAuth2 Resource Server

La elección entre el filtro JJWT personalizado y Spring Security OAuth2 Resource Server no es una cuestión de preferencia sino de contexto: el tipo de Authorization Server disponible, la necesidad de rotación automática de claves y la integración con el modelo de seguridad de Spring determinan cuál es más adecuado en cada escenario.

| Aspecto | Filtro personalizado JJWT | Spring Security OAuth2 Resource Server |
|---|---|---|
| Configuración | Código Java (GlobalFilter) | Principalmente YAML + mínimo Java |
| Rotación de claves | Manual (actualizar la clave en config) | Automática vía JWKS endpoint |
| Compatibilidad con OIDC | No nativa | Sí (discovery, userinfo endpoint) |
| Flexibilidad en extracción de claims | Total (código Java) | Alta (JwtAuthenticationConverter) |
| Integración con Spring Security | No | Sí (SecurityContext, @PreAuthorize) |
| Dependencias | `io.jsonwebtoken:jjwt-api` | `spring-boot-starter-oauth2-resource-server` |
| Caso de uso recomendado | JWT propietario, sin OIDC | Authorization Server estándar |

## Buenas y malas prácticas

Hacer:
- Centralizar la validación de JWT en el Gateway y propagar la identidad como headers internos (`X-User-Id`, `X-User-Roles`) en lugar de reenviar el token a cada microservicio. Esto elimina la duplicación de la lógica de validación y permite rotar la clave JWT en un único punto.
- Usar `RS256` (RSA asimétrico) en lugar de `HS256` (HMAC simétrico) cuando el Gateway y el Authorization Server son procesos distintos. Con `RS256`, solo el Authorization Server tiene la clave privada para firmar; el Gateway solo necesita la clave pública para verificar. Si la clave pública del Gateway se expone accidentalmente, no permite emitir nuevos tokens.
- Eliminar los headers `X-User-Id`, `X-User-Roles` y `X-Tenant-Id` de las peticiones entrantes antes de añadir los del JWT validado. Un cliente que conozca los nombres de los headers internos puede inyectar una identidad falsa si el Gateway no los elimina primero.

Evitar:
- Exponer los microservicios directamente a internet sin pasar por el Gateway. La seguridad de la arquitectura descrita depende de que los microservicios solo sean accesibles desde la red interna del clúster. Un microservicio expuesto públicamente recibe los headers `X-User-Id` y `X-User-Roles` que cualquier cliente puede falsificar.
- Loguear el token JWT completo en los logs de debug o de error del Gateway. Los tokens son credenciales equivalentes a contraseñas; su presencia en logs los expone a cualquier persona con acceso al sistema de logging o a los ficheros de log.
- No validar el claim `iss` (issuer) en el filtro JJWT personalizado. Sin validación del emisor, un token firmado con la misma clave por un sistema distinto (otro entorno, otro microservicio) podría usarse para autenticarse en el Gateway.

---

← [6.5.3 Clúster Redis e in-memory](./06-23-gateway-rate-limiter-variantes.md) | [Índice](./README.md) | [6.6.2 JwtAuthFilter con JJWT →](./06-25-gateway-jwt-filter.md)
