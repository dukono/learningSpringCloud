# 3.1 Cliente declarativo básico con OpenFeign

← [2.14 Testing de aplicaciones con Eureka](sc-eureka-testing.md) | [Índice](README.md) | [3.2.1 Configuración por clase Java](sc-feign-configuracion-java.md) →

---

## Introducción

Spring Cloud OpenFeign convierte una interfaz Java anotada en un cliente HTTP totalmente funcional, sin necesidad de escribir código de implementación. El problema que resuelve es la verbosidad y el boilerplate que implica usar `RestTemplate` o `WebClient` para cada llamada remota: URLs hardcodeadas, manejo manual de serialización, y lógica de error dispersa. Feign centraliza todo ese contrato en una interfaz con anotaciones de Spring MVC, haciendo que llamar a un servicio remoto sea tan legible como invocar un método local. Se necesita cuando tienes microservicios que se llaman entre sí de forma síncrona y quieres que el código del consumidor sea limpio, testeable y mantenible.

> [PREREQUISITO] Antes de este módulo debes conocer cómo funciona Spring Cloud Eureka (registro de servicios) y tener claro qué es `lb://` como esquema de URL con balanceo de carga.

## Arquitectura del cliente declarativo

Feign opera como un proxy dinámico: en tiempo de arranque Spring registra un bean proxy para cada interfaz anotada con `@FeignClient`. Cuando se invoca un método del proxy, Feign construye un `RequestTemplate`, aplica los interceptores registrados, delega la ejecución HTTP al cliente subyacente (por defecto `java.net.HttpURLConnection`), y finalmente decodifica la respuesta con el `Decoder` configurado.

```
  Llamada Java
       │
       ▼
 ┌─────────────────────────┐
 │   Proxy dinámico Feign  │
 │  (generado en arranque) │
 └────────────┬────────────┘
              │ construye
              ▼
      ┌───────────────────┐
      │  RequestTemplate  │  ← RequestInterceptors aplicados aquí
      └────────┬──────────┘
               │ ejecuta
               ▼
      ┌───────────────────┐
      │  HTTP Client      │  (default / OkHttp / Apache HC5)
      └────────┬──────────┘
               │ respuesta HTTP
               ▼
      ┌───────────────────┐
      │  Decoder / Error  │
      │  Decoder          │
      └───────────────────┘
               │
               ▼
         Objeto Java
```

## Ejemplo central

El siguiente ejemplo muestra el mínimo necesario para declarar y usar un cliente Feign en una aplicación Spring Boot: habilitación con `@EnableFeignClients`, declaración de la interfaz con `@FeignClient` y uso del cliente en un servicio.

```java
// 1. Habilitar Feign en la aplicación
package com.example.orders;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.openfeign.EnableFeignClients;

@SpringBootApplication
@EnableFeignClients(basePackages = "com.example.orders.clients")
public class OrdersApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrdersApplication.class, args);
    }
}
```

```java
// 2. Declarar el cliente Feign (en paquete com.example.orders.clients)
package com.example.orders.clients;

import com.example.orders.dto.InventoryResponse;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestParam;

@FeignClient(
    name = "inventory-service",           // serviceId en Eureka → resuelto como lb://inventory-service
    path = "/api/v1"                      // prefijo de ruta común a todos los métodos
)
public interface InventoryClient {

    @GetMapping("/items/{id}")
    InventoryResponse getItem(@PathVariable("id") Long itemId);

    @GetMapping("/items")
    java.util.List<InventoryResponse> searchItems(
        @RequestParam("category") String category,
        @RequestParam(value = "page", defaultValue = "0") int page
    );
}
```

```java
// 3. DTO de respuesta
package com.example.orders.dto;

public record InventoryResponse(Long id, String name, int stock) {}
```

```java
// 4. Servicio que usa el cliente Feign
package com.example.orders.service;

import com.example.orders.clients.InventoryClient;
import com.example.orders.dto.InventoryResponse;
import org.springframework.stereotype.Service;

@Service
public class OrderService {

    private final InventoryClient inventoryClient;

    public OrderService(InventoryClient inventoryClient) {
        this.inventoryClient = inventoryClient;
    }

    public boolean hasStock(Long itemId, int required) {
        InventoryResponse item = inventoryClient.getItem(itemId);
        return item.stock() >= required;
    }
}
```

```yaml
# 5. application.yml — el serviceId debe coincidir con spring.application.name del servicio remoto
spring:
  application:
    name: orders-service
  cloud:
    openfeign:
      client:
        config:
          inventory-service:
            connectTimeout: 2000
            readTimeout: 5000
```

## Tabla de elementos clave

