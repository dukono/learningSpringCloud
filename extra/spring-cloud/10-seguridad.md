# Parte 10 — Seguridad

← [Parte 9 — Observabilidad](./09-observabilidad.md) | [Volver al índice](./README.md) | Siguiente: [Parte 11 — Kubernetes](./11-kubernetes.md) →

---

## 10.1 Seguridad en arquitecturas de microservicios

Asegurar un sistema de microservicios es más complejo que un monolito porque hay múltiples superficies de ataque:

```
EXTERNOS:
  Usuario → Gateway → [autenticación aquí]
                          ↓
INTERNOS (service-to-service):
  Pedidos → Productos → Inventario
  [¿quién puede llamar a quién?]
```

### Los tres problemas de seguridad en microservicios

1. **Autenticación:** ¿Quién es este usuario/servicio?
2. **Autorización:** ¿Tiene permiso para hacer esto?
3. **Propagación:** ¿Cómo pasa la identidad del usuario de servicio en servicio?

### Patrón recomendado

```
[Usuario] → JWT → [Gateway]
                     │ valida JWT
                     │ extrae claims (userId, roles)
                     │ propaga como headers internos (X-User-Id, X-User-Roles)
                     ↓
              [Microservicio A]
                     │ lee headers internos (no valida JWT de nuevo)
                     │ aplica autorización de negocio
                     │ propaga headers en llamadas internas
                     ↓
              [Microservicio B]
```

---

## 10.2 OAuth2 y JWT en Spring Cloud

### Conceptos clave

| Término | Descripción |
|---|---|
| **OAuth2** | Protocolo de autorización (define el flujo de tokens) |
| **OIDC (OpenID Connect)** | Capa de identidad sobre OAuth2 (añade autenticación) |
| **JWT** | Formato del token (JSON Web Token, firmado digitalmente) |
| **Authorization Server** | Emite tokens (Keycloak, Okta, Spring Authorization Server) |
| **Resource Server** | Microservicio que protege sus recursos con tokens |
| **Client** | Aplicación que obtiene tokens para acceder a recursos |

### Flujo OAuth2 típico en microservicios

```
1. Usuario hace login → [Authorization Server] emite JWT
2. Cliente envía JWT en cabecera Authorization: Bearer <token>
3. Gateway valida el JWT contra la clave pública del Auth Server
4. Gateway extrae claims y los propaga a los microservicios
5. Microservicios usan los claims para autorización de negocio
```

### Dependencias

```xml
<!-- Resource Server (para microservicios) -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

### Configurar un microservicio como Resource Server

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .sessionManagement(sm -> sm.sessionCreationPolicy(STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/productos/**").hasRole("USER")
                .requestMatchers(HttpMethod.POST, "/api/productos/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt.jwtAuthenticationConverter(jwtAuthConverter()))
            );

        return http.build();
    }

    @Bean
    public JwtAuthenticationConverter jwtAuthConverter() {
        JwtGrantedAuthoritiesConverter grantedAuthoritiesConverter =
            new JwtGrantedAuthoritiesConverter();
        grantedAuthoritiesConverter.setAuthoritiesClaimName("roles");  // claim en el JWT
        grantedAuthoritiesConverter.setAuthorityPrefix("ROLE_");

        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(grantedAuthoritiesConverter);
        return converter;
    }
}
```

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          # Validación de JWT: opción 1 — JWKS endpoint del Auth Server
          jwk-set-uri: http://auth-server:8080/realms/miapp/protocol/openid-connect/certs
          # Opción 2 — clave pública directa
          # public-key-location: classpath:public.pem
          issuer-uri: http://auth-server:8080/realms/miapp
```

---

## 10.3 Spring Authorization Server

**Spring Authorization Server** es el servidor de autorización oficial de Spring (sustituto de Spring Security OAuth2 que fue deprecado). Puede usarse en lugar de Keycloak o Okta cuando se quiere un Auth Server 100% Java/Spring.

### Dependencia Maven

```xml
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-oauth2-authorization-server</artifactId>
</dependency>
```

### Configuración mínima

```java
@Configuration
@Import(OAuth2AuthorizationServerConfiguration.class)
public class AuthServerConfig {

    @Bean
    public RegisteredClientRepository registeredClientRepository() {
        RegisteredClient webClient = RegisteredClient.withId(UUID.randomUUID().toString())
            .clientId("web-client")
            .clientSecret("{bcrypt}" + new BCryptPasswordEncoder().encode("secret"))
            .clientAuthenticationMethod(ClientAuthenticationMethod.CLIENT_SECRET_BASIC)
            .authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
            .authorizationGrantType(AuthorizationGrantType.REFRESH_TOKEN)
            .redirectUri("http://localhost:3000/callback")
            .scope(OidcScopes.OPENID)
            .scope("pedidos:read")
            .scope("pedidos:write")
            .build();

        return new InMemoryRegisteredClientRepository(webClient);
    }

    @Bean
    public JWKSource<SecurityContext> jwkSource() {
        KeyPair keyPair = generateRsaKey();
        RSAKey rsaKey = new RSAKey.Builder((RSAPublicKey) keyPair.getPublic())
            .privateKey(keyPair.getPrivate())
            .keyID(UUID.randomUUID().toString())
            .build();
        return new ImmutableJWKSet<>(new JWKSet(rsaKey));
    }

