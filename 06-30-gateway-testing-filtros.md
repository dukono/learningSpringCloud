# 6.7.3 Tests de filtros y seguridad: @SpringBootTest + WireMock

← [6.7.2 Tests de rutas](./06-29-gateway-testing-rutas.md) | [Índice](./README.md) | [6.7.4 Tests de rate limiting →](./06-31-gateway-testing-rate-limiting.md)

---

Los filtros personalizados —autenticación JWT, propagación de cabeceras, validación HMAC— son la lógica más crítica del Gateway y la que más errores silenciosos puede producir: un filtro de autenticación mal configurado puede dejar rutas protegidas accesibles sin token, o bloquear rutas públicas que deberían funcionar sin autenticación. Testear esta capa requiere levantar el contexto completo del Gateway (`@SpringBootTest`) porque los filtros forman parte del pipeline reactivo que solo está disponible con el contexto completo, y WireMock para simular los servicios backend sin necesidad de levantarlos realmente.

> [PREREQUISITO] Dependencias `spring-boot-starter-test`, `wiremock-spring-boot` y perfil `test` con `application-test.yml` que deshabilita Redis y Eureka (ver 6.7.1).

## Qué verifican los tests de filtros

Los tests de filtros cubren el comportamiento del pipeline reactivo ante las distintas combinaciones de token y ruta: token válido que propaga identidad, token inválido que corta el flujo antes de llegar al backend, y rutas públicas que no requieren autenticación. El diagrama siguiente resume los escenarios obligatorios para cada filtro principal.

```
TESTS DE FILTROS verifican:

  JwtAuthFilter:
  ┌─────────────────────────────────────────────────────┐
  │ Escenario 1: Token válido                           │
  │   Cliente → [Bearer válido] → Gateway → Servicio   │
  │   Backend recibe X-User-Id, X-User-Roles           │
  │   Respuesta: 200                                   │
  │                                                     │
  │ Escenario 2: Token expirado / malformado            │
  │   Cliente → [Bearer inválido] → Gateway             │
  │   Gateway devuelve 401 (sin llamar al backend)     │
  │                                                     │
  │ Escenario 3: Sin token en ruta pública              │
  │   Cliente → [sin Authorization] → Gateway → Servicio│
  │   Respuesta: 200 (bypass del filtro)               │
  │                                                     │
  │ Escenario 4: Headers internos inyectados por cliente│
  │   Cliente → [X-User-Id: falso] → Gateway            │
  │   Backend recibe X-User-Id del JWT (no del cliente)│
  └─────────────────────────────────────────────────────┘

  CorrelationIdFilter:
  ┌─────────────────────────────────────────────────────┐
  │ Escenario 1: Sin X-Correlation-Id en la petición   │
  │   Backend recibe X-Correlation-Id generado por GW  │
  │                                                     │
  │ Escenario 2: Con X-Correlation-Id del cliente      │
  │   Backend recibe el mismo X-Correlation-Id          │
  └─────────────────────────────────────────────────────┘
```

## Tests del JwtAuthFilter

El `JwtAuthFilter` implementado en 6.6.2 valida el token JWT y propaga la identidad como headers internos. Los tests deben cubrir los cuatro escenarios del diagrama anterior. Para generar tokens de test se usa la misma librería JJWT con la misma clave secreta que el filtro de producción.

