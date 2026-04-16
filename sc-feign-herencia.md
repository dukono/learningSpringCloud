# 4.12 Herencia de interfaz y @SpringQueryMap — contratos compartidos y query params tipados

<- [4.11 Cliente HTTP subyacente](sc-feign-cliente-http.md) | [Índice](README.md) | [4.13 Testing](sc-feign-testing.md) ->

---

## Introducción

El patrón de herencia en Feign permite que la interfaz de un controlador Spring MVC sirva simultáneamente como contrato del servidor y como tipo del cliente Feign. En lugar de duplicar las anotaciones de mapeo (`@GetMapping`, `@RequestParam`, etc.) en la interfaz del cliente, el cliente extiende directamente la interfaz que también implementa el controlador del proveedor. Un único punto de definición garantiza que cliente y servidor siempre están sincronizados en paths, verbos y tipos.

`@SpringQueryMap` es una anotación complementaria que resuelve un problema frecuente: cómo pasar un objeto POJO como conjunto de parámetros de query string sin tener que declarar cada campo como `@RequestParam` individual. Con `@SpringQueryMap`, Feign convierte cada campo del POJO en un parámetro de query, manteniendo la tipificación del objeto y evitando la expansión manual de decenas de parámetros de filtro.

Ambas herramientas tienen sus costes: la herencia introduce acoplamiento entre el servicio proveedor y el consumidor, y `@SpringQueryMap` tiene limitaciones con tipos anidados que el desarrollador debe conocer.

---

## Diagrama: estructura del patrón de herencia

El siguiente diagrama muestra cómo la interfaz compartida une al controlador del proveedor con el cliente del consumidor.

```
 Módulo compartido (artifact: catalog-api)
 ─────────────────────────────────────────
 interface CatalogApi {
     @GetMapping("/api/products/{id}")
     ProductDto getProduct(@PathVariable String id);

     @GetMapping("/api/products")
     List<ProductDto> search(@SpringQueryMap ProductFilter filter);
 }

     ┌─────────────────────────────────┐
     │                                 │
     ▼                                 ▼
 catalog-service                  order-service
 ───────────────                  ─────────────
 @RestController                  @FeignClient(
 class CatalogController           name = "catalog-service"
     implements CatalogApi {    )
     // implementación real     interface CatalogClient
 }                                   extends CatalogApi {}
                                  // heredan las anotaciones de CatalogApi
```

---

## Ejemplo central

El siguiente ejemplo muestra el patrón completo: la interfaz compartida en el módulo API, el controlador en el servicio proveedor y el cliente Feign en el consumidor. También muestra el uso de `@SpringQueryMap` con un objeto de filtro.

**catalog-api/src/main/java/com/example/catalog/api/CatalogApi.java (módulo compartido):**

```java
package com.example.catalog.api;

import com.example.catalog.dto.ProductDto;
import com.example.catalog.dto.ProductFilter;
import org.springframework.cloud.openfeign.SpringQueryMap;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;

import java.util.List;

/**
 * Contrato compartido entre el proveedor (CatalogController) y el consumidor (CatalogClient).
 * Publicado como artifact Maven independiente para que ambos servicios lo dependan.
 *
 * IMPORTANTE: las anotaciones @RequestMapping en una interfaz compartida con Spring MVC
 * y Feign tienen sutilezas — ver sección de comparación.
 */
@RequestMapping("/api/products")
public interface CatalogApi {

    @GetMapping("/{productId}")
    ProductDto getProduct(@PathVariable("productId") String productId);

    /**
     * @SpringQueryMap convierte ProductFilter en parámetros de query string.
     * Sin esta anotación, Feign intentaría serializar el objeto como body JSON.
     *
     * ProductFilter { category, minPrice, maxPrice, inStock, page, size }
     * → GET /api/products?category=electronics&minPrice=10&page=0&size=20
     */
    @GetMapping
    List<ProductDto> search(@SpringQueryMap ProductFilter filter);
}
```

**catalog-api/src/main/java/com/example/catalog/dto/ProductFilter.java:**

```java
package com.example.catalog.dto;

/**
 * POJO de filtro para búsqueda de productos.
 * Feign serializa cada campo no-null como parámetro de query string.
 * Los campos null se omiten de la URL.
 *
 * Usar record o clase con getters: @SpringQueryMap usa los nombres de los campos
 * (o getters si la clase no es un record) para construir los parámetros.
 */
public record ProductFilter(
        String category,
        Double minPrice,
        Double maxPrice,
        Boolean inStock,
        int page,
        int size
) {}
```

**catalog-service/CatalogController.java (proveedor — implementa CatalogApi):**

```java
package com.example.catalog.controller;

import com.example.catalog.api.CatalogApi;
import com.example.catalog.dto.ProductDto;
import com.example.catalog.dto.ProductFilter;
import com.example.catalog.service.CatalogService;
import org.springframework.web.bind.annotation.RestController;

import java.util.List;

@RestController
public class CatalogController implements CatalogApi {

    private final CatalogService catalogService;

    public CatalogController(CatalogService catalogService) {
        this.catalogService = catalogService;
    }

    @Override
    public ProductDto getProduct(String productId) {
        return catalogService.findById(productId);
    }

    @Override
    public List<ProductDto> search(ProductFilter filter) {
        return catalogService.search(filter);
    }
}
```

