# 3.10 Testing / Verificación de OpenFeign

← [3.9.2 Compresión de peticiones y respuestas](sc-feign-compresion.md) | [Índice](README.md) | [4.1 Circuit Breaker — Estados y Transiciones](sc-circuitbreaker-estados.md) →

---

## Introducción

Testear clientes Feign requiere estrategias distintas según el objetivo del test. Hay tres niveles de testing claramente diferenciados: tests unitarios del servicio que usa el cliente Feign (donde el cliente se reemplaza con un mock), tests de integración del cliente Feign contra un servidor HTTP stub (donde se verifica que la serialización y el mapeo HTTP son correctos), y tests end-to-end con Spring Cloud Contract (donde se verifica el contrato completo productor-consumidor). Cada estrategia usa herramientas diferentes — `@MockBean`, WireMock con `@AutoConfigureWireMock`, o Spring Cloud Contract Stub Runner — y tiene su propio alcance de verificación.

## Pirámide de testing para Feign

La elección de la estrategia depende de qué se quiere verificar. Los tres niveles son complementarios, no excluyentes.

```
                    ┌───────────────────────┐
                    │  Contract Testing     │  ← Verifica contrato E2E con el productor
                    │  (Stub Runner)        │    (más lento, más cobertura)
                    └───────────┬───────────┘
                    ┌───────────┴───────────┐
                    │  Integration Test     │  ← Verifica serialización HTTP real
                    │  (WireMock)           │    (verifica que el cliente envía lo correcto)
                    └───────────┬───────────┘
                    ┌───────────┴───────────┐
                    │  Unit Test            │  ← Verifica lógica del servicio
                    │  (@MockBean Feign)    │    (rápido, sin red)
                    └───────────────────────┘
```

## Ejemplo central

El siguiente ejemplo muestra las tres estrategias de testing de forma completa y ejecutable. Incluye un servicio real, su cliente Feign, y los tres tipos de test.

```java
// El cliente Feign a testear
package com.example.orders.clients;

import com.example.orders.dto.InventoryResponse;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestParam;

@FeignClient(name = "inventory-service", path = "/api/v1")
public interface InventoryClient {

    @GetMapping("/items/{id}")
    InventoryResponse getItem(@PathVariable("id") Long id);

    @GetMapping("/items/availability")
    boolean checkAvailability(
        @RequestParam("sku") String sku,
        @RequestParam("qty") int quantity
    );
}
```

```java
// El servicio que usa el cliente Feign (la unidad a testear)
package com.example.orders.service;

import com.example.orders.clients.InventoryClient;
import com.example.orders.dto.InventoryResponse;
import com.example.orders.exception.InsufficientStockException;
import org.springframework.stereotype.Service;

@Service
public class OrderService {

    private final InventoryClient inventoryClient;

    public OrderService(InventoryClient inventoryClient) {
        this.inventoryClient = inventoryClient;
    }

    public InventoryResponse getItemOrThrow(Long id) {
        InventoryResponse item = inventoryClient.getItem(id);
        if (item == null) {
            throw new IllegalArgumentException("Ítem no encontrado: " + id);
        }
        return item;
    }

    public void validateStock(String sku, int requiredQty) {
        boolean available = inventoryClient.checkAvailability(sku, requiredQty);
        if (!available) {
            throw new InsufficientStockException(sku, requiredQty);
        }
    }
}
```

```java
// DTOs
package com.example.orders.dto;

public record InventoryResponse(Long id, String name, int stock) {}
```

```java
package com.example.orders.exception;

public class InsufficientStockException extends RuntimeException {
    public InsufficientStockException(String sku, int qty) {
        super("Stock insuficiente para SKU " + sku + ": se requieren " + qty);
    }
}
```