    @Bean
    public AuthorizationServerSettings authorizationServerSettings() {
        return AuthorizationServerSettings.builder()
            .issuer("http://auth-server:9000")
            .build();
    }
}
```

---

## 10.4 Propagación de tokens entre servicios

### Problema

Cuando el Gateway recibe una petición del usuario con su JWT y la reenvía al Servicio A, y el Servicio A necesita llamar al Servicio B, ¿cómo sabe B quién es el usuario?

### Solución: propagar el JWT original

#### En el Gateway: pasar el JWT a los microservicios

```java
// En el Gateway, el JWT llega en Authorization: Bearer <token>
// Se propaga automáticamente si se configura así:
spring:
  cloud:
    gateway:
      default-filters:
        - TokenRelay=   # ← propaga el token del usuario a todos los servicios
```

#### En los microservicios: propagar en llamadas Feign

```java
@Component
public class FeignTokenInterceptor implements RequestInterceptor {

    @Override
    public void apply(RequestTemplate template) {
        // Obtener el token del contexto de seguridad de Spring
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();

        if (auth instanceof JwtAuthenticationToken jwtAuth) {
            String tokenValue = jwtAuth.getToken().getTokenValue();
            template.header("Authorization", "Bearer " + tokenValue);
        }
    }
}
```

### Solución alternativa: headers internos sin JWT

El Gateway valida el JWT una vez y propaga claims como headers simples:

```java
// En el Gateway (GlobalFilter)
ServerHttpRequest request = exchange.getRequest().mutate()
    .header("X-User-Id", claims.getSubject())
    .header("X-User-Email", claims.get("email", String.class))
    .header("X-User-Roles", claims.get("roles", String.class))
    .build();
```

```java
// En el microservicio (sin Spring Security)
@RestController
public class PedidosController {

    @GetMapping("/mis-pedidos")
    public List<Pedido> misPedidos(@RequestHeader("X-User-Id") String userId) {
        return pedidosService.obtenerPorUsuario(userId);
    }
}
```

**IMPORTANTE:** Esta aproximación requiere que los microservicios solo sean accesibles a través del Gateway (red interna), no directamente desde el exterior.

---

## 10.5 Seguridad en el Config Server

El Config Server puede contener secretos (contraseñas de BD, API keys). Debe estar protegido.

### Autenticación básica en el Config Server

```yaml
# application.yml del Config Server
spring:
  security:
    user:
      name: config-user
      password: ${CONFIG_SERVER_PASSWORD}
```

```yaml
# En cada Config Client
spring:
  cloud:
    config:
      uri: http://localhost:8888
      username: config-user
      password: ${CONFIG_SERVER_PASSWORD}
```

### Cifrado de propiedades sensibles en Git

Ver sección [3.8 — Cifrado y descifrado de propiedades sensibles](./03-config.md#38-cifrado-y-descifrado-de-propiedades-sensibles).

### Integración con HashiCorp Vault

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-vault-config</artifactId>
</dependency>
```

```yaml
spring:
  cloud:
    vault:
      uri: http://vault:8200
      authentication: TOKEN
      token: ${VAULT_TOKEN}
      kv:
        enabled: true
        default-context: application
        backend: secret
```

---

## 10.6 mTLS y comunicación segura entre microservicios

**mTLS (Mutual TLS)** es el mecanismo por el que dos servicios se autentican mutuamente mediante certificados digitales, sin necesidad de tokens en las cabeceras.

### Cuándo usar mTLS

- Entorno de alta seguridad (finanzas, salud, defensa)
- Comunicación servicio-a-servicio sin pasar por el Gateway
- Complemento a OAuth2 para zero-trust networking

### Configuración con Spring Boot y keystore

```yaml
server:
  ssl:
    enabled: true
    key-store: classpath:keystore.p12
    key-store-password: ${KEYSTORE_PASSWORD}
    key-store-type: PKCS12
    key-alias: pedidos-service
    # mTLS — requerir certificado del cliente
    client-auth: need
    trust-store: classpath:truststore.p12
    trust-store-password: ${TRUSTSTORE_PASSWORD}
```

### RestTemplate con certificado cliente

```java
@Bean
public RestTemplate restTemplate() throws Exception {
    KeyStore keyStore = KeyStore.getInstance("PKCS12");
    keyStore.load(
        new FileInputStream("keystore.p12"),
        "password".toCharArray()
    );

    SSLContext sslContext = SSLContextBuilder.create()
        .loadKeyMaterial(keyStore, "password".toCharArray())
        .loadTrustMaterial(trustStore, null)
        .build();

    CloseableHttpClient httpClient = HttpClients.custom()
        .setSSLContext(sslContext)
        .build();

    HttpComponentsClientHttpRequestFactory factory =
        new HttpComponentsClientHttpRequestFactory(httpClient);

    return new RestTemplate(factory);
}
```

---

← [Parte 9 — Observabilidad](./09-observabilidad.md) | [Volver al índice](./README.md) | Siguiente: [Parte 11 — Kubernetes](./11-kubernetes.md) →