**order-service/CatalogClient.java (consumidor — extiende CatalogApi via Feign):**

```java
package com.example.orderservice.client;

import com.example.catalog.api.CatalogApi;
import org.springframework.cloud.openfeign.FeignClient;

/**
 * El cliente hereda TODAS las anotaciones de CatalogApi.
 * No se necesita redeclarar @GetMapping, @PathVariable, @SpringQueryMap, etc.
 * Si CatalogApi cambia, el cliente se actualiza automáticamente al actualizar la dependencia.
 */
@FeignClient(
        name      = "catalog-service",
        contextId = "catalogInherited"
)
public interface CatalogClient extends CatalogApi {
    // Sin cuerpo: todo viene heredado de CatalogApi
}
```

> [CONCEPTO] `@SpringQueryMap` acepta cualquier POJO con campos accesibles (record, clase con campos públicos, o clase con getters siguiendo la convención JavaBeans). Los campos con valor `null` se omiten del query string. Para incluir explícitamente un campo nulo, no hay soporte nativo: hay que recurrir a `@RequestParam` individual o a un `Encoder` personalizado.

---

## Tabla de elementos clave

| Elemento | Descripción |
|---|---|
| Herencia de interfaz | Patrón donde la interfaz del proveedor define el contrato y el cliente Feign la extiende |
| `@SpringQueryMap` | Anotación que convierte un POJO en parámetros de query string en una llamada GET |
| Módulo API compartido | Artifact Maven/Gradle que contiene la interfaz, DTOs y la anotación; dependencia de ambos servicios |
| Campos `null` en `@SpringQueryMap` | Se omiten del query string; no hay forma nativa de forzar `null` a string `"null"` |
| Tipos anidados en `@SpringQueryMap` | No soportados de forma nativa; Feign aplana el objeto un nivel; objetos anidados se serializan como `toString()` |
| `@RequestMapping` en la interfaz | Se hereda en el cliente; el proveedor también la hereda; puede causar conflicto si el controlador ya tiene `@RequestMapping` propia |

---

## Buenas y malas prácticas

**Hacer:**
- Publicar la interfaz compartida en un módulo Maven/Gradle independiente (`catalog-api`) con sus DTOs. Esta separación permite que el proveedor evolucione su implementación sin modificar el contrato, y que el consumidor actualice la versión del contrato de forma controlada.
- Usar `@SpringQueryMap` con records de Java en lugar de clases mutables. Los records son inmutables, tienen nombres de campo claros y Feign los serializa correctamente. Evitan el problema de clases con getters que no siguen la convención JavaBeans.
- Verificar el query string generado activando `logger-level: FULL` en el cliente en tests. `@SpringQueryMap` puede producir URLs sorprendentes con campos de tipo `Collection` o arrays (se repite el nombre del parámetro: `?ids=1&ids=2&ids=3`).

**Evitar:**
- Compartir la interfaz cuando proveedor y consumidor evolucionan a ritmos muy diferentes. Si el proveedor añade nuevos endpoints frecuentemente, el módulo compartido se convierte en un punto de acoplamiento que fuerza actualizaciones coordinadas. En ese caso, duplicar la interfaz del cliente y usar Spring Cloud Contract para validar la compatibilidad es una arquitectura más desacoplada.
- Poner lógica de negocio o dependencias de infraestructura en el módulo de la interfaz compartida. El módulo API debe contener solo la interfaz y los DTOs; cualquier dependencia adicional (p. ej. Jackson, Validation) debe ser `provided` o `optional` para no contaminar el classpath del consumidor.
- Usar `@SpringQueryMap` con objetos que tienen campos de tipo `LocalDate`, `ZonedDateTime` u otros tipos de fecha sin verificar la serialización. Feign usa `toString()` por defecto para estos tipos, que puede producir formatos no ISO-8601 dependiendo del locale del sistema. Registrar un `QueryMapEncoder` personalizado o usar `String` en el DTO de filtro para fechas.

---

## Comparación: herencia vs contrato duplicado

| Aspecto | Herencia de interfaz | Contrato duplicado en el consumidor |
|---|---|---|
| Sincronización cliente-servidor | Automática (actualización de dependencia) | Manual (puede quedar desincronizado) |
| Acoplamiento | Alto (cambios en API afectan al consumidor) | Bajo (consumidor es independiente) |
| Adecuado para | Equipos que controlan proveedor y consumidor | Equipos independientes con ritmos de release distintos |
| Detección de incompatibilidades | En tiempo de compilación (cambio de dependencia) | En tiempo de ejecución o con Spring Cloud Contract |
| Ventaja operativa | Una sola fuente de verdad | Menor acoplamiento de release |

> [EXAMEN] En entrevista: "¿Cuál es el principal riesgo de compartir la interfaz entre el controlador del proveedor y el cliente Feign?" Respuesta modelo: el acoplamiento de build. Si el proveedor cambia la firma de un método (añade un parámetro, cambia el tipo de retorno), el consumidor no compila hasta que actualiza la versión del módulo compartido, forzando un release coordinado entre ambos servicios. Este acoplamiento viola el principio de despliegue independiente de microservicios.

---

<- [4.11 Cliente HTTP subyacente](sc-feign-cliente-http.md) | [Índice](README.md) | [4.13 Testing](sc-feign-testing.md) ->