```java
// ESTRATEGIA 1: Test unitario del servicio con @MockBean
// Objetivo: verificar la lógica de negocio en OrderService
// No se ejecuta ninguna petición HTTP real
package com.example.orders.service;

import com.example.orders.clients.InventoryClient;
import com.example.orders.dto.InventoryResponse;
import com.example.orders.exception.InsufficientStockException;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.mock.mockito.MockBean;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.Mockito.when;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE)
class OrderServiceUnitTest {

    @Autowired
    private OrderService orderService;

    // @MockBean reemplaza el bean real de InventoryClient con un mock de Mockito
    // El proxy Feign no se crea en este contexto — se usa directamente el mock
    @MockBean
    private InventoryClient inventoryClient;

    @Test
    void getItemOrThrow_returnsItem_whenClientReturnsValidResponse() {
        InventoryResponse expected = new InventoryResponse(1L, "Widget Pro", 50);
        when(inventoryClient.getItem(1L)).thenReturn(expected);

        InventoryResponse result = orderService.getItemOrThrow(1L);

        assertThat(result).isEqualTo(expected);
        assertThat(result.stock()).isEqualTo(50);
    }

    @Test
    void getItemOrThrow_throwsException_whenClientReturnsNull() {
        when(inventoryClient.getItem(99L)).thenReturn(null);

        assertThatThrownBy(() -> orderService.getItemOrThrow(99L))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("99");
    }

    @Test
    void validateStock_throwsException_whenStockInsufficient() {
        when(inventoryClient.checkAvailability("SKU-001", 10)).thenReturn(false);

        assertThatThrownBy(() -> orderService.validateStock("SKU-001", 10))
            .isInstanceOf(InsufficientStockException.class)
            .hasMessageContaining("SKU-001");
    }
}
```

```java
// ESTRATEGIA 2: Test de integración del cliente Feign con WireMock
// Objetivo: verificar que el cliente Feign serializa/deserializa correctamente
// y que los headers, paths y query params se envían como se espera
package com.example.orders.clients;

import com.example.orders.dto.InventoryResponse;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.cloud.contract.wiremock.AutoConfigureWireMock;

import static com.github.tomakehurst.wiremock.client.WireMock.*;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

// @AutoConfigureWireMock(port = 0) inicia un servidor WireMock en puerto aleatorio
// y configura la propiedad wiremock.server.port automáticamente
@SpringBootTest(
    webEnvironment = SpringBootTest.WebEnvironment.NONE,
    properties = {
        // Apuntar el cliente Feign al servidor WireMock en lugar de Eureka
        "spring.cloud.openfeign.client.config.inventory-service.url=http://localhost:${wiremock.server.port}",
        // Deshabilitar Eureka para este test
        "eureka.client.enabled=false",
        "spring.cloud.discovery.enabled=false"
    }
)
@AutoConfigureWireMock(port = 0)
class InventoryClientIntegrationTest {

    @Autowired
    private InventoryClient inventoryClient;

    @Test
    void getItem_deserializesResponseCorrectly() {
        stubFor(get(urlEqualTo("/api/v1/items/42"))
            .willReturn(aResponse()
                .withStatus(200)
                .withHeader("Content-Type", "application/json")
                .withBody("""
                    {"id": 42, "name": "Widget Pro", "stock": 100}
                    """)));

        InventoryResponse result = inventoryClient.getItem(42L);

        assertThat(result.id()).isEqualTo(42L);
        assertThat(result.name()).isEqualTo("Widget Pro");
        assertThat(result.stock()).isEqualTo(100);
    }

    @Test
    void checkAvailability_sendsCorrectQueryParams() {
        stubFor(get(urlPathEqualTo("/api/v1/items/availability"))
            .withQueryParam("sku", equalTo("SKU-001"))
            .withQueryParam("qty", equalTo("5"))
            .willReturn(aResponse()
                .withStatus(200)
                .withHeader("Content-Type", "application/json")
                .withBody("true")));

        boolean result = inventoryClient.checkAvailability("SKU-001", 5);

        assertThat(result).isTrue();
    }

    @Test
    void getItem_propagatesErrorDecoder_on404() {
        stubFor(get(urlEqualTo("/api/v1/items/999"))
            .willReturn(aResponse()
                .withStatus(404)
                .withHeader("Content-Type", "application/json")
                .withBody("""
                    {"error": "Item not found", "itemId": 999}
                    """)));

        // Con ErrorDecoder personalizado, se espera excepción de dominio
        // Sin ErrorDecoder personalizado, se espera FeignException.NotFound
        assertThatThrownBy(() -> inventoryClient.getItem(999L))
            .isInstanceOf(Exception.class);
    }
}
```

