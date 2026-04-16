# 8.4 OAuth2 Client servlet — registro, gestión de tokens y Client Credentials

← [8.3 Resource Server avanzado — claims, opaque tokens y multi-tenancy](sc-security-resource-server-advanced.md) | [Índice (README.md)](README.md) | [8.5 OAuth2 Client avanzado — WebClient, ReactiveManager y logout OIDC](sc-security-oauth2-client-advanced.md) →

---

Un microservicio actúa como OAuth2 Client cuando necesita obtener tokens para llamar a otros servicios (Machine-to-Machine, flujo Client Credentials) o cuando actúa de intermediario entre el usuario y los servicios backend (flujo Authorization Code). Spring Security 6.x gestiona ambos flujos a través de `ClientRegistration`, `OAuth2AuthorizedClientManager` y el mecanismo de intercepción de tokens en `RestClient`. El flujo más frecuente en microservicios backend es Client Credentials: el servicio obtiene un access token con su propia identidad (sin usuario) para llamar a otro servicio downstream.

> [PREREQUISITO] Requiere `spring-boot-starter-oauth2-client`. Para llamadas HTTP downstream con tokens: `spring-boot-starter-web` (RestClient/RestTemplate).

## Flujo Client Credentials en microservicios

```
┌─────────────────────────────────────────────────────────────┐
│  Microservicio A (OAuth2 Client)                             │
│                                                              │
│  1. OAuth2AuthorizedClientManager solicita token al AS      │
│     POST /oauth2/token                                       │
│     grant_type=client_credentials                           │
│     client_id=service-a                                     │
│     client_secret=secret                                    │
│     scope=orders:read                                       │
│                                                              │
│  2. AS devuelve access_token JWT                            │
│                                                              │
│  3. RestClient incluye automáticamente                      │
│     Authorization: Bearer <token>                           │
│     en cada petición a Microservicio B                      │
└─────────────────────────────────────────────────────────────┘
```

## Ejemplo central: Client Credentials con RestClient

### Dependencias Maven

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
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-oauth2-client</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>
```

### application.yml — registro del cliente OAuth2

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          # Nombre del registro — se referencia en el código como "order-service-cc"
          order-service-cc:
            provider: my-auth-server
            client-id: inventory-service
            client-secret: ${CLIENT_SECRET}
            authorization-grant-type: client_credentials
            scope: orders:read, orders:write
        provider:
          my-auth-server:
            issuer-uri: https://auth.example.com
            # Alternativa sin OIDC discovery:
            # token-uri: https://auth.example.com/oauth2/token
```

### Configuración del OAuth2AuthorizedClientManager y RestClient

```java
package com.example.client;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.oauth2.client.AuthorizedClientServiceOAuth2AuthorizedClientManager;
import org.springframework.security.oauth2.client.OAuth2AuthorizedClientManager;
import org.springframework.security.oauth2.client.OAuth2AuthorizedClientProviderBuilder;
import org.springframework.security.oauth2.client.OAuth2AuthorizedClientService;
import org.springframework.security.oauth2.client.registration.ClientRegistrationRepository;
import org.springframework.security.oauth2.client.web.reactive.function.client.ServletOAuth2AuthorizedClientExchangeFilterFunction;
import org.springframework.web.client.RestClient;

@Configuration
public class OAuth2ClientConfig {

    // OAuth2AuthorizedClientManager para contextos sin request HTTP (batch, schedulers)
    // Usar AuthorizedClientServiceOAuth2AuthorizedClientManager en lugar del basado en request
    @Bean
    public OAuth2AuthorizedClientManager authorizedClientManager(
            ClientRegistrationRepository registrations,
            OAuth2AuthorizedClientService clientService) {

        var provider = OAuth2AuthorizedClientProviderBuilder.builder()
            .clientCredentials()   // soporta el flujo client_credentials
            .refreshToken()        // renueva tokens expirados automáticamente
            .build();

        var manager = new AuthorizedClientServiceOAuth2AuthorizedClientManager(
            registrations, clientService);
        manager.setAuthorizedClientProvider(provider);

        return manager;
    }

    // RestClient con intercepción automática de tokens OAuth2
    @Bean
    public RestClient orderServiceClient(OAuth2AuthorizedClientManager authorizedClientManager) {
        // El exchange filter function añade el Bearer token en cada petición
        var oauth2Filter = new ServletOAuth2AuthorizedClientExchangeFilterFunction(
            authorizedClientManager);
        // Registración por defecto para este cliente
        oauth2Filter.setDefaultClientRegistrationId("order-service-cc");

        return RestClient.builder()
            .baseUrl("https://order-service.example.com")
            // Adaptar el filter de WebFlux a RestClient (Spring Boot 3.4+)
            .requestInterceptor((request, body, execution) -> {
                // Obtener token del AuthorizedClientManager
                var clientRequest = org.springframework.security.oauth2.client.web.reactive
                    .function.client.ServerOAuth2AuthorizedClientExchangeFilterFunction
                    .clientRegistrationId("order-service-cc");
                // Añadir header Authorization
                request.getHeaders().setBearerAuth(
                    getAccessToken(authorizedClientManager, "order-service-cc"));
                return execution.execute(request, body);
            })
            .build();
    }

    private String getAccessToken(OAuth2AuthorizedClientManager manager, String registrationId) {
        var request = org.springframework.security.oauth2.client.OAuth2AuthorizeRequest
            .withClientRegistrationId(registrationId)
            .principal("inventory-service")  // nombre del servicio como principal anónimo
            .build();
        var client = manager.authorize(request);
        return client.getAccessToken().getTokenValue();
    }
}
```

