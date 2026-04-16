# 10.5 Clase base de tests del producer

← [10.4.3 Custom matchers y extensión del DSL](sc-contract-custom-matchers.md) | [Índice](README.md) | [10.6 Stub JAR y modos de StubRunner](sc-contract-stubs.md) →

---

## Introducción

El plugin de Spring Cloud Contract genera tests automáticamente, pero esos tests no saben cómo arrancar tu aplicación: no conocen tu contexto Spring, tus beans, ni la URL donde escucha tu servidor. La clase base de tests del producer es el contrato entre el plugin y tu código: tú dices cómo preparar el entorno, el plugin dice qué verificar. Sin una clase base correctamente configurada, los tests generados no compilan o fallan con `NullPointerException` antes de ejecutar ni una aserción. Este fichero cubre los cuatro modos de clase base (RestAssured, MockMvc, WebTestClient, Messaging), cuándo usar cada uno y cómo configurarlos correctamente para Spring Boot 4.0.x.

> [TAREA] El desarrollador necesita implementar la clase base de tests del producer para que los tests generados por el plugin compilen y ejecuten correctamente.

> [RESULTADO] Después de leer este fichero el desarrollador puede escribir una clase base que extiende el contrato correcto según el testMode configurado, arranca el contexto apropiado y permite que todos los tests generados compilen y pasen sin modificaciones adicionales.

> [PREREQUISITO] Configuración del plugin (10.3.1 y 10.3.2) y DSL de contratos (10.4.1).

## Representación visual

El siguiente diagrama muestra la relación entre el plugin, la clase base y los tests generados para cada modo.

```
plugin (pom.xml)
  └─ <testMode>MOCKMVC</testMode>
  └─ <baseClassForTests>com.example.BaseContractTest</baseClassForTests>
         │
         ▼
src/test/java/com/example/BaseContractTest.java
  extends ContractVerifierMockMvc    ← seleccionado por testMode
         │
         ▼
build/generated-test-sources/contracts/
  OrderVerifierTest.java             ← generado, extends BaseContractTest
    test_shouldReturnOrder()         ← generado por el plugin
```

Tabla de herencia según testMode:

```
testMode            Clase base a extender                  Cliente HTTP del test
──────────────────────────────────────────────────────────────────────────────
MOCKMVC             ContractVerifierMockMvc                MockMvc (sin servidor)
WEBTESTCLIENT       ContractVerifierWebTestClient          WebTestClient reactivo
RESTASSURED         ContractVerifierRestAssured            REST Assured (servidor real)
EXPLICIT            (ninguna; se usa RestTemplate directo) RestTemplate/HttpClient
MESSAGING           ContractVerifierMessagingBase          Spring Messaging / Stream
```

## Ejemplo central

Los siguientes ejemplos muestran la clase base completa para los tres modos más usados en producción, con la configuración de contexto correcta para Spring Boot 4.0.x.

**Modo MockMvc — para APIs servlet síncronas (más común):**

```java
package com.example.contract;

import io.restassured.module.mockmvc.RestAssuredMockMvc;
import org.junit.jupiter.api.BeforeEach;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.cloud.contract.verifier.messaging.boot.AutoConfigureMessageVerifier;
import org.springframework.web.context.WebApplicationContext;

// @SpringBootTest arranca el contexto completo. Para tests más rápidos,
// considera @WebMvcTest(OrderController.class) si el contrato solo toca un controller.
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.MOCK)
public abstract class BaseContractTest {

    @Autowired
    private WebApplicationContext context;

    @BeforeEach
    void setup() {
        RestAssuredMockMvc.webAppContextSetup(context);
    }
}
```

**pom.xml del producer (testMode MOCKMVC):**

```xml
<plugin>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-contract-maven-plugin</artifactId>
    <version>${spring-cloud-contract.version}</version>
    <extensions>true</extensions>
    <configuration>
        <testMode>MOCKMVC</testMode>
        <baseClassForTests>com.example.contract.BaseContractTest</baseClassForTests>
    </configuration>
</plugin>
```

**Modo WebTestClient — para APIs reactivas (WebFlux):**

```java
package com.example.contract;

import org.junit.jupiter.api.BeforeEach;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.cloud.contract.verifier.http.OkHttpHttpVerifier;
import org.springframework.test.web.reactive.server.WebTestClient;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public abstract class BaseContractTestReactive {

    @Autowired
    private WebTestClient webTestClient;

    @BeforeEach
    void setup() {
        // WebTestClient configurado con el contexto reactivo
        io.restassured.module.webtestclient.RestAssuredWebTestClient
            .webTestClient(webTestClient);
    }
}
```

**pom.xml del producer (testMode WEBTESTCLIENT):**

```xml
<configuration>
    <testMode>WEBTESTCLIENT</testMode>
    <baseClassForTests>com.example.contract.BaseContractTestReactive</baseClassForTests>
</configuration>
```

**Modo Messaging — para producers de eventos:**

```java
package com.example.contract;

import org.junit.jupiter.api.BeforeEach;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.cloud.contract.verifier.messaging.boot.AutoConfigureMessageVerifier;
import org.springframework.messaging.MessageChannel;
import org.springframework.cloud.stream.binder.test.OutputDestination;
import org.springframework.cloud.stream.binder.test.TestChannelBinderConfiguration;
import org.springframework.context.annotation.Import;

@SpringBootTest
@AutoConfigureMessageVerifier
@Import(TestChannelBinderConfiguration.class)
public abstract class BaseMessagingContractTest {

    @Autowired
    protected OutputDestination outputDestination;

    // Spring Cloud Contract inyecta el ContractVerifierMessaging
    // automáticamente gracias a @AutoConfigureMessageVerifier
}
```

**baseClassMappings — múltiples clases base para contratos distintos:**

