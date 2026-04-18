# 10.6 Spring Cloud Contract — Clases Base Productor

← [10.5 Plugin Maven y Gradle](sc-contract-plugin-config.md) | [Índice](README.md) | [10.7 Stub Runner](sc-contract-stub-runner.md) →

---

## Introducción

Las clases base del productor son el punto de extensión donde el desarrollador configura el contexto de test sobre el que los tests generados automáticamente por Spring Cloud Contract verificarán los contratos. Los tests generados extienden la clase base y heredan su configuración, por lo que la clase base debe dejar el sistema preparado para recibir llamadas exactamente como define el contrato. Es un patrón obligatorio: sin una clase base correctamente configurada, los tests generados no pueden ejecutarse.

> [PREREQUISITO] Este nodo requiere haber comprendido el funcionamiento del plugin de [10.5 Plugin Maven y Gradle](sc-contract-plugin-config.md).

## Por qué los tests generados necesitan una clase base

El plugin de Spring Cloud Contract genera tests Java que contienen el código de verificación del contrato (aserciones sobre el response), pero no incluyen configuración del entorno de test. La clase base es la responsable de preparar ese entorno: iniciar el contexto Spring, configurar MockMvc/WebTestClient, inyectar dependencias y establecer datos de prueba.

```
Contrato (.groovy) 
         │
         ▼
Plugin Maven/Gradle
         │
         ▼
Test generado (GENERATED — no editar):
─────────────────────────────────────────
public class ContractVerifierTest extends BaseOrderTest {
    @Test
    public void validate_shouldReturnOrder() throws Exception {
        // código generado por SCC — llama al endpoint y verifica el response
        Response response = given()
            .get("/orders/1");
        assertThat(response.statusCode()).isEqualTo(200);
        assertThat(response.jsonPath().getString("status")).isEqualTo("CONFIRMED");
    }
}
─────────────────────────────────────────
         │  extiende
         ▼
Clase base (CÓDIGO DEL DESARROLLADOR):
─────────────────────────────────────────
public abstract class BaseOrderTest {
    @BeforeEach
    void setup() {
        RestAssuredMockMvc.standaloneSetup(new OrderController(...));
    }
}
```

> [CONCEPTO] Los tests generados son código **no editable** que se regenera en cada build. Toda la lógica de configuración del test debe estar en la clase base. El desarrollador **nunca debe modificar** los tests generados directamente.

## Clase base con RestAssuredMockMvc (MockMvc)

El patrón más común en aplicaciones Spring MVC (no reactivas) es configurar `RestAssuredMockMvc.standaloneSetup()` en el `@BeforeEach` de la clase base. Este enfoque no levanta el contexto completo de Spring, lo que hace los tests más rápidos.

```java
// src/test/java/com/example/order/BaseOrderTest.java
package com.example.order;

import com.example.order.controller.OrderController;
import com.example.order.service.OrderService;
import io.restassured.module.mockmvc.RestAssuredMockMvc;
import org.junit.jupiter.api.BeforeEach;
import org.mockito.Mock;
import org.mockito.MockitoAnnotations;

// La clase NO tiene anotaciones de Spring — usa MockMvc standalone
// Esto es más rápido que @SpringBootTest pero requiere configurar mocks manualmente
public abstract class BaseOrderTest {

    @Mock
    private OrderService orderService;

    @BeforeEach
    public void setup() {
        // Inicializa los mocks de Mockito
        MockitoAnnotations.openMocks(this);

        // Configura comportamiento de los mocks
        org.mockito.Mockito.when(orderService.getOrder(1L))
            .thenReturn(new Order(1L, "CONFIRMED", 150.00));

        // Configura RestAssuredMockMvc con el controlador bajo test
        // standaloneSetup: no levanta contexto Spring, solo el controlador
        RestAssuredMockMvc.standaloneSetup(new OrderController(orderService));
    }
}
```

> [CONCEPTO] `RestAssuredMockMvc.standaloneSetup(controller)` configura un servidor MockMvc con solo el controlador especificado, sin levantar el contexto completo de Spring Boot. Es la opción más rápida para tests de contratos HTTP simples donde no se necesita autowiring complejo.

## Clase base con @SpringBootTest y contexto completo

Cuando la lógica del controlador requiere el contexto completo de Spring (base de datos, beans complejos, seguridad), la clase base puede usar `@SpringBootTest`.

```java
// src/test/java/com/example/payment/BasePaymentTest.java
package com.example.payment;

import com.example.payment.controller.PaymentController;
import io.restassured.module.mockmvc.RestAssuredMockMvc;
import org.junit.jupiter.api.BeforeEach;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import com.example.payment.service.PaymentGateway;

// @SpringBootTest levanta el contexto completo — más lento pero más realista
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.MOCK)
@AutoConfigureMockMvc
public abstract class BasePaymentTest {

    @Autowired
    private PaymentController paymentController;

    // @MockBean reemplaza el bean real en el contexto de Spring
    @MockBean
    private PaymentGateway paymentGateway;

    @BeforeEach
    public void setup() {
        // Configura el comportamiento del mock del gateway externo
        org.mockito.Mockito.when(paymentGateway.charge(org.mockito.ArgumentMatchers.any()))
            .thenReturn(new PaymentResult("APPROVED", "TXN-001"));

        // Configura RestAssuredMockMvc con el controlador inyectado por Spring
        RestAssuredMockMvc.standaloneSetup(paymentController);
    }
}
```