### Uso del RestClient con OAuth2 en un servicio

```java
package com.example.client;

import org.springframework.stereotype.Service;
import org.springframework.web.client.RestClient;

@Service
public class OrderServiceAdapter {

    private final RestClient orderServiceClient;

    public OrderServiceAdapter(RestClient orderServiceClient) {
        this.orderServiceClient = orderServiceClient;
    }

    public Order getOrder(String orderId) {
        // El token se añade automáticamente en la petición
        return orderServiceClient.get()
            .uri("/orders/{id}", orderId)
            .retrieve()
            .body(Order.class);
    }
}
```

> [CONCEPTO] `AuthorizedClientServiceOAuth2AuthorizedClientManager` (sin request HTTP) es el manager correcto para tareas batch, schedulers y contextos sin `HttpServletRequest`. El manager por defecto (`DefaultOAuth2AuthorizedClientManager`) requiere un request HTTP activo. Si se usa el manager por defecto en un scheduler, la obtención del token fallará con `IllegalStateException`.

## Tabla de elementos clave

| Componente / Propiedad | Descripción |
|---|---|
| `ClientRegistration` | Configuración de un cliente OAuth2: client-id, secret, grant-type, scopes, provider |
| `authorization-grant-type: client_credentials` | Flujo M2M: el servicio obtiene token con su propia identidad, sin usuario |
| `OAuth2AuthorizedClientManager` | Orquesta la obtención y renovación de tokens; dos implementaciones: con y sin request |
| `AuthorizedClientServiceOAuth2AuthorizedClientManager` | Manager sin request HTTP; usar en tareas batch, schedulers, eventos |
| `OAuth2AuthorizedClientProviderBuilder` | Builder de flujos soportados: `clientCredentials()`, `refreshToken()`, `authorizationCode()` |
| `OAuth2AuthorizeRequest.withClientRegistrationId()` | Solicita autorización para un registro concreto; `.principal()` es el nombre del cliente |
| `client-secret: ${CLIENT_SECRET}` | El secret nunca debe hardcodearse; usar variables de entorno o Vault |

## Buenas y malas prácticas

**Hacer:**
- Usar `AuthorizedClientServiceOAuth2AuthorizedClientManager` para todos los flujos Client Credentials en microservicios backend; el manager gestiona automáticamente la caché del token y su renovación antes de la expiración.
- Almacenar `CLIENT_SECRET` en un secret manager (Vault, AWS Secrets Manager, Kubernetes Secrets) y referenciarlo como `${CLIENT_SECRET}`; nunca en `application.yml` versionado.
- Configurar `scope` mínimo necesario en el registro: si el servicio solo necesita leer órdenes, no solicitar `orders:write`; el principio de mínimo privilegio aplica también a los tokens M2M.

**Evitar:**
- Obtener tokens manualmente con `RestTemplate` contra el endpoint `/oauth2/token` en lugar de usar `OAuth2AuthorizedClientManager`; el manager gestiona caché, renovación y reintentos automáticamente.
- Compartir un mismo `ClientRegistration` entre múltiples microservicios con distintos scopes: cada microservicio debe tener su propio `client-id` con los scopes específicos que necesita.
- Usar `grant-type: password` en microservicios: está deprecado en OAuth2.1 y expone credenciales de usuario al servicio intermediario.

---

← [8.3 Resource Server avanzado — claims, opaque tokens y multi-tenancy](sc-security-resource-server-advanced.md) | [Índice (README.md)](README.md) | [8.5 OAuth2 Client avanzado — WebClient, ReactiveManager y logout OIDC](sc-security-oauth2-client-advanced.md) →
