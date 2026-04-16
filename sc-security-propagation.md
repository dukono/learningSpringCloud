# 8.8 Propagación de identidad entre microservicios síncronos

← [8.7 Autorización por URL y métodos con Spring Security](sc-security-authorization.md) | [Índice (README.md)](README.md) | [8.9 Propagación de identidad en contextos asíncronos y mensajería](sc-security-propagation-async.md) →

---

Cuando un microservicio llama a otro mediante REST o Feign, el token JWT del usuario original debe propagarse en el header `Authorization` del request downstream — de lo contrario, el servicio destino recibe un request sin autenticar y responde 401. Existen dos estrategias: Token Relay (propagar el token del usuario tal cual) y Client Credentials (el servicio obtiene su propio token con su identidad). La elección depende del modelo de autorización: si el servicio destino necesita saber quién es el usuario final, usar Token Relay; si solo necesita saber que la petición viene de un servicio autorizado, usar Client Credentials. En muchos sistemas ambos se combinan: el Gateway hace Token Relay del token del usuario, y los microservicios internos se autentican mutuamente con Client Credentials.

> [PREREQUISITO] Requiere `spring-boot-starter-oauth2-resource-server` en el servicio que recibe y `spring-boot-starter-oauth2-client` en el servicio que reenvía. Para OpenFeign: `spring-cloud-starter-openfeign`.

## Estrategias de propagación de identidad

```
┌──────────────────────────────────────────────────────────────────┐
│  Token Relay (propagar token del usuario)                         │
│                                                                   │
│  Client ──→ Service A ──→ Service B                              │
│   Bearer T    Bearer T      Bearer T    ← mismo token            │
│                                                                   │
│  Ventaja: Service B conoce la identidad del usuario final        │
│  Riesgo: el token del usuario puede tener scopes inadecuados     │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  Client Credentials (token de servicio)                           │
│                                                                   │
│  Client ──→ Service A ──→ Service B                              │
│   Bearer T    Bearer S      Bearer S    ← token de Service A     │
│                                                                   │
│  Ventaja: Service B solo autoriza servicios conocidos            │
│  Limitación: Service B no conoce quién es el usuario final       │
└──────────────────────────────────────────────────────────────────┘
```

## Ejemplo central: Token Relay con RestClient y Feign

### Token Relay manual con RestClient (servlet)

```java
package com.example.propagation;

import org.springframework.http.HttpHeaders;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationToken;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestClient;

@Component
public class InventoryServiceClient {

    private final RestClient restClient;

    public InventoryServiceClient(RestClient.Builder builder) {
        this.restClient = builder
            .baseUrl("https://inventory-service.example.com")
            .requestInterceptor((request, body, execution) -> {
                // Extraer el token JWT del SecurityContext del hilo actual
                var auth = SecurityContextHolder.getContext().getAuthentication();
                if (auth instanceof JwtAuthenticationToken jwtAuth) {
                    String tokenValue = jwtAuth.getToken().getTokenValue();
                    request.getHeaders().set(HttpHeaders.AUTHORIZATION, "Bearer " + tokenValue);
                }
                return execution.execute(request, body);
            })
            .build();
    }

    public InventoryItem getInventory(String productId) {
        return restClient.get()
            .uri("/inventory/{id}", productId)
            .retrieve()
            .body(InventoryItem.class);
    }
}
```

### Token Relay con OpenFeign mediante RequestInterceptor

```java
package com.example.propagation;

import feign.RequestInterceptor;
import feign.RequestTemplate;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationToken;
import org.springframework.stereotype.Component;

// RequestInterceptor de Feign: se aplica a todos los clientes Feign del contexto
@Component
public class JwtFeignRequestInterceptor implements RequestInterceptor {

    @Override
    public void apply(RequestTemplate template) {
        var auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth instanceof JwtAuthenticationToken jwtAuth) {
            String tokenValue = jwtAuth.getToken().getTokenValue();
            template.header("Authorization", "Bearer " + tokenValue);
        }
        // Si no hay autenticación activa (ej: scheduler), no se añade header
        // El servicio destino rechazará la petición con 401 — comportamiento correcto
    }
}
```

```java
package com.example.propagation;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;

// El interceptor JwtFeignRequestInterceptor se aplica automáticamente
@FeignClient(name = "inventory-service", url = "${inventory.service.url}")
public interface InventoryFeignClient {

    @GetMapping("/inventory/{productId}")
    InventoryItem getInventory(@PathVariable String productId);
}
```

### Client Credentials para comunicación servicio-a-servicio

