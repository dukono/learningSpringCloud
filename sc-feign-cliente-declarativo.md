# 4.2 Declaración de @FeignClient — interfaz, atributos y mapeo de métodos HTTP

<- [4.1 Setup y bootstrap](sc-feign-setup.md) | [Índice](README.md) | [4.3 Configuración por clase Java](sc-feign-configuracion-java.md) ->

---

## Introducción

La declaración de un cliente Feign es el acto de traducir un contrato HTTP externo a tipos Java. En lugar de construir manualmente la petición HTTP, el desarrollador escribe una interfaz anotada donde cada método describe un endpoint: el verbo HTTP, la ruta, los parámetros de query, las cabeceras y el cuerpo. Feign genera el proxy en tiempo de arranque y convierte cada invocación al método en la petición HTTP correspondiente.

Sin esta capa declarativa, cada punto de integración entre servicios requiere código de infraestructura repetitivo y propenso a errores: construcción de URLs, serialización manual, gestión de `PathVariable` en strings, etc. Con `@FeignClient`, la interfaz Java es el único artefacto de integración; el contrato es visible, testeable y refactorizable.

> [PREREQUISITO] El módulo debe estar inicializado con `@EnableFeignClients` según se describe en 4.1. Sin ese paso, las interfaces `@FeignClient` no se registran como beans de Spring.

---

## Diagrama: anatomía de una interfaz @FeignClient

El siguiente diagrama muestra la correspondencia entre los elementos de la interfaz Feign y la petición HTTP que genera el proxy.

```
@FeignClient(
  name        = "inventory-service",   ←── nombre lógico (Eureka/LoadBalancer)
  path        = "/api/inventory",      ←── prefijo de ruta para todos los métodos
  configuration = FeignAuthConfig.class, ←── beans custom: Encoder, Interceptor…
  fallback    = InventoryFallback.class, ←── clase de fallback (requiere CB activo)
  contextId   = "inventoryFeign"       ←── ID único (necesario con 2+ clientes al mismo name)
)
public interface InventoryClient {

  @GetMapping("/{productId}")          ←── verbo + ruta relativa al path del cliente
  InventoryResponse getInventory(
      @PathVariable("productId") String id,   ←── segmento de ruta
      @RequestParam("warehouse") String wh    ←── parámetro de query string
  );

  @PostMapping("/reserve")
  void reserve(@RequestBody ReservationRequest request);  ←── cuerpo JSON

  @DeleteMapping("/{productId}")
  ResponseEntity<Void> release(
      @PathVariable("productId") String id,
      @RequestHeader("X-Correlation-Id") String correlationId  ←── cabecera HTTP
  );
}
```

---

## Ejemplo central

El siguiente ejemplo muestra un cliente Feign completo para un servicio de catálogo de productos. Cubre los casos de uso más frecuentes: GET con `@PathVariable` y `@RequestParam`, POST con `@RequestBody`, DELETE con `@RequestHeader`, y el uso de `contextId` para dos clientes al mismo servicio con configuraciones distintas.

```java
package com.example.orderservice.client;

import com.example.orderservice.config.FeignAuthConfig;
import com.example.orderservice.config.FeignReadOnlyConfig;
import com.example.orderservice.dto.ProductDto;
import com.example.orderservice.dto.ReservationRequest;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

/**
 * Cliente de escritura: usa configuración con autenticación (FeignAuthConfig).
 * contextId diferencia este bean del cliente de solo lectura al mismo servicio.
 */
@FeignClient(
        name        = "catalog-service",
        path        = "/api/products",
        configuration = FeignAuthConfig.class,
        contextId   = "catalogWrite"
)
public interface CatalogWriteClient {

    @PostMapping("/reserve")
    ResponseEntity<Void> reserve(@RequestBody ReservationRequest request);

    @DeleteMapping("/{productId}/reservation")
    void cancelReservation(
            @PathVariable("productId") String productId,
            @RequestHeader("X-Idempotency-Key") String idempotencyKey
    );
}

/**
 * Cliente de solo lectura: sin autenticación, timeouts más cortos (FeignReadOnlyConfig).
 * Mismo name que CatalogWriteClient pero contextId distinto → dos beans independientes.
 */
@FeignClient(
        name        = "catalog-service",
        path        = "/api/products",
        configuration = FeignReadOnlyConfig.class,
        contextId   = "catalogRead",
        decode404   = true            // 404 se decodifica como null en lugar de FeignException
)
public interface CatalogReadClient {

    @GetMapping("/{productId}")
    ProductDto getProduct(@PathVariable("productId") String productId);

    @GetMapping
    List<ProductDto> listProducts(
            @RequestParam("category") String category,
            @RequestParam(value = "page", defaultValue = "0") int page,
            @RequestParam(value = "size", defaultValue = "20") int size
    );
}
```

> [CONCEPTO] `contextId` es obligatorio cuando dos interfaces `@FeignClient` tienen el mismo atributo `name`. Sin él, Spring detecta dos `FeignClientFactoryBean` con el mismo nombre de bean y lanza `BeanDefinitionOverrideException` en arranque.

