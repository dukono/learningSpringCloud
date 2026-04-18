# 8.9 Testing de seguridad OAuth2 — JWT mocks y Authorization Server simulado

← [sc-security-mtls.md](sc-security-mtls.md) | [Índice](README.md) | [sc-kubernetes-arquitectura.md](sc-kubernetes-arquitectura.md) →

---

## Introducción

Testear la seguridad OAuth2 en microservicios plantea un reto: los tests no deben depender de un Authorization Server real corriendo. Spring Security Test y la librería `spring-security-test` proporcionan utilidades para simular tokens JWT firmados, usuarios autenticados con scopes específicos y flujos OAuth2 completos sin infraestructura real. Para tests de integración que necesiten un AS completo, se puede usar WireMock o Testcontainers con Keycloak. El objetivo es verificar que los endpoints protegidos devuelven 401/403 correctamente y que `@PreAuthorize` aplica las reglas esperadas.

## @WithMockUser — tests unitarios simples

`@WithMockUser` es la utilidad más simple de Spring Security Test. Crea un `Authentication` en el `SecurityContext` con el usuario, roles y authorities especificadas. No genera JWT real — solo simula el contexto de seguridad. Es adecuado para tests de `@PreAuthorize` donde solo importan los roles, no los claims del token.

```java
package com.example.productservice.controller;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.security.test.context.support.WithMockUser;
import org.springframework.test.web.servlet.MockMvc;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@WebMvcTest(ProductController.class)
class ProductControllerWithMockUserTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    @WithMockUser(username = "user1", roles = {"USER"})
    void getProducts_withUserRole_returns200() throws Exception {
        mockMvc.perform(get("/products"))
            .andExpect(status().isOk());
    }

    @Test
    void getProducts_withoutAuth_returns401() throws Exception {
        mockMvc.perform(get("/products"))
            .andExpect(status().isUnauthorized());
    }

    @Test
    @WithMockUser(username = "user1", roles = {"USER"})
    void adminEndpoint_withUserRole_returns403() throws Exception {
        mockMvc.perform(get("/admin/products"))
            .andExpect(status().isForbidden());
    }
}
```

> [ADVERTENCIA] `@WithMockUser` no genera JWT. Si el código del controller accede a `@AuthenticationPrincipal Jwt jwt`, el `principal` no será un `Jwt` sino un `User`, y lanzará `ClassCastException`. Para tests que acceden a claims JWT, usar `SecurityMockMvcRequestPostProcessors.jwt()`.

## SecurityMockMvcRequestPostProcessors.jwt() — tests con JWT real

`SecurityMockMvcRequestPostProcessors.jwt()` crea un `JwtAuthenticationToken` real con claims configurables, sin necesidad de firmar el token criptográficamente. Es el método más completo para testear Resource Servers: permite configurar `sub`, `scope`, claims personalizados y authorities exactamente como llegarían de un AS real.

```java
package com.example.productservice.controller;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors;
import org.springframework.test.web.servlet.MockMvc;

import java.time.Instant;
import java.util.Map;

import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.jwt;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;

@WebMvcTest(ProductController.class)
class ProductControllerJwtTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    void getProducts_withReadScope_returns200() throws Exception {
        mockMvc.perform(get("/products")
            .with(jwt()
                .jwt(token -> token
                    .subject("user123")
                    .issuer("http://auth-server:9000")
                    .claim("scope", "products.read")
                    .claim("email", "user@example.com")
                    .expiresAt(Instant.now().plusSeconds(3600)))
                .authorities(new SimpleGrantedAuthority("SCOPE_products.read"))))
            .andExpect(status().isOk());
    }

    @Test
    void createProduct_withWriteScope_returns201() throws Exception {
        mockMvc.perform(post("/products")
            .contentType("application/json")
            .content("{\"name\":\"Widget\",\"price\":9.99}")
            .with(jwt().authorities(
                new SimpleGrantedAuthority("SCOPE_products.write"))))
            .andExpect(status().isCreated());
    }

    @Test
    void getProducts_withWrongScope_returns403() throws Exception {
        mockMvc.perform(get("/products")
            .with(jwt().authorities(
                new SimpleGrantedAuthority("SCOPE_orders.read"))))
            .andExpect(status().isForbidden());
    }

    @Test
    void getProducts_withExpiredToken_returns401() throws Exception {
        // Sin token — simulación de token expirado/ausente
        mockMvc.perform(get("/products"))
            .andExpect(status().isUnauthorized());
    }
}
```

## Mock OAuth2 Server con WireMock

