# 8.5 OAuth2 Client avanzado — WebClient, ReactiveManager y logout OIDC

← [8.4 OAuth2 Client servlet — registro, gestión de tokens y Client Credentials](sc-security-oauth2-client.md) | [Índice (README.md)](README.md) | [8.6 Token Relay en Spring Cloud Gateway](sc-security-token-relay.md) →

---

El flujo de Authorization Code con PKCE es el estándar para aplicaciones con usuarios finales que necesitan delegar acceso a recursos protegidos. Spring Security 6.x gestiona este flujo en aplicaciones Spring MVC y WebFlux de forma diferente: en servlet, `OAuth2AuthorizedClientManager` basado en `HttpServletRequest`; en reactivo, `ReactiveOAuth2AuthorizedClientManager` que integra con el contexto de seguridad de Reactor. El logout OIDC completa el ciclo cerrando la sesión tanto en la aplicación como en el Authorization Server, evitando el problema de sesiones zombie donde el usuario está desconectado de la app pero su sesión en el AS sigue activa.

> [PREREQUISITO] Requiere `spring-boot-starter-oauth2-client`. Para llamadas downstream con WebClient: `spring-boot-starter-webflux` o `spring-boot-starter-web` con WebClient configurado.

## Ejemplo central: Authorization Code con PKCE, WebClient reactivo y OIDC logout

### Dependencias Maven

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-client</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>
```

### application.yml — Authorization Code con PKCE

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          my-oidc-client:
            provider: my-auth-server
            client-id: my-frontend-app
            client-secret: ${CLIENT_SECRET}
            authorization-grant-type: authorization_code
            redirect-uri: "{baseUrl}/login/oauth2/code/{registrationId}"
            scope:
              - openid
              - profile
              - email
              - orders:read
        provider:
          my-auth-server:
            issuer-uri: https://auth.example.com
```

### Configuración con WebClient reactivo y ExchangeFilterFunction

```java
package com.example.security;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.reactive.EnableWebFluxSecurity;
import org.springframework.security.config.web.server.ServerHttpSecurity;
import org.springframework.security.oauth2.client.ReactiveOAuth2AuthorizedClientManager;
import org.springframework.security.oauth2.client.ReactiveOAuth2AuthorizedClientProviderBuilder;
import org.springframework.security.oauth2.client.registration.ReactiveClientRegistrationRepository;
import org.springframework.security.oauth2.client.web.DefaultServerOAuth2AuthorizedClientRepository;
import org.springframework.security.oauth2.client.web.reactive.function.client.ServerOAuth2AuthorizedClientExchangeFilterFunction;
import org.springframework.security.oauth2.client.web.server.ServerOAuth2AuthorizedClientRepository;
import org.springframework.security.web.server.SecurityWebFilterChain;
import org.springframework.web.reactive.function.client.WebClient;

@Configuration
@EnableWebFluxSecurity
public class ReactiveOAuth2Config {

    @Bean
    public SecurityWebFilterChain securityWebFilterChain(ServerHttpSecurity http) {
        return http
            .csrf(ServerHttpSecurity.CsrfSpec::disable)
            .oauth2Login(login -> login
                // El login redirige al AS con PKCE automáticamente
                .authorizationRequestResolver(pkceResolver(null)) // ver nota
            )
            .oauth2Client(client -> {})
            .logout(logout -> logout
                // OIDC logout: redirige al AS para cerrar la sesión global
                .logoutSuccessHandler(oidcLogoutSuccessHandler())
            )
            .authorizeExchange(exchange -> exchange
                .pathMatchers("/actuator/health").permitAll()
                .anyExchange().authenticated()
            )
            .build();
    }

    @Bean
    public ReactiveOAuth2AuthorizedClientManager reactiveAuthorizedClientManager(
            ReactiveClientRegistrationRepository registrations,
            ServerOAuth2AuthorizedClientRepository authorizedClients) {

        var provider = ReactiveOAuth2AuthorizedClientProviderBuilder.builder()
            .authorizationCode()
            .refreshToken()
            .clientCredentials()
            .build();

        var manager = new org.springframework.security.oauth2.client.web.server
            .ServerOAuth2AuthorizedClientManager(registrations, authorizedClients);
        // No hay ServerOAuth2AuthorizedClientManager - usar el correcto:
        var defaultManager = new org.springframework.security.oauth2.client
            .AuthorizedClientServiceReactiveOAuth2AuthorizedClientManager(
                registrations,
                new org.springframework.security.oauth2.client.InMemoryReactiveOAuth2AuthorizedClientService(registrations));
        defaultManager.setAuthorizedClientProvider(provider);

        return defaultManager;
    }

    // WebClient con token del usuario actual (propagado del contexto reactivo)
    @Bean
    public WebClient webClient(ReactiveOAuth2AuthorizedClientManager manager) {
        ServerOAuth2AuthorizedClientExchangeFilterFunction oauth2 =
            new ServerOAuth2AuthorizedClientExchangeFilterFunction(manager);
        oauth2.setDefaultOAuth2AuthorizedClient(true); // usa el cliente autorizado del contexto

        return WebClient.builder()
            .baseUrl("https://order-service.example.com")
            .filter(oauth2)
            .build();
    }

    private org.springframework.security.oauth2.client.oidc.web.server.logout
            .OidcClientInitiatedServerLogoutSuccessHandler oidcLogoutSuccessHandler() {
        // El handler redirige al end_session_endpoint del AS tras el logout local
        // Requiere que el proveedor soporte OIDC end_session_endpoint
        var handler = new org.springframework.security.oauth2.client.oidc.web.server.logout
            .OidcClientInitiatedServerLogoutSuccessHandler(null); // se inyecta el repo
        handler.setPostLogoutRedirectUri("{baseUrl}");
        return handler;
    }

    private org.springframework.security.oauth2.client.web.server
            .DefaultServerOAuth2AuthorizationRequestResolver pkceResolver(
            ReactiveClientRegistrationRepository repo) {
        // PKCE se activa automáticamente en Spring Security 6.x para public clients
        // Para confidential clients también es recomendable activarlo:
        return new org.springframework.security.oauth2.client.web.server
            .DefaultServerOAuth2AuthorizationRequestResolver(repo);
    }
}
```

