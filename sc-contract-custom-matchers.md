# 10.4.3 Custom matchers y extensión del DSL

← [10.4.2 Matchers estándar en contratos](sc-contract-matchers.md) | [Índice](README.md) | [10.5 Clase base de tests del producer](sc-contract-producer-tests.md) →

---

## Introducción

Los matchers estándar de Spring Cloud Contract cubren el 80 % de los casos, pero en proyectos reales aparecen reglas de negocio que ningún matcher built-in puede expresar: un NIF español, un código IBAN, un identificador de producto interno con formato propietario. Intentar forzar estas reglas con `byRegex()` produce contratos largos e ilegibles; peor aún, el propio equipo consumer deja de entender qué valida el contrato. La solución es `byCommand()`, los BodyMatchers avanzados y las extensiones del DSL: mecanismos que permiten delegar la validación a código Java propio sin salir del ciclo CDC. Este fichero cubre todos los puntos de extensión del motor de verificación de Spring Cloud Contract.

> [TAREA] El desarrollador necesita implementar custom matchers y extensiones del DSL para validar reglas de negocio propias no cubiertas por los matchers estándar.

> [RESULTADO] Después de leer este fichero el desarrollador puede escribir contratos que validan con `byCommand()`, componer matchers con `anyOf()`/`allOf()`, aplicar `matchingJsonPath` con predicados propios y registrar un `StubRunnerExtension` para comportamiento dinámico en stubs.

> [PREREQUISITO] Dominio de matchers estándar (10.4.2) y DSL base (10.4.1).

## Representación visual

El siguiente diagrama muestra los cinco puntos de extensión y en qué fase del ciclo CDC actúa cada uno.

```
PUNTO DE EXTENSIÓN          FASE DE ACTUACIÓN          FICHERO IMPLICADO
─────────────────────────────────────────────────────────────────────────
byCommand(method)           Producer test (verifica)   Contrato + clase base
BodyMatchers avanzados      Producer test (verifica)   Contrato
anyOf() / allOf()           Producer test (verifica)   Contrato
StubRunnerExtension         Consumer test (stub)       Clase de extensión
Custom WireMock transformer Consumer stub (runtime)    Transformer registrado
```

Diagrama de flujo:

```
[Contrato .groovy]
  └─ bodyMatchers {
       jsonPath('$.nif', byCommand('com.example.NifValidator.validate'))
       jsonPath('$.amount', anyOf(byType(), byRegex(/[0-9]+(\.[0-9]{2})?/)))
     }
         │
         ▼
[Plugin genera test producer]          [Plugin genera stub WireMock]
  └─ llama NifValidator.validate()       └─ valor concreto del productor en el mapping
  └─ si false → test falla
```

## Ejemplo central

El siguiente ejemplo completo implementa un custom matcher para un NIF, usa `anyOf()` combinando tipo y formato, y registra un `StubRunnerExtension` que añade un header dinámico en el stub del consumer.

**Contrato (src/test/resources/contracts/orders/shouldReturnOrder.groovy):**

```groovy
import org.springframework.cloud.contract.spec.Contract

Contract.make {
    description "Pedido con NIF validado por custom matcher"
    request {
        method GET()
        url '/orders/42'
        headers { contentType(applicationJson()) }
    }
    response {
        status OK()
        headers { contentType(applicationJson()) }
        body(
            id:        value(producer(42), consumer(42)),
            nif:       value(producer('12345678Z'), consumer('12345678Z')),
            amount:    value(producer(99.99), consumer(99.99)),
            createdAt: $(producer(regex(iso8601WithOffset())), consumer('2025-01-15T10:00:00Z'))
        )
        bodyMatchers {
            // byCommand: delega la validación del NIF a un método estático en la base class
            jsonPath('$.nif', byCommand('assertValidNif($it)'))
            // anyOf: el amount debe ser numérico O seguir el patrón decimal
            jsonPath('$.amount', anyOf(byType(), byRegex(/[0-9]+(\.[0-9]{2})?/)))
            // matchingJsonPath: verifica que id sea positivo usando JsonPath predicate
            jsonPath('$.id', byType { minOccurrence(1) })
            // byEquality: verifica igualdad exacta para campo invariante
            jsonPath('$.status', byEquality())
        }
    }
}
```

**Clase base del producer (src/test/java/com/example/BaseContractTest.java):**

```java
package com.example;

import io.restassured.module.mockmvc.RestAssuredMockMvc;
import org.junit.jupiter.api.BeforeEach;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.web.context.WebApplicationContext;

@SpringBootTest
public abstract class BaseContractTest {

    @Autowired
    WebApplicationContext context;

    @BeforeEach
    void setup() {
        RestAssuredMockMvc.webAppContextSetup(context);
    }

    // método referenciado en byCommand('assertValidNif($it)')
    // Spring Cloud Contract inyecta el valor real como $it
    public static void assertValidNif(String nif) {
        if (nif == null || !nif.matches("[0-9]{8}[A-HJ-NP-TV-Z]")) {
            throw new AssertionError("NIF inválido: " + nif);
        }
    }
}
```

**StubRunnerExtension para el consumer (src/test/java/com/example/DynamicHeaderExtension.java):**

