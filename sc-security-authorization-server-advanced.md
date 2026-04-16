# 8.11 Spring Authorization Server — personalización avanzada y OIDC

← [8.10 Spring Authorization Server — configuración base](sc-security-authorization-server.md) | [Índice (README.md)](README.md) | [8.12 Seguridad en microservicios reactivos con WebFlux](sc-security-reactive.md) →

---

La configuración base de Spring Authorization Server emite tokens con las claims estándar OAuth2. En producción, los equipos necesitan añadir claims de negocio al JWT (userId, tenantId, roles de la base de datos propia), personalizar el UserInfo endpoint de OIDC, e implementar la gestión de consentimientos. Spring Authorization Server provee tres puntos de extensión principales: `OAuth2TokenCustomizer` para modificar el contenido del JWT, `OidcUserInfoService` para el endpoint `/userinfo`, y el `AuthenticationProvider` personalizado para flujos de autenticación propios. Todos estos puntos de extensión son beans de Spring configurables sin modificar el código del servidor.

> [PREREQUISITO] Requiere el fichero 8.10 (configuración base del Authorization Server). Sin la configuración base funcional, la personalización no tiene efecto.

## Ejemplo central: claims personalizadas, UserInfo OIDC y consentimiento

### OAuth2TokenCustomizer — añadir claims al JWT

```java
package com.example.authserver;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.oauth2.server.authorization.OAuth2TokenType;
import org.springframework.security.oauth2.server.authorization.token.JwtEncodingContext;
import org.springframework.security.oauth2.server.authorization.token.OAuth2TokenCustomizer;

@Configuration
public class TokenCustomizationConfig {

    private final UserRepository userRepository;

    public TokenCustomizationConfig(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    // Personaliza el contenido del JWT antes de firmarlo
    @Bean
    public OAuth2TokenCustomizer<JwtEncodingContext> jwtTokenCustomizer() {
        return context -> {
            // Solo personalizar access tokens y ID tokens, no refresh tokens
            if (OAuth2TokenType.ACCESS_TOKEN.equals(context.getTokenType())) {
                String username = context.getPrincipal().getName();
                // Cargar datos del usuario desde la base de datos propia
                userRepository.findByUsername(username).ifPresent(user -> {
                    context.getClaims()
                        .claim("user_id", user.getId().toString())
                        .claim("tenant_id", user.getTenantId())
                        .claim("roles", user.getRoles())
                        .claim("email", user.getEmail());
                });
            }

            // Añadir claims al ID Token (OIDC)
            if ("id_token".equals(context.getTokenType().getValue())) {
                String username = context.getPrincipal().getName();
                userRepository.findByUsername(username).ifPresent(user -> {
                    context.getClaims()
                        .claim("preferred_username", username)
                        .claim("email", user.getEmail())
                        .claim("email_verified", true)
                        .claim("name", user.getFullName());
                });
            }
        };
    }
}
```

### OidcUserInfoService — UserInfo endpoint personalizado

```java
package com.example.authserver;

import org.springframework.security.oauth2.core.oidc.OidcUserInfo;
import org.springframework.security.oauth2.server.authorization.oidc.authentication.OidcUserInfoAuthenticationContext;
import org.springframework.security.oauth2.server.authorization.oidc.authentication.OidcUserInfoAuthenticationToken;
import org.springframework.stereotype.Service;

import java.util.HashMap;
import java.util.Map;
import java.util.function.Function;

@Service
public class CustomOidcUserInfoService
        implements Function<OidcUserInfoAuthenticationContext, OidcUserInfo> {

    private final UserRepository userRepository;

    public CustomOidcUserInfoService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Override
    public OidcUserInfo apply(OidcUserInfoAuthenticationContext context) {
        OidcUserInfoAuthenticationToken authentication = context.getAuthentication();
        String username = authentication.getPrincipal().getName();

        Map<String, Object> claims = new HashMap<>();
        userRepository.findByUsername(username).ifPresent(user -> {
            // Claims del perfil del usuario
            claims.put("sub", username);
            claims.put("name", user.getFullName());
            claims.put("preferred_username", username);
            claims.put("email", user.getEmail());
            claims.put("email_verified", true);

            // Claims de negocio propias
            claims.put("tenant_id", user.getTenantId());
            claims.put("roles", user.getRoles());
        });

        return new OidcUserInfo(claims);
    }
}
```

```java
// Registro del UserInfoService en el AuthorizationServer configurer
@Bean
@Order(1)
public SecurityFilterChain authorizationServerSecurityFilterChain(
        HttpSecurity http,
        CustomOidcUserInfoService userInfoService) throws Exception {

    OAuth2AuthorizationServerConfiguration.applyDefaultSecurity(http);
    http
        .getConfigurer(OAuth2AuthorizationServerConfigurer.class)
        .oidc(oidc -> oidc
            .userInfoEndpoint(userInfo -> userInfo
                .userInfoMapper(userInfoService)
            )
        );

    http.exceptionHandling(e -> e.authenticationEntryPoint(
        new org.springframework.security.web.authentication.LoginUrlAuthenticationEntryPoint("/login")));

    return http.build();
}
```

### Gestión de consentimiento OAuth2 personalizado

