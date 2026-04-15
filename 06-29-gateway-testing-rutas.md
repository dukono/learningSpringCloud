# 6.7.2 Tests de rutas con RouteLocator Java DSL

← [6.7.1 Estrategia de testing](./06-28-gateway-testing-estrategia.md) | [Índice](./README.md) | [6.7.3 Tests de filtros y seguridad →](./06-30-gateway-testing-filtros.md)

---

Cuando las rutas del Gateway se definen en Java DSL mediante `RouteLocator`, los tests deben verificar que la lógica de enrutamiento funciona antes de arrancar la aplicación completa. `@SpringBootTest` con una configuración de rutas en `@TestConfiguration` permite declarar un `RouteLocator` específico para el test, independiente del `application.yml`, lo que hace el test más preciso: se prueba exactamente la ruta que se define en el test, sin que las demás rutas de la aplicación interfieran. WireMock actúa como backend, y `verify(getRequestedFor(...))` confirma que el path transformado llegó correctamente al servicio de destino.

> [PREREQUISITO] Dependencias `spring-boot-starter-test` y `wiremock-spring-boot`. El perfil `test` de `application-test.yml` debe deshabilitar Eureka y Redis (ver 6.7.1).

## Qué verifican los tests de rutas con RouteLocator

Los tests de rutas se centran en cuatro comportamientos del enrutador: que el predicate de path coincide con las URLs correctas, que los filtros de transformación modifican el path antes de enviarlo al backend, que los predicates de método HTTP filtran los métodos no permitidos, y que el orden de las rutas resuelve correctamente los conflictos de prefijo. El diagrama siguiente muestra qué verifica cada categoría y qué herramienta de aserción se usa.

```
TESTS DE RUTAS verifican:

  1. Path matching (predicate)
     /api/v1/productos/123  → predicate Path coincide → WireMock recibe petición
     /ruta/desconocida      → ninguna ruta coincide   → 404

  2. Transformación de path (StripPrefix, RewritePath)
     Cliente envía:  GET /api/v1/productos/123
     Gateway aplica: StripPrefix=2
     Backend recibe: GET /productos/123
     ↑ se verifica con verify(getRequestedFor(urlEqualTo("/productos/123")))

  3. Restricción de método HTTP
     GET  /api/v1/productos/123  → coincide con ruta GET    → 200
     POST /api/v1/productos      → no coincide con Method=GET → 404/405

  4. Orden de rutas con el mismo prefijo
     /api/v1/admin/usuarios (order=1) tiene prioridad sobre /api/v1/** (order=10)
     El test confirma que el backend de admin recibe la petición, no el genérico
```

## Implementación completa

La clave del enfoque con `RouteLocator` es declarar las rutas en una clase `@TestConfiguration` anidada. Spring carga esta configuración en lugar del YAML y construye exactamente las rutas que el test necesita verificar.

