# 4.13 Testing — verificación del cliente Feign con @MockBean, WireMock y Stub Runner

<- [4.12 Herencia de interfaz y @SpringQueryMap](sc-feign-herencia.md) | [Índice](README.md) | [5.1 Circuit Breaker: estados, transiciones y modelo de evaluación](sc-circuitbreaker-estados.md) ->

---

## Introducción

Un cliente Feign tiene tres capas verificables: la interfaz en sí (¿declara correctamente el contrato HTTP?), el comportamiento del consumidor (¿gestiona bien las respuestas del cliente Feign?) y la integración real contra un servidor HTTP (¿serializa correctamente la petición? ¿mapea bien la respuesta?). Cada capa requiere una estrategia de test distinta.

El test de unidad del consumidor aísla el cliente Feign con `@MockBean` y verifica que el servicio de negocio reacciona correctamente a los distintos resultados del cliente. El test de integración del cliente en sí arranca un servidor HTTP embebido (WireMock) y verifica que el proxy generado por Feign produce la petición correcta y deserializa la respuesta esperada. Spring Cloud Contract Stub Runner añade una tercera opción: usar los stubs generados a partir de los contratos del proveedor, garantizando que el consumidor no solo compila contra la interfaz correcta sino que también se comporta correctamente ante las respuestas reales del proveedor.

---

## Estrategia 1: test de unidad del consumidor con @MockBean

Esta estrategia verifica que el servicio de negocio (consumidor del cliente Feign) maneja correctamente los distintos resultados del cliente: éxito, excepción, resultado vacío. El cliente Feign se reemplaza por un mock.

```java
package com.example.orderservice.service;

import com.example.orderservice.client.CatalogClient;
import com.example.orderservice.dto.ProductDto;
import com.example.orderservice.exception.ProductNotFoundException;
import feign.FeignException;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.mock.mockito.MockBean;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.BDDMockito.given;
import static org.mockito.Mockito.verify;

/**
 * Test de unidad de OrderService: verifica el comportamiento del servicio
 * ante distintos resultados del CatalogClient (mock).
 *
 * No carga el contexto completo de Spring Cloud Feign ni LoadBalancer.
 * Rápido y aislado.
 */
@SpringBootTest(classes = {OrderService.class})  // contexto mínimo
class OrderServiceUnitTest {

    @MockBean
    CatalogClient catalogClient;           // Feign client reemplazado por mock

    @Autowired
    OrderService orderService;

    @Test
    void createOrder_whenProductAvailable_shouldSucceed() {
        // given
        ProductDto product = new ProductDto("P001", "Widget", 9.99, true);
        given(catalogClient.getProduct("P001")).willReturn(product);

        // when
        OrderDto order = orderService.createOrder("P001", 2);

        // then
        assertThat(order.totalAmount()).isEqualTo(19.98);
        verify(catalogClient).getProduct("P001");
    }

    @Test
    void createOrder_whenProductNotFound_shouldThrowDomainException() {
        // given
        given(catalogClient.getProduct("MISSING"))
                .willThrow(new ProductNotFoundException("MISSING"));

        // then
        assertThatThrownBy(() -> orderService.createOrder("MISSING", 1))
                .isInstanceOf(ProductNotFoundException.class)
                .hasMessageContaining("MISSING");
    }

    @Test
    void createOrder_whenCatalogUnavailable_shouldActivateFallback() {
        // given: simular que el servicio remoto está caído
        given(catalogClient.getProduct("P002"))
                .willThrow(FeignException.ServiceUnavailable.class);

        // then: el servicio debe devolver respuesta degradada, no propagar la excepción
        OrderDto degradedOrder = orderService.createOrder("P002", 1);
        assertThat(degradedOrder.status()).isEqualTo("PENDING_AVAILABILITY_CHECK");
    }
}
```

---

## Estrategia 2: test de integración del cliente Feign con WireMock

Esta estrategia arranca el cliente Feign real (con su Encoder, Decoder, Interceptores) contra un servidor HTTP embebido. Verifica que la petición HTTP generada es correcta y que la respuesta se deserializa como se espera.

**Dependencia para WireMock con Spring Boot:**

```xml
<dependency>
    <groupId>org.wiremock.integrations</groupId>
    <artifactId>wiremock-spring-boot</artifactId>
    <version>3.1.0</version>
    <scope>test</scope>
</dependency>
```