Para tests de integración que necesitan un Authorization Server funcional (por ejemplo, cuando el microservicio usa `issuer-uri` y descarga el JWKS en el arranque), se puede usar WireMock para simular los endpoints del AS. La librería `spring-authorization-server` facilita la generación de JWTs reales firmados con claves RSA para tests.

```java
package com.example.productservice.integration;

import com.github.tomakehurst.wiremock.WireMockServer;
import com.github.tomakehurst.wiremock.core.WireMockConfiguration;
import org.junit.jupiter.api.AfterAll;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.http.HttpEntity;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpMethod;
import org.springframework.http.ResponseEntity;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;

import static com.github.tomakehurst.wiremock.client.WireMock.aResponse;
import static com.github.tomakehurst.wiremock.client.WireMock.get;
import static com.github.tomakehurst.wiremock.client.WireMock.urlEqualTo;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class ProductServiceIntegrationTest {

    static WireMockServer wireMockServer;

    @BeforeAll
    static void startWireMock() {
        wireMockServer = new WireMockServer(WireMockConfiguration.wireMockConfig().port(9999));
        wireMockServer.start();

        // Simular el endpoint de metadatos OIDC que Spring Boot descarga al arrancar
        wireMockServer.stubFor(get(urlEqualTo("/.well-known/openid-configuration"))
            .willReturn(aResponse()
                .withHeader("Content-Type", "application/json")
                .withBody("""
                    {
                        "issuer": "http://localhost:9999",
                        "jwks_uri": "http://localhost:9999/oauth2/jwks",
                        "token_endpoint": "http://localhost:9999/oauth2/token"
                    }
                """)));

        // Simular el JWKS endpoint con clave pública RSA para verificación
        wireMockServer.stubFor(get(urlEqualTo("/oauth2/jwks"))
            .willReturn(aResponse()
                .withHeader("Content-Type", "application/json")
                .withBody(TestJwtKeyHelper.getPublicJwks())));
    }

    @AfterAll
    static void stopWireMock() {
        wireMockServer.stop();
    }

    @DynamicPropertySource
    static void overrideProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.security.oauth2.resourceserver.jwt.issuer-uri",
            () -> "http://localhost:9999");
    }

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    void getProducts_withValidJwt_returns200() {
        String token = TestJwtKeyHelper.generateSignedToken("user123", "products.read");
        HttpHeaders headers = new HttpHeaders();
        headers.setBearerAuth(token);
        ResponseEntity<String> response = restTemplate.exchange(
            "/products", HttpMethod.GET, new HttpEntity<>(headers), String.class);
        org.assertj.core.api.Assertions.assertThat(response.getStatusCode().value()).isEqualTo(200);
    }
}
```

## Testcontainers con Keycloak como Authorization Server real

Para tests de integración de máxima fidelidad, se puede levantar un Keycloak real mediante Testcontainers. Es más lento que WireMock pero verifica el flujo completo de autenticación.

```java
package com.example.productservice.integration;

import dasniko.testcontainers.keycloak.KeycloakContainer;
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.HttpEntity;
import org.springframework.http.HttpHeaders;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
class ProductServiceKeycloakTest {

    @Container
    static KeycloakContainer keycloak = new KeycloakContainer("quay.io/keycloak/keycloak:24.0")
        .withRealmImportFile("/keycloak-realm-export.json");

    @DynamicPropertySource
    static void overrideProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.security.oauth2.resourceserver.jwt.issuer-uri",
            () -> keycloak.getAuthServerUrl() + "/realms/test-realm");
    }

    @Test
    void fullOAuth2Flow_withRealKeycloak() {
        // Obtener token real de Keycloak
        String token = obtainTokenFromKeycloak(
            keycloak.getAuthServerUrl(), "test-client", "secret", "products.read");
        // ... rest of test
    }
}
```

## Troubleshooting: 401 vs 403 y errores comunes

Distinguir entre 401 y 403 es fundamental para diagnosticar problemas de seguridad. El código 401 (Unauthorized) significa que la autenticación falló o falta; el código 403 (Forbidden) significa que la autenticación fue exitosa pero el usuario no tiene permisos suficientes.

| Error | Código | Causa probable | Diagnóstico |
|-------|--------|----------------|-------------|
| Token ausente | 401 | No se envió `Authorization: Bearer` | Verificar header |
| Token inválido/corrupto | 401 | Firma no verificable | Verificar `jwk-set-uri` |
| Token expirado | 401 | Claim `exp` en el pasado | Verificar reloj del AS |
| Issuer no coincide | 401 | Claim `iss` != `issuer-uri` configurado | Verificar `issuer-uri` |
| Scope insuficiente | 403 | Token válido pero sin el scope requerido | Revisar `@PreAuthorize` y prefijos |
| `hasRole` con scope | 403 | `hasRole("read")` busca `ROLE_read`, el token tiene `SCOPE_read` | Cambiar a `hasAuthority("SCOPE_read")` |