```java
import com.github.tomakehurst.wiremock.client.WireMock;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.cloud.contract.wiremock.AutoConfigureWireMock;
import org.springframework.cloud.gateway.route.RouteLocator;
import org.springframework.cloud.gateway.route.builder.RouteLocatorBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.springframework.test.web.reactive.server.WebTestClient;

import static com.github.tomakehurst.wiremock.client.WireMock.*;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")
@AutoConfigureWireMock(port = 0)
class ProductosRouteLocatorTest {

    @Autowired
    private WebTestClient webTestClient;

    // El RouteLocator del test apunta al WireMock; el puerto se inyecta
    // vía @DynamicPropertySource antes de que Spring construya el contexto
    @DynamicPropertySource
    static void wireMockProperties(DynamicPropertyRegistry registry) {
        // La propiedad se usa en el bean routeLocator() de abajo
        registry.add("test.backend.url",
            () -> "http://localhost:${wiremock.server.port}");
    }

    @BeforeEach
    void resetWireMock() {
        WireMock.reset();
    }

    // ---- Tests de matching y transformación de path ----

    @Test
    void stripPrefix_elimina_segmentos_correctos() {
        // Backend espera /productos/123 (sin los dos primeros segmentos /api/v1)
        stubFor(get(urlEqualTo("/productos/123"))
            .willReturn(ok("""
                {"id": 123, "nombre": "Camisa blanca"}
                """)
                .withHeader("Content-Type", "application/json")));

        webTestClient.get().uri("/api/v1/productos/123")
            .exchange()
            .expectStatus().isOk()
            .expectBody()
                .jsonPath("$.id").isEqualTo(123);

        // Verificación crítica: el backend recibió el path transformado, no el original
        verify(getRequestedFor(urlEqualTo("/productos/123")));
    }

    @Test
    void rewritePath_transforma_path_con_regex() {
        // Backend espera /api/usuarios/42 según la regex de RewritePath
        stubFor(get(urlEqualTo("/api/usuarios/42"))
            .willReturn(ok("""
                {"id": 42, "nombre": "Ana García"}
                """)
                .withHeader("Content-Type", "application/json")));

        webTestClient.get().uri("/users/42")
            .exchange()
            .expectStatus().isOk();

        verify(getRequestedFor(urlEqualTo("/api/usuarios/42")));
    }

    @Test
    void path_sin_ruta_coincidente_devuelve_404() {
        webTestClient.get().uri("/ruta/completamente/desconocida")
            .exchange()
            .expectStatus().isNotFound();
    }

    @Test
    void metodo_incorrecto_no_coincide_con_ruta_solo_get() {
        // La ruta de productos tiene predicate Method=GET
        webTestClient.post().uri("/api/v1/productos")
            .bodyValue("{\"nombre\": \"test\"}")
            .header("Content-Type", "application/json")
            .exchange()
            .expectStatus().isNotFound();

        // El backend no debe haber recibido nada
        verify(0, postRequestedFor(urlEqualTo("/productos")));
    }

    @Test
    void ruta_admin_con_order_menor_tiene_prioridad() {
        // Stub para admin (order=1) — si llegara a la ruta genérica daría 404
        stubFor(get(urlEqualTo("/admin/usuarios"))
            .willReturn(ok("{\"admin\": true}")
                .withHeader("Content-Type", "application/json")));

        webTestClient.get().uri("/api/v1/admin/usuarios")
            .header("X-Internal-Token", "a3f9c1b8d2e4f6a0b1c2d3e4f5a6b7c8d9e0f1a2")
            .exchange()
            .expectStatus().isOk();

        // Verificar que llegó al backend de admin, no al genérico
        verify(getRequestedFor(urlEqualTo("/admin/usuarios")));
    }

    // ---- Configuración: rutas definidas en Java DSL, no en YAML ----

    @TestConfiguration
    static class TestRoutesConfig {

        // El puerto de WireMock se resuelve a través de la propiedad inyectada.
        // El bean recibe el valor ya resuelto via @Value; no se puede usar
        // @DynamicPropertySource directamente dentro de @TestConfiguration,
        // por eso se externaliza como propiedad.
        @Bean
        RouteLocator testRouteLocator(
                RouteLocatorBuilder builder,
                @org.springframework.beans.factory.annotation.Value(
                    "${test.backend.url}") String backendUrl) {

            return builder.routes()

                // Ruta de productos: StripPrefix=2 + solo GET/HEAD
                .route("productos-route", r -> r
                    .path("/api/v1/productos/**")
                    .and().method("GET", "HEAD")
                    .filters(f -> f.stripPrefix(2))
                    .uri(backendUrl))

                // Ruta admin: mayor prioridad (order=1), requiere header interno
                .route("admin-route", r -> r
                    .order(1)
                    .path("/api/v1/admin/**")
                    .and().header("X-Internal-Token", "[a-f0-9]{40}")
                    .filters(f -> f.stripPrefix(2))
                    .uri(backendUrl))

                // Ruta con RewritePath: convierte /users/{id} → /api/usuarios/{id}
                .route("usuarios-route", r -> r
                    .path("/users/**")
                    .filters(f -> f.rewritePath(
                        "/users/(?<id>.*)",
                        "/api/usuarios/${id}"))
                    .uri(backendUrl))

                .build();
        }
    }
}
```