> [ADVERTENCIA] `decode404 = true` hace que una respuesta HTTP 404 sea procesada por el `Decoder` en lugar de por el `ErrorDecoder`. El método devolverá `null` si el tipo de retorno es un objeto, o una lista vacía si es `Collection`. Usar solo cuando el 404 es un resultado semántico válido del negocio, no un error.

---

## Tabla de elementos clave

Los atributos de `@FeignClient` y las anotaciones de mapeo que un senior debe dominar para entrevista.

| Elemento | Tipo | Default | Descripción |
|---|---|---|---|
| `name` | `String` | — | Nombre lógico del servicio; resuelto por LoadBalancer/Eureka |
| `url` | `String` | — | URL fija; desactiva la resolución por LoadBalancer |
| `path` | `String` | `""` | Prefijo de ruta aplicado a todos los métodos del cliente |
| `configuration` | `Class[]` | `FeignClientsConfiguration` | Clase(s) `@Configuration` locales del cliente |
| `fallback` | `Class` | `void.class` | Clase de fallback estática; requiere `feign.circuitbreaker.enabled=true` |
| `fallbackFactory` | `Class` | `void.class` | `FallbackFactory<T>` para fallback con acceso a la causa |
| `contextId` | `String` | valor de `name` | ID único del bean; obligatorio con múltiples clientes al mismo `name` |
| `decode404` | `boolean` | `false` | Si `true`, trata 404 como respuesta decodificable, no como error |
| `@GetMapping` / `@PostMapping` … | anotación | — | Mapeo de método al verbo HTTP y ruta relativa |
| `@PathVariable` | anotación | — | Vincula parámetro al segmento `{variable}` de la ruta |
| `@RequestParam` | anotación | — | Añade parámetro de query string a la URL |
| `@RequestHeader` | anotación | — | Añade cabecera HTTP a la petición |
| `@RequestBody` | anotación | — | Serializa el parámetro como cuerpo de la petición (JSON por defecto) |

---

## Buenas y malas prácticas

**Hacer:**
- Añadir `contextId` siempre que dos clientes apunten al mismo `name`, aunque hoy solo haya uno. Si en el futuro se añade un segundo con otra configuración, no se necesita refactorización ni hay riesgo de arranque fallido.
- Usar `path` en `@FeignClient` para el prefijo común de todos los endpoints del servicio en lugar de repetirlo en cada método. Reduce el surface de cambio cuando el API del servicio remoto cambia de versión (`/api/v1/` → `/api/v2/`).
- Declarar el tipo de retorno como `ResponseEntity<T>` cuando se necesita acceder a cabeceras o al código de estado HTTP exacto de la respuesta. Para el caso común (solo cuerpo), `T` directo es más legible.

**Evitar:**
- Omitir el nombre del parámetro en `@PathVariable("id")` cuando el nombre del parámetro Java no coincide exactamente con el placeholder de la ruta. Con compilación sin `-parameters`, Feign no puede inferir el nombre y lanza `IllegalStateException` en arranque.
- Anotar la interfaz `@FeignClient` con `@Component` o incluirla en el `@ComponentScan` de la aplicación. Feign gestiona su propio ciclo de registro; la doble anotación puede crear dos instancias del mismo cliente con comportamientos divergentes.
- Poner lógica de transformación dentro de la interfaz `@FeignClient` mediante métodos `default`. Esa lógica no pasa por el pipeline de Feign (Encoder, Interceptor, Logger) y genera inconsistencia en el comportamiento observable.

---

## Comparación: `@FeignClient` vs RestClient / WebClient

Para un senior que evalúa cuándo usar Feign frente a las alternativas del ecosistema Spring.

| Aspecto | `@FeignClient` | `RestClient` (Spring 6.1+) | `WebClient` |
|---|---|---|---|
| Estilo | Declarativo (interfaz) | Imperativo (builder) | Imperativo / reactivo |
| Integración LoadBalancer | Automática | Manual con `@LoadBalanced` | Manual con `@LoadBalanced` |
| Integración Circuit Breaker | Nativa vía `feign.circuitbreaker.enabled` | Manual | Manual |
| Soporte reactivo | Experimental (Mono/Flux) | No | Nativo |
| Testing | `@MockBean` de la interfaz | Mock del `RestClient.Builder` | `MockWebServer` / `ExchangeFunction` |
| Verbosidad para múltiples endpoints | Baja (un método = un endpoint) | Alta | Alta |
| Recomendado para | Comunicación síncrona entre microservicios | Llamadas HTTP puntuales | Comunicación reactiva / streaming |

> [EXAMEN] En entrevista: "¿Cuándo usarías `RestClient` en lugar de Feign?" La respuesta modelo: cuando la llamada HTTP es puntual y no justifica declarar una interfaz completa, cuando se necesita control granular del request en tiempo de ejecución (body construido dinámicamente), o cuando la dependencia `spring-cloud-starter-openfeign` no está en el proyecto por política de arquitectura.

---

<- [4.1 Setup y bootstrap](sc-feign-setup.md) | [Índice](README.md) | [4.3 Configuración por clase Java](sc-feign-configuracion-java.md) ->