```java
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.security.Keys;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.cloud.contract.wiremock.AutoConfigureWireMock;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.springframework.test.web.reactive.server.WebTestClient;

import javax.crypto.SecretKey;
import java.nio.charset.StandardCharsets;
import java.util.Date;
import java.util.List;
import java.util.Map;

import static com.github.tomakehurst.wiremock.client.WireMock.*;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")
@AutoConfigureWireMock(port = 0)
class JwtAuthFilterTest {

    @Autowired
    private WebTestClient webTestClient;

    // La misma clave secreta que usa JwtAuthFilter (inyectada desde application-test.yml)
    @Value("${jwt.secret}")
    private String jwtSecret;

    @BeforeEach
    void resetStubs() {
        reset();
    }

    private String buildToken(String userId, List<String> roles, long expiresInMs) {
        SecretKey key = Keys.hmacShaKeyFor(jwtSecret.getBytes(StandardCharsets.UTF_8));
        return Jwts.builder()
            .subject(userId)
            .claim("roles", roles)
            .issuedAt(new Date())
            .expiration(new Date(System.currentTimeMillis() + expiresInMs))
            .signWith(key)
            .compact();
    }

    @Test
    void token_valido_enruta_al_servicio_y_propaga_identidad() {
        // Stub: el backend responde 200 si recibe los headers correctos
        stubFor(get(urlEqualTo("/productos"))
            .willReturn(ok("{\"items\":[]}")
                .withHeader("Content-Type", "application/json")));

        String token = buildToken("usr-123", List.of("ROLE_USER"), 60_000);

        webTestClient.get().uri("/api/v1/productos")
            .header("Authorization", "Bearer " + token)
            .exchange()
            .expectStatus().isOk();

        // El backend debe haber recibido los headers de identidad propagados
        verify(getRequestedFor(urlEqualTo("/productos"))
            .withHeader("X-User-Id", equalTo("usr-123"))
            .withHeader("X-User-Roles", containing("ROLE_USER")));
    }

    @Test
    void token_expirado_devuelve_401_sin_llamar_al_backend() {
        // Token con expiración en el pasado (-1000 ms)
        String expiredToken = buildToken("usr-123", List.of("ROLE_USER"), -1_000);

        webTestClient.get().uri("/api/v1/productos")
            .header("Authorization", "Bearer " + expiredToken)
            .exchange()
            .expectStatus().isUnauthorized();

        // El backend no debe haber recibido ninguna petición
        verify(0, getRequestedFor(urlEqualTo("/productos")));
    }

    @Test
    void token_malformado_devuelve_401() {
        webTestClient.get().uri("/api/v1/productos")
            .header("Authorization", "Bearer esto.no.es.un.jwt.valido")
            .exchange()
            .expectStatus().isUnauthorized();

        verify(0, getRequestedFor(urlEqualTo("/productos")));
    }

    @Test
    void sin_token_en_ruta_protegida_devuelve_401() {
        webTestClient.get().uri("/api/v1/productos")
            .exchange()
            .expectStatus().isUnauthorized();

        verify(0, getRequestedFor(urlEqualTo("/productos")));
    }

    @Test
    void ruta_publica_sin_token_pasa_directamente() {
        stubFor(get(urlEqualTo("/"))
            .willReturn(ok("{\"status\":\"ok\"}")
                .withHeader("Content-Type", "application/json")));

        // /api/public/** es una ruta pública definida en application-test.yml
        webTestClient.get().uri("/api/public/info")
            .exchange()
            .expectStatus().isOk();
    }

    @Test
    void header_x_user_id_inyectado_por_cliente_es_sobreescrito_por_el_filtro() {
        stubFor(get(urlEqualTo("/productos"))
            .willReturn(ok("{\"items\":[]}")
                .withHeader("Content-Type", "application/json")));

        String token = buildToken("usr-legit", List.of("ROLE_USER"), 60_000);

        // El cliente intenta inyectar una identidad falsa
        webTestClient.get().uri("/api/v1/productos")
            .header("Authorization", "Bearer " + token)
            .header("X-User-Id", "usr-atacante")   // intento de suplantación
            .exchange()
            .expectStatus().isOk();

        // El backend debe recibir el userId del JWT, no el inyectado por el cliente
        verify(getRequestedFor(urlEqualTo("/productos"))
            .withHeader("X-User-Id", equalTo("usr-legit"))   // del JWT
            .withoutHeader("X-User-Id"));                     // no el falso
        // Nota: withoutHeader verifica que no hay instancias adicionales del header
    }
}
```

> [CONCEPTO] `verify(0, getRequestedFor(...))` verifica que WireMock no recibió ninguna petición al backend. Esto confirma que el filtro cortó el flujo antes de que la petición llegara al servicio, que es el comportamiento correcto cuando el token es inválido.

## Tests del CorrelationIdFilter

El filtro de correlación genera un UUID como `X-Correlation-Id` si la petición no lleva uno, o propaga el que trae el cliente. Los tests verifican ambos comportamientos:

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")
@AutoConfigureWireMock(port = 0)
class CorrelationIdFilterTest {

    @Autowired
    private WebTestClient webTestClient;