```java
package com.example;

import org.springframework.cloud.contract.stubrunner.StubRunnerExtension;
import org.springframework.cloud.contract.stubrunner.StubRunnerOptions;
import com.github.tomakehurst.wiremock.WireMockServer;

public class DynamicHeaderExtension implements StubRunnerExtension {

    @Override
    public void afterStart(StubRunnerOptions options,
                           WireMockServer wireMockServer) {
        // Añade un stub adicional con header dinámico calculado en runtime
        wireMockServer.stubFor(
            com.github.tomakehurst.wiremock.client.WireMock
                .get(com.github.tomakehurst.wiremock.client.WireMock.urlEqualTo("/health"))
                .willReturn(com.github.tomakehurst.wiremock.client.WireMock.aResponse()
                    .withStatus(200)
                    .withHeader("X-Trace-Id", java.util.UUID.randomUUID().toString())
                    .withBody("{\"status\":\"UP\"}")));
    }
}
```

**Registro de la extensión (src/test/resources/META-INF/services/org.springframework.cloud.contract.stubrunner.StubRunnerExtension):**

```
com.example.DynamicHeaderExtension
```

**Test del consumer usando la extensión:**

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureStubRunner(
    ids = "com.example:order-service:+:stubs:8090",
    stubsMode = StubRunnerProperties.StubsMode.CLASSPATH
)
class OrderConsumerTest {

    @Autowired
    private OrderClient orderClient;

    @Test
    void shouldFetchOrderWithValidatedNif() {
        OrderDTO order = orderClient.getOrder(42L);
        assertThat(order.getId()).isEqualTo(42L);
        assertThat(order.getNif()).matches("[0-9]{8}[A-HJ-NP-TV-Z]");
    }
}
```

**pom.xml (dependencias mínimas):**

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-contract-verifier</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-contract-stub-runner</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>io.rest-assured</groupId>
        <artifactId>spring-mock-mvc</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

## Tabla de elementos clave

Los siguientes matchers y puntos de extensión son los que un profesional senior debe dominar para evaluaciones técnicas.

| Elemento | Síntaxis DSL | Lado | Descripción |
|---|---|---|---|
| `byCommand(expr)` | `jsonPath('$.f', byCommand('Cls.method($it)'))` | Producer | Delega validación a método estático; `$it` = valor real |
| `byEquality()` | `jsonPath('$.f', byEquality())` | Producer | Igualdad exacta al valor del contrato |
| `byNull()` | `jsonPath('$.f', byNull())` | Producer | Verifica que el campo sea JSON null |
| `matchingJsonPath(path)` | `jsonPath('$.arr', byType { minOccurrence(1) })` | Producer | Valida estructura con predicado de tipo |
| `anyOf(m1, m2)` | `anyOf(byType(), byRegex(/pattern/))` | Producer | Pasa si cualquiera de los matchers pasa |
| `allOf(m1, m2)` | `allOf(byRegex(...), byType())` | Producer | Pasa solo si todos los matchers pasan |
| `StubRunnerExtension` | SPI — clase implementa interfaz | Consumer | Hook ejecutado tras arrancar WireMock; permite añadir stubs extra |
| Custom transformer | Clase extiende `ResponseDefinitionTransformer` | Consumer | Transforma la respuesta WireMock en runtime |
| `byType { minOccurrence(n) }` | bloque Groovy en bodyMatchers | Producer | Valida arrays: al menos N elementos del tipo correcto |

## Buenas y malas prácticas

**Hacer:**

- Colocar el método referenciado en `byCommand()` en la clase base de tests del producer y declararlo `public static`; si no es público, el test generado lanza `IllegalAccessException` en runtime.
- Usar `anyOf(byType(), byRegex(...))` cuando un campo puede tener varias representaciones válidas (int o decimal formateado); fuerza al producer a cumplir al menos una regla sin sobreconstrenir.
- Registrar `StubRunnerExtension` vía SPI (archivo en `META-INF/services`) en lugar de instanciarlo manualmente; así la extensión se activa para todos los tests sin modificar la anotación `@AutoConfigureStubRunner`.
- Documentar cada `byCommand()` con un comentario en el contrato que explique la regla de negocio validada; los contratos son documentación viva compartida entre equipos.

**Evitar:**

- No usar `byCommand()` para lógica que realiza llamadas a base de datos o servicios externos: el método se ejecuta durante el test del producer, lo que introduce dependencias de infraestructura en los tests de contrato y causa fallos intermitentes en CI.
- No combinar `allOf()` con tres o más matchers restrictivos sobre el mismo campo: si la API evoluciona y uno deja de cumplirse, el error de compilación del test generado no indica qué matcher falló, dificultando el diagnóstico.
- No registrar transformers WireMock que modifiquen el body de la respuesta para añadir campos no presentes en el contrato: el consumer podría comenzar a depender de esos campos extra, creando un contrato implícito no verificado.
- No usar `byEquality()` en campos como timestamps o UUIDs generados: fuerza al producer a devolver exactamente el valor del contrato, rompiendo el test en cuanto la implementación usa valores dinámicos.

## Comparación: byCommand vs byRegex vs byType

Las tres estrategias de validación personalizada tienen casos de uso distintos. Elegir la incorrecta produce tests frágiles o imposibles de mantener.

| Criterio | `byRegex()` | `byType()` | `byCommand()` |
|---|---|---|---|
| Regla de negocio compleja | No (solo formato) | No (solo tipo Java) | Sí (lógica arbitraria) |
| Legibilidad en contrato | Alta (patrón visible) | Alta (tipo explícito) | Media (referencia a método) |
| Mantenimiento | Bajo | Muy bajo | Medio (método en base class) |
| Riesgo de acoplamiento | Bajo | Bajo | Medio (la base class debe tener el método) |
| Caso típico | UUID, email, fecha | String, Integer, Boolean | NIF, IBAN, código interno |

---

← [10.4.2 Matchers estándar en contratos](sc-contract-matchers.md) | [Índice](README.md) | [10.5 Clase base de tests del producer](sc-contract-producer-tests.md) →

