# 8.10 Spring Authorization Server — configuración base

← [8.9 Propagación de identidad en contextos asíncronos y mensajería](sc-security-propagation-async.md) | [Índice (README.md)](README.md) | [8.11 Spring Authorization Server — personalización avanzada y OIDC](sc-security-authorization-server-advanced.md) →

---

Spring Authorization Server (SAS) 1.x es la implementación de Authorization Server de Spring que reemplaza al antiguo Spring Security OAuth2 (deprecado desde 2019). Implementa OAuth2.1, OpenID Connect 1.0 y los endpoints estándar: `/oauth2/authorize`, `/oauth2/token`, `/oauth2/jwks`, `/oauth2/introspect`, `/oauth2/revoke` y `/.well-known/openid-configuration`. La configuración base establece los tres pilares: los `RegisteredClient` (clientes OAuth2 registrados), el `JWKSource` (claves de firma de tokens), y la `SecurityFilterChain` de autorización. Sin estos tres componentes el servidor no puede emitir tokens válidos.

> [PREREQUISITO] Requiere `spring-security-oauth2-authorization-server` (incluido en el BOM `spring-boot-dependencies`). Este fichero cubre la configuración mínima funcional; la personalización avanzada (claims personalizados, OIDC, flujos adicionales) está en el fichero 8.11.

## Arquitectura del Authorization Server

```
┌──────────────────────────────────────────────────────────────────┐
│  Spring Authorization Server (SAS 1.x)                           │
│                                                                   │
│  Endpoints OAuth2 / OIDC                                         │
│  ├── POST /oauth2/token          ← emitir tokens                 │
│  ├── GET  /oauth2/authorize      ← flujo Authorization Code      │
│  ├── GET  /oauth2/jwks           ← claves públicas para verify   │
│  ├── POST /oauth2/introspect     ← validar token opaco           │
│  ├── POST /oauth2/revoke         ← revocar token                 │
│  └── GET  /.well-known/openid-configuration ← discovery         │
│                                                                   │
│  Componentes configurables                                        │
│  ├── RegisteredClientRepository ← clientes registrados          │
│  ├── AuthorizationServerSettings ← URLs de los endpoints        │
│  ├── JWKSource                  ← clave RSA/EC para firmar JWT   │
│  └── OAuth2TokenCustomizer      ← añadir claims custom al JWT   │
└──────────────────────────────────────────────────────────────────┘
```

## Ejemplo central: Authorization Server mínimo funcional

### Dependencias Maven

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>4.0.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    <!-- Authorization Server -->
    <dependency>
        <groupId>org.springframework.security</groupId>
        <artifactId>spring-security-oauth2-authorization-server</artifactId>
    </dependency>
    <!-- JPA para persistir tokens y clients (producción) -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
</dependencies>
```

### SecurityConfig — tres SecurityFilterChains requeridas

```java
package com.example.authserver;

import com.nimbusds.jose.jwk.JWKSet;
import com.nimbusds.jose.jwk.RSAKey;
import com.nimbusds.jose.jwk.source.ImmutableJWKSet;
import com.nimbusds.jose.jwk.source.JWKSource;
import com.nimbusds.jose.proc.SecurityContext;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.annotation.Order;
import org.springframework.security.config.Customizer;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.oauth2.core.AuthorizationGrantType;
import org.springframework.security.oauth2.core.ClientAuthenticationMethod;
import org.springframework.security.oauth2.core.oidc.OidcScopes;
import org.springframework.security.oauth2.jwt.JwtDecoder;
import org.springframework.security.oauth2.server.authorization.client.InMemoryRegisteredClientRepository;
import org.springframework.security.oauth2.server.authorization.client.RegisteredClient;
import org.springframework.security.oauth2.server.authorization.client.RegisteredClientRepository;
import org.springframework.security.oauth2.server.authorization.config.annotation.web.configuration.OAuth2AuthorizationServerConfiguration;
import org.springframework.security.oauth2.server.authorization.config.annotation.web.configurers.OAuth2AuthorizationServerConfigurer;
import org.springframework.security.oauth2.server.authorization.settings.AuthorizationServerSettings;
import org.springframework.security.oauth2.server.authorization.settings.ClientSettings;
import org.springframework.security.oauth2.server.authorization.settings.TokenSettings;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.LoginUrlAuthenticationEntryPoint;