> [CONCEPTO] Definir las rutas en `@TestConfiguration` con `RouteLocator` tiene una ventaja sobre el YAML de test: los predicates se verifican en el mismo lenguaje que el código de producción cuando las rutas se definen en Java DSL. Un error tipográfico en un predicate de YAML no produce error de compilación; un error en el `RouteLocatorBuilder` sí.

## Parámetros clave de RouteLocatorBuilder en tests

La tabla siguiente relaciona cada método del `RouteLocatorBuilder` con su equivalente en YAML, lo que facilita traducir rutas definidas en `application.yml` a la Java DSL cuando se construye el `@TestConfiguration`. Los métodos más relevantes en el contexto de testing son `.order()`, `.and()` y `.uri()`, porque son los que determinan qué ruta gana cuando hay ambigüedad y a qué servidor apunta la petición durante el test.

| Método | Equivalente YAML | Descripción |
|---|---|---|
| `.path("/api/v1/**")` | `- Path=/api/v1/**` | Predicate de path con wildcard |
| `.method("GET", "HEAD")` | `- Method=GET,HEAD` | Predicate de método HTTP |
| `.header("nombre", "regex")` | `- Header=nombre, regex` | Predicate de header con regex |
| `.order(n)` | `order: n` | Prioridad de la ruta (menor = mayor prioridad) |
| `.filters(f -> f.stripPrefix(2))` | `- StripPrefix=2` | Filtro de transformación de path |
| `.filters(f -> f.rewritePath(...))` | `- RewritePath=...` | Filtro de reescritura con regex |
| `.uri(url)` | `uri: http://host` | Backend de destino |

> [EXAMEN] El método `.and()` en el `RouteLocatorBuilder` combina predicates con lógica AND: la ruta solo coincide si todos los predicates son verdaderos. Si se encadenan sin `.and()`, el último predicate sobreescribe el anterior. Este comportamiento es la fuente más frecuente de rutas que nunca coinciden cuando se usa la Java DSL.

## Buenas y malas prácticas

Hacer:
- Usar `verify(0, getRequestedFor(...))` para confirmar que el backend no recibió peticiones en los tests de 404 y método incorrecto. Sin esta verificación, el test solo comprueba el código de respuesta pero no si el Gateway llegó a llamar al backend.
- Declarar el `RouteLocator` de test en una clase `@TestConfiguration` anidada estática. Esto hace que la configuración sea visible y mantenible junto al test que la usa, sin contaminar el contexto de otros tests.
- Inyectar la URL del backend como `@Value` en el bean `RouteLocator` usando `@DynamicPropertySource` para el puerto aleatorio de WireMock. Hardcodear el puerto hace los tests frágiles en CI.

Evitar:
- Usar `WireMock.stubFor(get(urlMatching(".*")).willReturn(ok(...)))` como stub genérico para todas las peticiones. Un stub que responde 200 a cualquier path hace que los tests de transformación de path siempre pasen aunque `StripPrefix` tenga el valor incorrecto.
- Incluir más de tres o cuatro rutas en el `@TestConfiguration`. Cada ruta añadida aumenta la complejidad del contexto de test y dificulta identificar cuál ruta está fallando. Si se necesitan más rutas, separar en clases de test distintas por grupo funcional.

---

← [6.7.1 Estrategia de testing](./06-28-gateway-testing-estrategia.md) | [Índice](./README.md) | [6.7.3 Tests de filtros y seguridad →](./06-30-gateway-testing-filtros.md)