Los atributos de `@FeignClient` determinan el comportamiento del cliente. A continuación se describen los más importantes:

| Atributo | Tipo | Descripción |
|---|---|---|
| `name` / `value` | String | Nombre lógico del servicio; si Eureka está activo se usa como serviceId para `lb://` |
| `url` | String | URL base fija; si se especifica, Eureka y LoadBalancer se ignoran |
| `path` | String | Prefijo de ruta añadido a todas las llamadas del cliente |
| `configuration` | Class | Clase de configuración específica para este cliente (Encoder, Decoder, etc.) |
| `fallback` | Class | Clase que implementa la interfaz y provee respuestas por defecto ante fallos |
| `fallbackFactory` | Class | Fábrica de fallback con acceso al `Throwable` que causó el fallo |
| `qualifiers` | String[] | Nombres de bean alternativos cuando hay varios clientes con el mismo nombre |
| `dismiss404` | boolean | Si `true`, un 404 no lanza excepción sino que devuelve `null` o `Optional.empty()` |

## Conceptos fundamentales

`@FeignClient` es la anotación núcleo que marca una interfaz como cliente HTTP declarativo. Cada anotación genera un bean proxy en el contexto de Spring, con un sub-contexto (ApplicationContext hijo) propio para la configuración del cliente.

`@EnableFeignClients` activa el escaneo de interfaces `@FeignClient` en los paquetes indicados. Sin esta anotación, el contexto no registra ningún proxy Feign aunque existan las interfaces. El atributo `defaultConfiguration` permite definir una configuración base para todos los clientes.

`lb://serviceId` es el esquema de URL que Spring Cloud LoadBalancer interpreta para resolver el nombre lógico del servicio a una instancia concreta. Feign genera este esquema automáticamente a partir del atributo `name` cuando no se especifica `url`.

`SpringMvcContract` es el `Contract` que Feign usa por defecto en Spring Cloud. Permite usar anotaciones de Spring MVC (`@GetMapping`, `@PostMapping`, `@RequestParam`, `@PathVariable`, `@RequestBody`, `@RequestHeader`) en las interfaces Feign, en lugar de las anotaciones propias de Feign (`@RequestLine`, `@Param`, etc.).

## Buenas y malas prácticas

**Buenas prácticas:**
- Usar `name` con el `spring.application.name` del servicio remoto para aprovechar la resolución dinámica vía Eureka.
- Especificar `path` en `@FeignClient` para evitar repetir el prefijo en cada método.
- Agrupar los clientes Feign en un paquete dedicado (`clients` o `feign`) y usar `basePackages` en `@EnableFeignClients`.
- Anotar **todos** los parámetros del método: un parámetro sin anotación se convierte en el cuerpo de la petición (solo uno puede ser cuerpo).

**Malas prácticas:**
- Usar `url` con una URL hardcodeada en producción: elimina la capacidad de resolución dinámica y el balanceo de carga.
- Poner `@EnableFeignClients` sin `basePackages` en proyectos grandes: escanea todo el classpath innecesariamente.
- Declarar el cliente Feign en el mismo paquete que la clase `@SpringBootApplication` sin configurar `basePackages`: puede funcionar pero es frágil ante refactorizaciones.

> [ADVERTENCIA] Un parámetro de método Feign sin anotación (`@RequestParam`, `@PathVariable`, `@RequestBody`, `@RequestHeader`) es tratado como cuerpo de la petición. Si hay dos parámetros sin anotar se produce un error en tiempo de arranque.

## Verificación y práctica

> [EXAMEN] **1.** ¿Qué anotación es imprescindible en la clase principal de Spring Boot para que los beans `@FeignClient` sean detectados y registrados en el contexto?

> [EXAMEN] **2.** Si `@FeignClient(name = "product-service")` y el servicio se registra en Eureka con `spring.application.name=product-service`, ¿qué URL interna genera Feign para las llamadas? ¿Qué componente resuelve esa URL?

> [EXAMEN] **3.** ¿Cuál es la diferencia entre el atributo `url` y el atributo `name` en `@FeignClient`? ¿Cuándo usarías cada uno?

> [EXAMEN] **4.** Tienes un método Feign `ProductResponse findProduct(Long id)` sin ninguna anotación en el parámetro. ¿Qué ocurrirá cuando Feign intente construir la petición HTTP?

> [EXAMEN] **5.** ¿Por qué `SpringMvcContract` es importante en Spring Cloud OpenFeign y qué ocurre si se cambia por `feign.Contract.Default`?

---

← [2.14 Testing de aplicaciones con Eureka](sc-eureka-testing.md) | [Índice](README.md) | [3.2.1 Configuración por clase Java](sc-feign-configuracion-java.md) →