```java
// ESTRATEGIA 3: Test con Spring Cloud Contract Stub Runner (contract testing)
// Objetivo: verificar el contrato entre el consumidor y el productor
// usando stubs generados por el productor
package com.example.orders.clients;

import com.example.orders.dto.InventoryResponse;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.cloud.contract.stubrunner.spring.AutoConfigureStubRunner;
import org.springframework.cloud.contract.stubrunner.spring.StubRunnerProperties;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(
    webEnvironment = SpringBootTest.WebEnvironment.NONE,
    properties = {
        "spring.cloud.openfeign.client.config.inventory-service.url=http://localhost:${stubrunner.runningstubs.com.example.inventory-service.port}"
    }
)
@AutoConfigureStubRunner(
    ids = "com.example:inventory-service:+:stubs:8081",
    stubsMode = StubRunnerProperties.StubsMode.LOCAL   // buscar stubs en ~/.m2 local
    // Para stubs remotos: StubRunnerProperties.StubsMode.REMOTE con repositoryRoot
)
class InventoryClientContractTest {

    @Autowired
    private InventoryClient inventoryClient;

    @Test
    void getItem_matchesProducerContract() {
        // El stub fue generado por el productor y garantiza que
        // /api/v1/items/1 devuelve el JSON acordado en el contrato
        InventoryResponse result = inventoryClient.getItem(1L);

        assertThat(result).isNotNull();
        assertThat(result.id()).isEqualTo(1L);
        // Los campos adicionales dependen del contrato definido en el productor
    }
}
```

## Tabla comparativa de estrategias

| Estrategia | Herramienta | Lo que verifica | Velocidad | Cuándo usar |
|---|---|---|---|---|
| Test unitario | `@MockBean` | Lógica del servicio que consume Feign | Muy rápida | Siempre para tests de servicio |
| Test integración | WireMock + `@AutoConfigureWireMock` | Serialización HTTP, headers, paths | Media | Al desarrollar o modificar el cliente |
| Contract test | Spring Cloud Contract Stub Runner | Contrato productor-consumidor | Lenta | Antes de cada release / CI |

## Buenas y malas prácticas

**Buenas prácticas:**
- Usar `@MockBean` para tests unitarios del servicio: es la forma correcta de aislar la lógica de negocio de la infraestructura HTTP.
- Usar WireMock para tests de integración del cliente: verifica que el JSON generado por `SpringEncoder` y parseado por `SpringDecoder` es correcto.
- Deshabilitar Eureka en los tests de integración: `eureka.client.enabled=false` evita intentos de registro durante el test.
- Usar `url` en las propiedades del test para apuntar al WireMock: es más limpio que sobreescribir la URL del servicio vía discovery.

**Malas prácticas:**
- Testear `@MockBean(InventoryClient.class)` en un test donde quieres verificar la serialización HTTP: el mock no envía ninguna petición real.
- Omitir `@AutoConfigureWireMock` y escribir el servidor stub manualmente: más frágil y verboso.

> [EXAMEN] La diferencia clave entre `@MockBean` y WireMock para testing de Feign está en el examen: `@MockBean` no verifica que el cliente Feign genera la petición HTTP correcta (no hay red), WireMock sí verifica el HTTP real.

## Verificación y práctica

> [EXAMEN] **1.** ¿Por qué se usa `@MockBean` en lugar de `@Autowired` de la implementación real al testear un servicio que usa un `FeignClient`?

> [EXAMEN] **2.** ¿Qué ventaja tiene WireMock frente a `@MockBean` cuando se quiere verificar que el cliente Feign serializa correctamente la petición HTTP?

> [EXAMEN] **3.** En un test con `@AutoConfigureWireMock(port = 0)`, ¿cómo se configura el cliente Feign para que apunte al servidor WireMock en lugar de Eureka?

> [EXAMEN] **4.** ¿Cuándo usarías Spring Cloud Contract Stub Runner en lugar de WireMock para testear un cliente Feign?

> [EXAMEN] **5.** Si en un test de integración con WireMock el `ErrorDecoder` personalizado lanza `InventoryNotFoundException` para HTTP 404, ¿cómo verificarías en el test que se lanza la excepción correcta?

---

← [3.9.2 Compresión de peticiones y respuestas](sc-feign-compresion.md) | [Índice](README.md) | [4.1 Circuit Breaker — Estados y Transiciones](sc-circuitbreaker-estados.md) →