```java
package com.example.authserver;

import org.springframework.security.oauth2.server.authorization.OAuth2AuthorizationConsentService;
import org.springframework.security.oauth2.server.authorization.JdbcOAuth2AuthorizationConsentService;
import org.springframework.security.oauth2.server.authorization.client.RegisteredClientRepository;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.jdbc.core.JdbcTemplate;

@Configuration
public class ConsentConfig {

    // Persistir consentimientos en base de datos
    @Bean
    public OAuth2AuthorizationConsentService authorizationConsentService(
            JdbcTemplate jdbcTemplate,
            RegisteredClientRepository registeredClientRepository) {
        return new JdbcOAuth2AuthorizationConsentService(
            jdbcTemplate, registeredClientRepository);
    }
}
```

```java
// Controlador para la pantalla de consentimiento personalizada
package com.example.authserver;

import org.springframework.security.oauth2.server.authorization.OAuth2AuthorizationConsent;
import org.springframework.security.oauth2.server.authorization.OAuth2AuthorizationConsentService;
import org.springframework.security.oauth2.server.authorization.client.RegisteredClientRepository;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;

import java.security.Principal;
import java.util.Set;

@Controller
public class ConsentController {

    private final RegisteredClientRepository clientRepository;
    private final OAuth2AuthorizationConsentService consentService;

    public ConsentController(RegisteredClientRepository clientRepository,
                              OAuth2AuthorizationConsentService consentService) {
        this.clientRepository = clientRepository;
        this.consentService = consentService;
    }

    // La pantalla de consentimiento muestra qué scopes está autorizando el usuario
    @GetMapping("/oauth2/consent")
    public String consent(
            Principal principal,
            Model model,
            @RequestParam(name = "client_id") String clientId,
            @RequestParam(name = "scope") String scope,
            @RequestParam(name = "state") String state) {

        var client = clientRepository.findByClientId(clientId);
        OAuth2AuthorizationConsent existingConsent = consentService
            .findById(clientId, principal.getName());

        Set<String> requestedScopes = Set.of(scope.split(" "));

        model.addAttribute("clientId", clientId);
        model.addAttribute("clientName", client.getClientName());
        model.addAttribute("state", state);
        model.addAttribute("scopes", requestedScopes);
        model.addAttribute("principalName", principal.getName());

        return "consent";  // template Thymeleaf: src/main/resources/templates/consent.html
    }
}
```

> [CONCEPTO] `OAuth2TokenCustomizer<JwtEncodingContext>` se ejecuta en el momento de firma del JWT, después de que Spring Authorization Server haya construido las claims estándar. Las claims añadidas aquí aparecerán en todos los tokens del tipo especificado, pero solo si el scope solicitado lo justifica — en producción, filtrar qué claims se añaden según los scopes concedidos en `context.getAuthorizedScopes()`.

## Tabla de elementos clave

| Componente | Descripción |
|---|---|
| `OAuth2TokenCustomizer<JwtEncodingContext>` | Hook para añadir o modificar claims del JWT antes de firmarlo |
| `context.getTokenType()` | Tipo de token que se está generando: `ACCESS_TOKEN`, `REFRESH_TOKEN`, `"id_token"` |
| `context.getClaims()` | Builder mutable de claims del JWT; permite `.claim(name, value)` |
| `context.getAuthorizedScopes()` | Scopes aprobados en el request; úsalos para filtrar qué claims añadir |
| `OidcUserInfoAuthenticationContext` | Contexto del endpoint `/userinfo`; el `getPrincipal()` es el token de acceso |
| `JdbcOAuth2AuthorizationConsentService` | Persiste consentimientos en base de datos; schema DDL en el JAR del AS |
| `JdbcOAuth2AuthorizationService` | Persiste tokens activos en base de datos; alternativa a la implementación en memoria |

## Buenas y malas prácticas

**Hacer:**
- Filtrar las claims añadidas por `OAuth2TokenCustomizer` según los scopes concedidos: solo añadir `email` si `openid profile email` están en `context.getAuthorizedScopes()`; reducir el tamaño del token a las claims estrictamente necesarias.
- Usar `JdbcOAuth2AuthorizationService` y `JdbcOAuth2AuthorizationConsentService` en producción; los equivalentes en memoria pierden toda la información de tokens activos y consentimientos al reiniciar.
- Hacer el `OAuth2TokenCustomizer` resiliente a errores del repositorio: si `userRepository.findByUsername()` falla, no lanzar excepción — emitir el token sin las claims de negocio y loggear el error; un fallo en la personalización no debe impedir la autenticación.

**Evitar:**
- Añadir datos sensibles (contraseñas, información médica, datos financieros) como claims del JWT: el payload es Base64url, legible por cualquiera con el token; las claims son para autorización, no para datos de negocio.
- Sobrecargar el JWT con claims muy grandes (listas completas de permisos por recurso, ACLs complejas): el token se envía en cada petición HTTP y headers excesivamente grandes pueden causar errores en proxies/load balancers con límite de header size.
- Ignorar la gestión de consentimiento para clientes confidenciales internos: aunque sea un flujo completamente controlado, el consentimiento explícito es un requisito de OAuth2 que algunos auditores de seguridad verifican.

---

← [8.10 Spring Authorization Server — configuración base](sc-security-authorization-server.md) | [Índice (README.md)](README.md) | [8.12 Seguridad en microservicios reactivos con WebFlux](sc-security-reactive.md) →