## Clase base con WebTestClient para aplicaciones reactivas

Para productores basados en Spring WebFlux, la clase base configura `WebTestClient` en lugar de MockMvc. El plugin debe tener `testMode = WEBTESTCLIENT` para generar tests que usen este cliente.

```java
// src/test/java/com/example/reactive/BaseReactiveTest.java
package com.example.reactive;

import com.example.reactive.handler.ProductHandler;
import com.example.reactive.router.ProductRouter;
import io.restassured.module.webtestclient.RestAssuredWebTestClient;
import org.junit.jupiter.api.BeforeEach;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public abstract class BaseReactiveTest {

    @Autowired
    private ProductRouter productRouter;

    @Autowired
    private ProductHandler productHandler;

    @BeforeEach
    public void setup() {
        // Para aplicaciones reactivas, se usa RestAssuredWebTestClient
        RestAssuredWebTestClient.standaloneSetup(productRouter, productHandler);
    }
}
```

## Múltiples clases base con baseClassMappings

Cuando el proyecto tiene contratos de múltiples dominios, `baseClassMappings` en el plugin asigna cada grupo de contratos a su propia clase base. La configuración del plugin determina qué clase base extiende cada test generado.

```xml
<!-- pom.xml — baseClassMappings para múltiples clases base -->
<configuration>
    <baseClassMappings>
        <!-- Contratos en src/test/resources/contracts/order/ → extienden BaseOrderTest -->
        <baseClassMapping>
            <contractPackageRegex>.*order.*</contractPackageRegex>
            <baseClassFQN>com.example.order.BaseOrderTest</baseClassFQN>
        </baseClassMapping>
        <!-- Contratos en src/test/resources/contracts/payment/ → extienden BasePaymentTest -->
        <baseClassMapping>
            <contractPackageRegex>.*payment.*</contractPackageRegex>
            <baseClassFQN>com.example.payment.BasePaymentTest</baseClassFQN>
        </baseClassMapping>
    </baseClassMappings>
</configuration>
```

```java
// Árbol de clases base resultante
//
// BaseOrderTest (abstract) ← generado/OrderTest.java
// BasePaymentTest (abstract) ← generado/PaymentTest.java
//
// Cada clase base configura solo el contexto necesario para su dominio
public abstract class BaseOrderTest {
    @BeforeEach
    public void setup() {
        RestAssuredMockMvc.standaloneSetup(new OrderController(/* mocks order */));
    }
}

public abstract class BasePaymentTest {
    @BeforeEach
    public void setup() {
        RestAssuredMockMvc.standaloneSetup(new PaymentController(/* mocks payment */));
    }
}
```

## Tabla de estrategias de clase base

| Estrategia | Anotación | Ventaja | Cuándo usar |
|---|---|---|---|
| Standalone MockMvc | ninguna (manual) | Rápido, sin Spring context | Controladores simples con pocas dependencias |
| `@SpringBootTest` + `@MockBean` | `@SpringBootTest` | Contexto real, mocking preciso | Controladores con dependencias complejas |
| WebTestClient standalone | ninguna | Rápido, reactivo | Routers/Handlers WebFlux simples |
| `@SpringBootTest` WebFlux | `@SpringBootTest` | Contexto reactivo completo | WebFlux con contexto completo |

## Buenas y malas prácticas

**Buenas prácticas**:
- Usar `standaloneSetup` siempre que sea posible — los tests de contratos son más rápidos sin contexto Spring completo.
- Declarar las clases base como `abstract` — los tests generados las extienden y no deben instanciarse directamente.
- Configurar los mocks con valores que coincidan exactamente con el body definido en los contratos.
- Usar `@MockBean` cuando se necesita Spring context completo para reemplazar beans de infraestructura externa (bases de datos, gateways externos).

**Malas prácticas**:
- Poner lógica de negocio en la clase base — solo debe contener configuración de test.
- Olvidar el `@BeforeEach` con `RestAssuredMockMvc.standaloneSetup()` — los tests generados lanzarán `NullPointerException`.
- Usar `@SpringBootTest(webEnvironment = RANDOM_PORT)` cuando solo se necesita MockMvc — levanta un servidor HTTP real innecesariamente.
- Editar los tests generados directamente — se pierden en el siguiente build.

## Verificación y práctica

> [EXAMEN] 1. ¿Por qué los tests generados por Spring Cloud Contract necesitan una clase base y qué debe contener esa clase base como mínimo?

> [EXAMEN] 2. ¿Qué método configura el contexto HTTP en la clase base cuando se usa MockMvc standalone?

> [EXAMEN] 3. ¿Cuál es la diferencia entre `standaloneSetup` y `@SpringBootTest` como estrategia para la clase base del productor?

> [EXAMEN] 4. ¿Qué debe configurarse en el plugin para que distintos contratos de distintos dominios extiendan distintas clases base?

> [EXAMEN] 5. ¿Qué clase de configuración se usa en la clase base para aplicaciones Spring WebFlux en lugar de RestAssuredMockMvc?

---

← [10.5 Plugin Maven y Gradle](sc-contract-plugin-config.md) | [Índice](README.md) | [10.7 Stub Runner](sc-contract-stub-runner.md) →