```xml
<configuration>
    <baseClassMappings>
        <baseClassMapping>
            <contractPackageRegex>.*orders.*</contractPackageRegex>
            <baseClassFQN>com.example.contract.OrderBaseContractTest</baseClassFQN>
        </baseClassMapping>
        <baseClassMapping>
            <contractPackageRegex>.*messaging.*</contractPackageRegex>
            <baseClassFQN>com.example.contract.BaseMessagingContractTest</baseClassFQN>
        </baseClassMapping>
    </baseClassMappings>
</configuration>
```

**Verificar tests generados (ubicación estándar):**

```bash
# Maven genera los tests en:
target/generated-test-sources/contracts/

# Ejemplo de clase generada (naming: NombreCarpeta + "Test"):
# target/generated-test-sources/contracts/com/example/OrderTest.java

# Ejecutar solo los tests de contrato:
mvn test -Dtest=*Test -pl producer-service
```

## Tabla de elementos clave

Los parámetros y clases que un profesional senior debe conocer de memoria para configurar la clase base correctamente.

| Elemento | Tipo | Default | Descripción |
|---|---|---|---|
| `<testMode>` | enum | `MOCKMVC` | Modo de test: MOCKMVC, WEBTESTCLIENT, RESTASSURED, EXPLICIT, MESSAGING |
| `<baseClassForTests>` | String | — | FQN de la clase base para todos los contratos |
| `<baseClassMappings>` | Lista | — | Mapa regex-paquete → clase base; sobreescribe `baseClassForTests` |
| `ContractVerifierMockMvc` | Clase abstracta | — | Clase base para testMode=MOCKMVC; configura RestAssuredMockMvc |
| `ContractVerifierWebTestClient` | Clase abstracta | — | Clase base para testMode=WEBTESTCLIENT |
| `ContractVerifierMessagingBase` | Clase abstracta | — | Clase base para contratos de mensajería |
| `@AutoConfigureMessageVerifier` | Anotación | — | Activa la infraestructura de verificación de mensajes en tests |
| `@SpringBootTest(MOCK)` | Enum | MOCK | Arranca MockMvc sin servidor; más rápido que RANDOM_PORT |
| `@SpringBootTest(RANDOM_PORT)` | Enum | — | Arranca servidor real; necesario para REST Assured o WebTestClient |
| `build/generated-test-sources` | Directorio | — | Ubicación de los tests generados por el plugin Maven |
| `basePackageForTests` | String | packageFromContractsDir | Paquete Java de los tests generados |
| `nameSuffixForTests` | String | `Test` | Sufijo de las clases generadas (ej: `OrderTest`) |

## Buenas y malas prácticas

**Hacer:**

- Usar `@SpringBootTest(webEnvironment = MOCK)` con MockMvc como primera opción: arranca el contexto completo pero sin servidor real, lo que reduce el tiempo de arranque un 40-60 % respecto a `RANDOM_PORT`.
- Definir `baseClassMappings` cuando el producer tiene tanto contratos HTTP como contratos de mensajería: las dos familias de contratos necesitan configuraciones de contexto distintas y mezclarlas en una única base class produce tests que no compilan.
- Declarar el método `setup()` con `@BeforeEach` (JUnit 5), nunca con `@Before` (JUnit 4): los tests generados por el plugin usan JUnit 5 desde Contract 4.x, y la mezcla de anotaciones causa `MissingMethodInvocationException` silenciosos.
- Añadir datos de prueba o stubs de repositorio en `@BeforeEach` de la clase base; los tests generados no pueden inyectar datos por sí mismos y fallan con `EmptyResultDataAccessException` si los repositorios no tienen datos.

**Evitar:**

- No extender la clase base directamente de `@SpringBootTest` omitiendo la herencia de `ContractVerifierMockMvc`: el test generado llama a `RestAssuredMockMvc.given()` sin contexto configurado, produciendo `IllegalStateException: no base URI or mock mvc instance specified`.
- No usar `@WebMvcTest` en la clase base si el contrato toca beans fuera del slice (servicios, repositorios): los tests generados fallarán con `NoSuchBeanDefinitionException` porque el slice no arranca esos beans. Usar `@SpringBootTest(MOCK)` o añadir `@Import` de los beans necesarios.
- No registrar `@MockBean` en la clase base para silenciar repositorios de producción y olvidar configurar el comportamiento del mock: el test generado asume una respuesta real y falla con `NullPointerException` al mapear la respuesta nula.
- No mezclar Kotlin y Java en la jerarquía de la clase base: el plugin genera tests en el idioma del contrato DSL, y si la base class está en Kotlin pero el test generado es Java, las anotaciones JUnit no se detectan correctamente en algunos casos edge.

## Comparación: @SpringBootTest vs @WebMvcTest vs Standalone

Elegir el nivel de contexto correcto en la clase base tiene impacto directo en la velocidad del pipeline CI y en la fiabilidad de los tests.

| Criterio | `@SpringBootTest(MOCK)` | `@WebMvcTest` | Standalone (sin Spring) |
|---|---|---|---|
| Tiempo arranque | Medio (~5 s) | Rápido (~1 s) | Muy rápido (<0,5 s) |
| Beans disponibles | Todos | Solo web layer | Ninguno |
| Requiere DB/Redis | Sí (o `@MockBean`) | No | No |
| Riesgo de falsos positivos | Bajo | Medio (beans mockeados) | Alto (sin integración) |
| Cuándo usar | APIs con lógica en servicios | Controller aislado | Lógica pura sin Spring |

---

← [10.4.3 Custom matchers y extensión del DSL](sc-contract-custom-matchers.md) | [Índice](README.md) | [10.6 Stub JAR y modos de StubRunner](sc-contract-stubs.md) →