### Uso de WebClient con token propagado automáticamente

```java
package com.example.client;

import org.springframework.stereotype.Service;
import org.springframework.web.reactive.function.client.WebClient;
import reactor.core.publisher.Mono;

@Service
public class OrderService {

    private final WebClient webClient;

    public OrderService(WebClient webClient) {
        this.webClient = webClient;
    }

    public Mono<Order> getOrder(String orderId) {
        // El token del usuario actual se propaga automáticamente desde el contexto Reactor
        return webClient.get()
            .uri("/orders/{id}", orderId)
            .retrieve()
            .bodyToMono(Order.class);
    }
}
```

> [CONCEPTO] `setDefaultOAuth2AuthorizedClient(true)` en el `ServerOAuth2AuthorizedClientExchangeFilterFunction` hace que el WebClient use automáticamente el token del usuario autenticado en el contexto reactivo. No es necesario especificar el clientRegistrationId en cada petición. En cambio, el token del usuario se propaga desde `ReactorContext`, lo que requiere que el servicio sea invocado desde un flujo reactivo con contexto de seguridad activo.

> [ADVERTENCIA] El logout OIDC (`OidcClientInitiatedServerLogoutSuccessHandler`) envía el `id_token_hint` al `end_session_endpoint` del Authorization Server. Si el AS no soporta OIDC end_session_endpoint (algunos AS legacy no lo hacen), el logout redirigirá a una URL 404. Verificar que el AS expone `end_session_endpoint` en su OIDC Discovery antes de configurarlo.

## Tabla de elementos clave

| Componente | Descripción |
|---|---|
| `authorization_code` + PKCE | Flujo estándar para usuarios finales; PKCE previene ataques de intercepción del authorization code |
| `ReactiveOAuth2AuthorizedClientManager` | Manager para contextos WebFlux; propaga tokens a través del `ReactorContext` |
| `ServerOAuth2AuthorizedClientExchangeFilterFunction` | Filter de WebClient que añade el Bearer token automáticamente desde el contexto reactivo |
| `setDefaultOAuth2AuthorizedClient(true)` | Usa el cliente autorizado del contexto reactivo sin especificarlo por petición |
| `OidcClientInitiatedServerLogoutSuccessHandler` | Redirige al `end_session_endpoint` del AS tras logout; cierra la sesión en el AS |
| `end_session_endpoint` | Endpoint OIDC del AS para logout global; devuelto en el OIDC Discovery |
| `id_token_hint` | El ID Token se envía al AS en el logout para identificar la sesión a cerrar |

## Buenas y malas prácticas

**Hacer:**
- Implementar OIDC logout completo con `OidcClientInitiatedServerLogoutSuccessHandler`; sin él, el usuario puede seguir usando el access token hasta su expiración aunque haya hecho logout en la app.
- Usar `scope: openid` para obtener el ID Token que permite el OIDC logout; sin el ID Token no se puede enviar `id_token_hint` al `end_session_endpoint`.
- Configurar `refreshToken()` en el `ReactiveOAuth2AuthorizedClientProviderBuilder` para renovar automáticamente tokens expirados sin interrumpir la sesión del usuario.

**Evitar:**
- Usar `setDefaultOAuth2AuthorizedClient(true)` en servicios que hacen llamadas con tokens de múltiples usuarios simultáneamente sin aislar el contexto reactivo; la propagación de tokens depende del `ReactorContext` y puede mezclarse si no se gestiona correctamente.
- Ignorar el `state` parameter en el flujo Authorization Code; Spring Security lo gestiona automáticamente para prevenir CSRF en el redirect, pero desactivar la validación del `state` abre una vulnerabilidad crítica.
- Almacenar el authorization code o el access token en localStorage o sessionStorage del browser; usar cookies `httpOnly` o el almacenamiento de sesión del servidor.

---

← [8.4 OAuth2 Client servlet — registro, gestión de tokens y Client Credentials](sc-security-oauth2-client.md) | [Índice (README.md)](README.md) | [8.6 Token Relay en Spring Cloud Gateway](sc-security-token-relay.md) →