    @BeforeEach
    void resetStubs() {
        reset();
        // Stub genérico para que el backend responda siempre 200
        stubFor(get(urlMatching("/.*"))
            .willReturn(ok("{}")
                .withHeader("Content-Type", "application/json")));
    }

    @Test
    void sin_correlation_id_el_filtro_genera_uno_y_lo_propaga() {
        webTestClient.get().uri("/api/public/salud")
            .exchange()
            .expectStatus().isOk();

        // El backend debe haber recibido un X-Correlation-Id generado automáticamente
        verify(getRequestedFor(urlMatching("/.*"))
            .withHeader("X-Correlation-Id", matching("[a-f0-9\\-]{36}"))); // UUID v4
    }

    @Test
    void con_correlation_id_del_cliente_el_filtro_lo_propaga_sin_modificar() {
        String correlationId = "req-12345-abcde";

        webTestClient.get().uri("/api/public/salud")
            .header("X-Correlation-Id", correlationId)
            .exchange()
            .expectStatus().isOk();

        verify(getRequestedFor(urlMatching("/.*"))
            .withHeader("X-Correlation-Id", equalTo(correlationId)));
    }
}
```

## Parámetros de configuración en application-test.yml

Los filtros en tests necesitan la misma configuración que en producción, pero con valores de test seguros:

| Propiedad | Valor de test | Descripción |
|---|---|---|
| `jwt.secret` | Cadena de 32+ caracteres fija | Clave reproducible para generar tokens en los tests |
| `gateway.security.enabled` | `true` (habilitar para tests de seguridad) | Si es `false`, los tests de autenticación no sirven |
| `gateway.security.public-paths` | `/api/public/**, /actuator/**` | Rutas que los tests de ruta pública usan |
| `app.backend.*.url` | `http://localhost:${wiremock.server.port}` | URLs del WireMock inyectadas por `@DynamicPropertySource` |

```yaml
# src/test/resources/application-test.yml (sección para tests de filtros)
jwt:
  secret: clave-de-test-de-32-caracteres-min   # exactamente 32+ chars para HS256

gateway:
  security:
    enabled: true
    public-paths:
      - /api/public/**
      - /actuator/health
```

> [ADVERTENCIA] La propiedad `gateway.security.enabled: false` del `application-test.yml` de la sección 6.7.1 desactiva el filtro JWT. Para los tests de seguridad se necesita activarlo. La solución es tener dos perfiles: `test` (seguridad desactivada, para tests de rutas) y `test-security` (seguridad activada, para tests de filtros). Usar `@ActiveProfiles("test-security")` en las clases de test de seguridad.

## Buenas y malas prácticas

Hacer:
- Cubrir siempre los cuatro escenarios del filtro de autenticación: token válido, token expirado, token malformado y ruta pública sin token. Una suite que solo prueba el happy path no protege contra regresiones en los casos de error.
- Usar la misma librería y clave para generar los tokens de test que usa el filtro en producción. Si el filtro usa JJWT con HS256, los tests deben generar tokens con JJWT y HS256, no con otra librería o algoritmo. Esto garantiza que los tests son fieles al comportamiento real.
- Verificar que el backend NO recibe peticiones en los escenarios de token inválido con `verify(0, getRequestedFor(...))`. Sin esta verificación, el test no confirma que el filtro cortó el flujo.

Evitar:
- No limpiar los stubs de WireMock entre tests (`reset()`). Los stubs de un test pueden responder a peticiones del siguiente, produciendo falsos positivos en las verificaciones.
- Crear un token de test con fecha de expiración muy larga (`999999999L` ms) sin documentarlo. Un token que expira en 30 años pasa los tests hoy pero puede producir confusión al revisarlo. Usar `60_000L` (1 minuto) para tokens válidos en tests.
- Combinar tests de rutas y tests de filtros en la misma clase de test. Los tests de rutas deben ser rápidos (sin lógica de seguridad) y los de filtros deben cubrir todos los escenarios de autenticación. Mezclarlos hace las clases grandes y difíciles de mantener.

---

← [6.7.2 Tests de rutas](./06-29-gateway-testing-rutas.md) | [Índice](./README.md) | [6.7.4 Tests de rate limiting →](./06-31-gateway-testing-rate-limiting.md)