> [EXAMEN] 401 = fallo de autenticación (token ausente, expirado o inválido). 403 = fallo de autorización (token válido pero sin permisos). Confundirlos es el error más frecuente al depurar seguridad OAuth2.

## Logging DEBUG de Spring Security

Activar el logging DEBUG de Spring Security revela el pipeline de filtros y el motivo exacto de cada 401/403. Es la herramienta más útil para diagnosticar problemas de seguridad en desarrollo.

```yaml
logging:
  level:
    org.springframework.security: DEBUG
    org.springframework.security.oauth2: TRACE
```

Con este nivel, los logs muestran exactamente qué filtro procesó la request, si el token fue validado correctamente, y qué autoridades se extrajeron del JWT. Buscar líneas como `BearerTokenAuthenticationFilter`, `JwtDecoder`, `JwtAuthenticationProvider` en los logs.

## CORS en flujo OAuth2 y interacción con Circuit Breaker

El CORS (Cross-Origin Resource Sharing) puede interferir con el flujo de redirect OAuth2 cuando el frontend y el AS están en dominios diferentes. La preflight request OPTIONS del navegador debe responder con los headers CORS correctos antes de que el browser envíe el token. Configurar CORS en el Gateway o en el Resource Server para permitir el header `Authorization`.

Un 401/403 de un microservicio downstream puede activar el Circuit Breaker de Resilience4j si está configurado para abrir con errores HTTP. Es importante configurar el Circuit Breaker para que NO reaccione ante 401/403: estos son errores de negocio (autorización), no fallos de red. Solo los timeouts y errores de conexión deben abrir el circuito.

```yaml
resilience4j:
  circuitbreaker:
    instances:
      inventory-service:
        ignore-exceptions:
          - org.springframework.security.access.AccessDeniedException
        record-failure-predicate: com.example.NetworkFailurePredicate
        # No registrar como fallo los 401/403
```

## Buenas y malas prácticas

Hacer:
- Usar `SecurityMockMvcRequestPostProcessors.jwt()` para tests de Resource Server que usan `@AuthenticationPrincipal Jwt`.
- Probar explícitamente tanto el caso 401 (sin token) como el 403 (token sin permisos) para cada endpoint protegido.
- Activar logging DEBUG de Spring Security solo en dev/test — es muy verboso para producción.
- Usar WireMock para simular el JWKS endpoint en tests de integración sin AS real.

Evitar:
- Usar `@WithMockUser` cuando el código accede a `@AuthenticationPrincipal Jwt` — lanza `ClassCastException`.
- Deshabilitar la seguridad completa en tests de integración (`@TestConfiguration` que permite todo) — los tests no verificarán nada.
- Ignorar el test del caso de token expirado — es el fallo más frecuente en producción.

## Verificación y práctica

```bash
# Test rápido de 401 sin token
curl -v http://localhost:8080/products 2>&1 | grep "< HTTP"
# < HTTP/1.1 401

# Test de 403 con scope incorrecto
TOKEN=$(curl -s -X POST http://localhost:9000/oauth2/token \
  -d "grant_type=client_credentials&scope=orders.read" \
  -u "test-client:secret" | jq -r '.access_token')
curl -v -H "Authorization: Bearer $TOKEN" http://localhost:8080/products 2>&1 | grep "< HTTP"
# < HTTP/1.1 403

# Decodificar el token para verificar scopes
echo $TOKEN | cut -d. -f2 | base64 -d 2>/dev/null | jq '.scope'
```

**Preguntas estilo examen VMware Spring Professional:**

1. ¿Por qué `@WithMockUser` no es adecuado para testear un controller que usa `@AuthenticationPrincipal Jwt`? ¿Qué alternativa debe usarse?

2. Un endpoint protegido con `@PreAuthorize("hasAuthority('SCOPE_products.read')")` devuelve 403 en un test con `.with(jwt().authorities(new SimpleGrantedAuthority("SCOPE_products.read")))`. ¿Cuál es la causa más probable?

3. ¿Cuál es la diferencia entre un error 401 y un error 403 en el contexto de Spring Security con OAuth2? Dar un ejemplo concreto de cada uno con tokens JWT.

---

← [sc-security-mtls.md](sc-security-mtls.md) | [Índice](README.md) | [sc-kubernetes-arquitectura.md](sc-kubernetes-arquitectura.md) →
```