import java.security.KeyPair;
import java.security.KeyPairGenerator;
import java.security.interfaces.RSAPrivateKey;
import java.security.interfaces.RSAPublicKey;
import java.time.Duration;
import java.util.UUID;

@Configuration
@EnableWebSecurity
public class AuthorizationServerConfig {

    // (1) SecurityFilterChain para los endpoints del Authorization Server
    @Bean
    @Order(1)
    public SecurityFilterChain authorizationServerSecurityFilterChain(HttpSecurity http) throws Exception {
        OAuth2AuthorizationServerConfiguration.applyDefaultSecurity(http);
        http
            .getConfigurer(OAuth2AuthorizationServerConfigurer.class)
            // Habilitar OpenID Connect 1.0
            .oidc(Customizer.withDefaults());
        http
            // Redirigir al login cuando se requiere autenticación en el flujo Authorization Code
            .exceptionHandling(exceptions -> exceptions
                .authenticationEntryPoint(
                    new LoginUrlAuthenticationEntryPoint("/login"))
            )
            // El Authorization Server también actúa como Resource Server para el userinfo endpoint
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()));

        return http.build();
    }

    // (2) SecurityFilterChain para el login del usuario (formulario estándar)
    @Bean
    @Order(2)
    public SecurityFilterChain defaultSecurityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health").permitAll()
                .anyRequest().authenticated()
            )
            .formLogin(Customizer.withDefaults());

        return http.build();
    }

    // (3) RegisteredClientRepository — clientes OAuth2 registrados
    @Bean
    public RegisteredClientRepository registeredClientRepository() {
        // Cliente para Authorization Code con PKCE (frontend/SPA)
        RegisteredClient frontendClient = RegisteredClient.withId(UUID.randomUUID().toString())
            .clientId("frontend-app")
            .clientSecret("{noop}frontend-secret")  // {noop} = sin hashing (solo para dev)
            .clientAuthenticationMethod(ClientAuthenticationMethod.CLIENT_SECRET_BASIC)
            .authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
            .authorizationGrantType(AuthorizationGrantType.REFRESH_TOKEN)
            .redirectUri("http://localhost:3000/callback")
            .postLogoutRedirectUri("http://localhost:3000")
            .scope(OidcScopes.OPENID)
            .scope(OidcScopes.PROFILE)
            .scope("orders:read")
            .scope("orders:write")
            .clientSettings(ClientSettings.builder()
                .requireProofKey(true)          // PKCE obligatorio
                .requireAuthorizationConsent(false)
                .build())
            .tokenSettings(TokenSettings.builder()
                .accessTokenTimeToLive(Duration.ofMinutes(15))
                .refreshTokenTimeToLive(Duration.ofDays(1))
                .reuseRefreshTokens(false)      // nuevo refresh token en cada uso
                .build())
            .build();

        // Cliente para Client Credentials (servicio-a-servicio)
        RegisteredClient serviceClient = RegisteredClient.withId(UUID.randomUUID().toString())
            .clientId("order-service")
            .clientSecret("{bcrypt}$2a$10$...")  // usar BCrypt en producción
            .clientAuthenticationMethod(ClientAuthenticationMethod.CLIENT_SECRET_BASIC)
            .authorizationGrantType(AuthorizationGrantType.CLIENT_CREDENTIALS)
            .scope("inventory:read")
            .tokenSettings(TokenSettings.builder()
                .accessTokenTimeToLive(Duration.ofMinutes(30))
                .build())
            .build();

        return new InMemoryRegisteredClientRepository(frontendClient, serviceClient);
    }

    // (4) JWKSource — par de claves RSA para firmar los JWT
    @Bean
    public JWKSource<SecurityContext> jwkSource() {
        KeyPair keyPair = generateRsaKey();
        RSAPublicKey publicKey = (RSAPublicKey) keyPair.getPublic();
        RSAPrivateKey privateKey = (RSAPrivateKey) keyPair.getPrivate();
        RSAKey rsaKey = new RSAKey.Builder(publicKey)
            .privateKey(privateKey)
            .keyID(UUID.randomUUID().toString())
            .build();
        JWKSet jwkSet = new JWKSet(rsaKey);
        return new ImmutableJWKSet<>(jwkSet);
    }

    private static KeyPair generateRsaKey() {
        try {
            KeyPairGenerator generator = KeyPairGenerator.getInstance("RSA");
            generator.initialize(2048);
            return generator.generateKeyPair();
        } catch (Exception ex) {
            throw new IllegalStateException("Failed to generate RSA key pair", ex);
        }
    }

    // (5) JwtDecoder para el userinfo endpoint
    @Bean
    public JwtDecoder jwtDecoder(JWKSource<SecurityContext> jwkSource) {
        return OAuth2AuthorizationServerConfiguration.jwtDecoder(jwkSource);
    }

    // (6) AuthorizationServerSettings — issuer URI
    @Bean
    public AuthorizationServerSettings authorizationServerSettings() {
        return AuthorizationServerSettings.builder()
            .issuer("https://auth.example.com")
            .build();
    }
}
```

> [ADVERTENCIA] `generateRsaKey()` genera un nuevo par de claves en cada arranque del servidor. En producción, la clave privada debe persistirse (en Vault, KMS, o keystore en base de datos) para que los tokens emitidos antes de un reinicio sigan siendo válidos. Los Resource Servers cachean el JWK Set — si la clave cambia, los tokens existentes dejarán de validarse hasta que el JWK Set se refresque.

> [CONCEPTO] `InMemoryRegisteredClientRepository` pierde los clientes registrados al reiniciar. En producción, usar `JdbcRegisteredClientRepository` que persiste en base de datos. Spring Authorization Server provee el DDL para crear las tablas necesarias: `spring-security-oauth2-authorization-server-schema.sql`.

## Tabla de elementos clave

| Componente | Descripción |
|---|---|
| `OAuth2AuthorizationServerConfiguration.applyDefaultSecurity(http)` | Aplica la configuración por defecto de todos los endpoints OAuth2/OIDC en un paso |
| `RegisteredClient` | Representa un cliente OAuth2 con sus grant types, scopes, redirect URIs y settings |
| `ClientSettings.requireProofKey(true)` | Requiere PKCE para este cliente; obligatorio para public clients (SPAs) |
| `TokenSettings.accessTokenTimeToLive()` | Configura la expiración del access token; default: 5 minutos |
| `InMemoryRegisteredClientRepository` | Repositorio en memoria; usar solo para dev/test |
| `JdbcRegisteredClientRepository` | Repositorio persistente en base de datos; para producción |
| `JWKSource` | Provee las claves JWK para firmar los JWT; `ImmutableJWKSet` para clave fija |
| `AuthorizationServerSettings.issuer()` | URI del issuer que aparece en el claim `iss` de todos los tokens |

## Buenas y malas prácticas

**Hacer:**
- Persistir las claves RSA del `JWKSource` en Vault o KMS en producción; una clave regenerada en cada arranque invalida todos los tokens existentes.
- Usar `{bcrypt}` en los `clientSecret` de producción; `{noop}` solo es aceptable en entornos de desarrollo.
- Configurar `reuseRefreshTokens(false)` para emitir un nuevo refresh token en cada renovación; esto permite detectar el uso de un refresh token robado (el refresh token anterior queda inválido).

**Evitar:**
- Usar `InMemoryRegisteredClientRepository` en producción; los clientes y sus datos de autorización se pierden al reiniciar el servidor.
- Configurar el `accessTokenTimeToLive` mayor de 60 minutos; en caso de compromiso del token, el tiempo de expiración corto limita la ventana de ataque.
- Ignorar el endpoint `/oauth2/revoke`; los tokens comprometidos deben poder revocarse sin esperar a su expiración natural.

---

← [8.9 Propagación de identidad en contextos asíncronos y mensajería](sc-security-propagation-async.md) | [Índice (README.md)](README.md) | [8.11 Spring Authorization Server — personalización avanzada y OIDC](sc-security-authorization-server-advanced.md) →