```java
package com.example.orderservice.client;

import com.example.orderservice.dto.ProductDto;
import com.github.tomakehurst.wiremock.client.WireMock;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.wiremock.spring.EnableWireMock;

import static com.github.tomakehurst.wiremock.client.WireMock.*;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

/**
 * Test de integración: carga el cliente Feign real y lo dirige contra WireMock.
 * Verifica que el proxy Feign genera la petición HTTP correcta y
 * deserializa la respuesta en el tipo Java esperado.
 *
 * @EnableWireMock: arranca WireMock en puerto aleatorio e inyecta
 * la propiedad ${wiremock.server.baseUrl} que se usa en el YAML de test.
 */
@SpringBootTest
@EnableWireMock
class CatalogClientIntegrationTest {

    @Autowired
    CatalogClient catalogClient;

    @Test
    void getProduct_whenServerReturns200_shouldDeserializeProductDto() {
        // given: stub WireMock para responder 200 con JSON
        stubFor(get(urlEqualTo("/api/products/P001"))
                .willReturn(aResponse()
                        .withStatus(200)
                        .withHeader("Content-Type", "application/json")
                        .withBody("""
                            {
                              "productId": "P001",
                              "name": "Widget Pro",
                              "price": 29.99,
                              "inStock": true
                            }
                            """)));

        // when
        ProductDto product = catalogClient.getProduct("P001");

        // then
        assertThat(product.productId()).isEqualTo("P001");
        assertThat(product.price()).isEqualTo(29.99);

        // verificar que la petición tenía las cabeceras correctas (si hay interceptores)
        verify(getRequestedFor(urlEqualTo("/api/products/P001"))
                .withHeader("Accept", containing("application/json")));
    }

    @Test
    void getProduct_whenServerReturns404_shouldThrowProductNotFoundException() {
        // given
        stubFor(get(urlEqualTo("/api/products/MISSING"))
                .willReturn(aResponse()
                        .withStatus(404)
                        .withBody("{\"error\": \"Product not found\"}")));

        // then: el ErrorDecoder debe convertir el 404 en ProductNotFoundException
        assertThatThrownBy(() -> catalogClient.getProduct("MISSING"))
                .isInstanceOf(com.example.orderservice.exception.ProductNotFoundException.class);
    }

    @Test
    void getProduct_whenServerReturns503_shouldThrowRetryableException() {
        // given: servidor temporalmente no disponible
        stubFor(get(urlEqualTo("/api/products/P002"))
                .willReturn(aResponse()
                        .withStatus(503)
                        .withHeader("Retry-After", "2")));

        // then
        assertThatThrownBy(() -> catalogClient.getProduct("P002"))
                .isInstanceOf(feign.RetryableException.class);
    }
}
```

**application-test.yml — apuntar el cliente Feign a WireMock:**

```yaml
# Usado durante los tests de integración de cliente Feign
feign:
  client:
    config:
      catalogInherited:
        url: ${wiremock.server.baseUrl}   # inyectado por @EnableWireMock

# Desactivar Eureka en tests
eureka:
  client:
    enabled: false
```

---

## Estrategia 3: Spring Cloud Contract Stub Runner

Esta estrategia usa los stubs generados a partir de los contratos CDC del proveedor. Garantiza que el consumidor se comporta correctamente ante las respuestas reales que el proveedor se comprometió a producir.

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-contract-stub-runner</artifactId>
    <scope>test</scope>
</dependency>
```

```java
package com.example.orderservice.client;

import com.example.orderservice.dto.ProductDto;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.cloud.contract.stubrunner.spring.AutoConfigureStubRunner;
import org.springframework.cloud.contract.stubrunner.spring.StubRunnerProperties;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Test con Stub Runner: descarga los stubs publicados por catalog-service
 * y los usa como servidor WireMock embebido.
 *
 * Los stubs se generan a partir de los contratos de catalog-service
 * (Spring Cloud Contract); el stub JAR se publica en Nexus/Artifactory.
 *
 * ids format: "groupId:artifactId:version:classifier:port"
 *   - version "+" = última disponible
 *   - port 0 = puerto aleatorio
 */
@SpringBootTest
@AutoConfigureStubRunner(
        ids              = "com.example:catalog-service:+:stubs:0",
        stubsMode        = StubRunnerProperties.StubsMode.REMOTE,
        repositoryRoot   = "https://nexus.example.com/repository/maven-public/"
)
class CatalogClientContractTest {