```java
package com.example.propagation;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.oauth2.client.AuthorizedClientServiceOAuth2AuthorizedClientManager;
import org.springframework.security.oauth2.client.OAuth2AuthorizedClientManager;
import org.springframework.security.oauth2.client.OAuth2AuthorizedClientProviderBuilder;
import org.springframework.security.oauth2.client.OAuth2AuthorizedClientService;
import org.springframework.security.oauth2.client.OAuth2AuthorizeRequest;
import org.springframework.security.oauth2.client.registration.ClientRegistrationRepository;
import org.springframework.web.client.RestClient;

@Configuration
public class ServiceClientConfig {

    @Bean
    public RestClient warehouseServiceClient(
            OAuth2AuthorizedClientManager clientManager) {

        return RestClient.builder()
            .baseUrl("https://warehouse-service.example.com")
            .requestInterceptor((request, body, execution) -> {
                // Obtener token de servicio (Client Credentials)
                var authRequest = OAuth2AuthorizeRequest
                    .withClientRegistrationId("warehouse-service-cc")
                    .principal("order-service")
                    .build();
                var authorizedClient = clientManager.authorize(authRequest);
                if (authorizedClient != null) {
                    request.getHeaders().setBearerAuth(
                        authorizedClient.getAccessToken().getTokenValue());
                }
                return execution.execute(request, body);
            })
            .build();
    }

    @Bean
    public OAuth2AuthorizedClientManager serviceClientManager(
            ClientRegistrationRepository registrations,
            OAuth2AuthorizedClientService clientService) {
        var provider = OAuth2AuthorizedClientProviderBuilder.builder()
            .clientCredentials()
            .build();
        var manager = new AuthorizedClientServiceOAuth2AuthorizedClientManager(
            registrations, clientService);
        manager.setAuthorizedClientProvider(provider);
        return manager;
    }
}
```

> [ADVERTENCIA] `SecurityContextHolder.getContext().getAuthentication()` en un servlet `RequestInterceptor` de Feign asume que el thread que ejecuta el interceptor es el mismo thread del request HTTP original. Si Feign usa un executor pool con threads diferentes, el `SecurityContext` puede ser null. Para garantizar la propagación en pools de threads, usar `DelegatingSecurityContextExecutor` o el patrón `SecurityContextHolder.MODE_INHERITABLETHREADLOCAL`.

## Tabla de elementos clave

| Componente | Descripción |
|---|---|
| `JwtAuthenticationToken.getToken().getTokenValue()` | Extrae el JWT string del `Authentication` actual para propagarlo en outgoing requests |
| `SecurityContextHolder.getContext().getAuthentication()` | Accede al principal del hilo actual; válido en threads de request servlet |
| `RequestInterceptor` (Feign) | Interceptor que se aplica a todas las peticiones de todos los clientes Feign en el contexto |
| `RestClient.requestInterceptor(...)` | Interceptor que se aplica a todas las peticiones de un `RestClient` concreto |
| `DelegatingSecurityContextExecutor` | Wraps un `Executor` para propagar el `SecurityContext` a threads del pool |
| `OAuth2AuthorizeRequest.withClientRegistrationId()` | Solicita autorización para un registro Client Credentials; devuelve el token con caché |
| `SecurityContextHolder.MODE_INHERITABLETHREADLOCAL` | Modo que permite que threads hijo hereden el `SecurityContext` del thread padre |

## Buenas y malas prácticas

**Hacer:**
- Usar `DelegatingSecurityContextExecutor` cuando se usa un pool de threads para llamadas downstream (async, @Async, virtual threads en executor pool): garantiza que el SecurityContext del request original está disponible en el thread del pool.
- Distinguir explícitamente entre Token Relay (token del usuario) y Client Credentials (token del servicio) en la arquitectura; mezclarlos implícitamente lleva a bugs donde un servicio propaga un token de servicio como si fuera del usuario.
- Centralizar el `RequestInterceptor` de Feign en un único `@Component` en lugar de configurarlo por cliente; un solo interceptor protege todos los clientes del contexto.

**Evitar:**
- Almacenar el token JWT en una variable de instancia o de clase para reutilizarlo entre requests; el token es específico del request y puede expirar entre llamadas — siempre obtenerlo del `SecurityContext` o del `OAuth2AuthorizedClientManager`.
- Propagar el token de usuario a servicios que no necesitan la identidad del usuario final; si el servicio downstream solo necesita saber que la petición viene de un servicio autorizado, usar Client Credentials para aplicar mínimo privilegio.
- Ignorar el 401 downstream como "error de red"; un 401 en una llamada interna entre microservicios siempre indica un problema de propagación de identidad — debe ser visible en los logs con el suficiente contexto para diagnosticar.

---

← [8.7 Autorización por URL y métodos con Spring Security](sc-security-authorization.md) | [Índice (README.md)](README.md) | [8.9 Propagación de identidad en contextos asíncronos y mensajería](sc-security-propagation-async.md) →