    @Autowired
    CatalogClient catalogClient;

    @Test
    void getProduct_usingContractStub_shouldReturnProductMatchingContract() {
        // El stub responderá con los datos definidos en el contrato de catalog-service
        // La respuesta es la misma que el proveedor garantiza en sus tests de contrato
        ProductDto product = catalogClient.getProduct("contract-product-id");

        assertThat(product).isNotNull();
        assertThat(product.productId()).isEqualTo("contract-product-id");
    }
}
```

---

## Tabla resumen: estrategia de test recomendada por situación

| Situación | Estrategia recomendada |
|---|---|
| Verificar lógica de negocio del consumidor ante errores del cliente Feign | `@MockBean` del cliente Feign (unidad) |
| Verificar que el Encoder produce la petición HTTP correcta | WireMock con `verify(requestedFor(...))` |
| Verificar que el Decoder deserializa la respuesta correctamente | WireMock con `stubFor` + assert en el resultado |
| Verificar que el ErrorDecoder convierte correctamente los errores HTTP | WireMock con respuestas 4xx/5xx stubbed |
| Verificar que los interceptores añaden las cabeceras esperadas | WireMock con `withHeader(...)` en el `verify` |
| Verificar compatibilidad con la API real del proveedor | Spring Cloud Contract Stub Runner |
| Verificar rendimiento y comportamiento bajo carga | Test de carga (ICaRUS) con instancia real |

---

## Preguntas de entrevista — nivel senior

**P1:** ¿Por qué usar `@MockBean` para el cliente Feign en lugar de `@Mock` de Mockito?

Respuesta: `@MockBean` registra el mock en el `ApplicationContext` de Spring Test, reemplazando el bean Feign real. `@Mock` crea un mock de Mockito fuera del contexto Spring; para que el servicio bajo test lo use, hay que inyectarlo manualmente o usar `@InjectMocks`. Con `@SpringBootTest`, `@MockBean` es la forma correcta porque garantiza que el mock es el mismo bean que Spring inyecta en el servicio.

**P2:** ¿Qué diferencia hay entre un test con WireMock y un test con Spring Cloud Contract Stub Runner?

Respuesta: WireMock con stubs manuales verifica que el cliente Feign genera la petición correcta y procesa la respuesta, pero los stubs son locales y pueden quedar desincronizados de lo que el proveedor realmente produce. Stub Runner usa stubs generados directamente de los contratos del proveedor: si el proveedor cambia su API y actualiza sus contratos, los stubs del consumidor fallan en el siguiente build, detectando la incompatibilidad antes de llegar a producción.

**P3:** ¿Qué pasa si el bean `CatalogClient` (Feign) falla en arranque porque Eureka no está disponible durante los tests?

Respuesta: hay que desactivar Eureka en el perfil de test con `eureka.client.enabled=false` en `application-test.yml`. Además, apuntar el cliente Feign a una URL fija (WireMock o localhost) mediante `feign.client.config.[clientName].url=http://localhost:PORT` para que el cliente no intente resolver el nombre por service discovery.

**P4:** ¿Cómo verificar que un `RequestInterceptor` añade correctamente la cabecera `Authorization` en todos los tests de integración del cliente?

Respuesta: en el test WireMock, tras la llamada al método del cliente, usar `verify(getRequestedFor(anyUrl()).withHeader("Authorization", matching("Bearer .*")))`. Para un test de unidad, mockear el `TokenProvider` que el interceptor usa y verificar que el bean mock fue invocado.

**P5:** ¿Cómo testear el comportamiento del `ErrorDecoder` ante un 503 con cabecera `Retry-After`?

Respuesta: en WireMock, stubbear la ruta con `aResponse().withStatus(503).withHeader("Retry-After", "2")`. Luego verificar que el cliente lanza `RetryableException` y que el campo `retryAfter` de la excepción apunta a una fecha aproximadamente 2 segundos en el futuro. Alternativamente, testear `CatalogErrorDecoder.decode()` en unidad pura sin Feign, pasando un `Response` construido manualmente con `Response.builder()`.

---

<- [4.12 Herencia de interfaz y @SpringQueryMap](sc-feign-herencia.md) | [Índice](README.md) | [5.1 Circuit Breaker: estados, transiciones y modelo de evaluación](sc-circuitbreaker-estados.md) ->
